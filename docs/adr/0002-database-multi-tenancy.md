# ADR-0002: Multi-tenancy — tenant columns on plain PostgreSQL, RLS as fail-closed enforcement

- **Status:** Evaluating — direction agreed; Phoenix integration spike before acceptance
- **Date:** 2026-10-21
- **Deciders:** Rico Metzger (founder)
- **Related:** ADR-0001 (stack), ADR-0008 (hosting: Fly.io EU)

## Context

The product is a multi-tenant SaaS holding freelancer finance data. The founder's
framing: multi-tenancy is a *model* distinction — "normally that is done with a tenant
column" — and the database must stay a **normal database** with no vendor-specific
features, so hosting stays swappable. Safety must be structural: a forgotten
`WHERE tenant_id = …` must not be able to leak another tenant's data.

## Agreed direction

**Shared-schema multi-tenancy on plain PostgreSQL: every tenant-owned table carries a
`tenant_id` column (the model), Row-Level Security is the enforcement layer on top (the
guarantee).** RLS is core Postgres since 9.5 — no vendor feature, works identically
self-hosted, on Fly, on Neon, anywhere. Supabase/Neon are *hosting* choices, not
database choices (Neon: pure Postgres + branching; Supabase: Postgres plus PostgREST
conventions we deliberately don't adopt).

Policy shape (fail-closed):

```sql
ALTER TABLE invoices FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON invoices
  USING (tenant_id = current_setting('app.tenant_id', true)::uuid);
```

App role: no `BYPASSRLS`, no superuser. If the setting is missing, the policy matches
nothing — leaks become zero-rows instead of another tenant's data.

## The open question: how exactly, in Phoenix

The founder used the transactional `with_tenant` pattern before and found it **a pain
with Phoenix** — but agrees "it would be safest". Before acceptance, the integration
must be designed against the real pain points, and proven in a spike.

### Known pain points of the transactional pattern

1. **Everything becomes a transaction.** Even reads need `BEGIN … SET LOCAL … SELECT …
   COMMIT`. Perf cost is minor (an extra round trip — this product is
   latency-tolerant); the real pain is ergonomics and forgetting to wrap.
2. **Process-context leaks.** Phoenix spawns processes that do *not* inherit tenant
   context: LiveView `Task.async`/`assign_async`, Presence, Oban jobs. Every spawned
   process must re-establish tenant scope explicitly.
3. **Fail-closed debugging.** An unwrapped `Repo.all` silently returns zero rows —
   secure, but confusing unless dev/test guardrails raise loudly.
4. **Test friction.** Ecto SQL sandbox already wraps tests in a transaction; `set_config`
   must cooperate with it, and factories must seed within tenant scope.
5. **Pooling discipline.** `set_config(..., true)` is transaction-local and pool-safe;
   any accidental session-level `SET` would leak across checkouts — must never appear.

### Pattern candidates for the spike

| Pattern | Mechanism | Friction | Safety |
|---|---|---|---|
| **A. Explicit `with_tenant/2` everywhere** | Caller wraps every unit of work in `Repo.transaction` + `set_config` | High (the founder's past pain) | Highest: intent is visible at every call site |
| **B. TenantContext (process dictionary) + Repo wrapper** | Plug stores tenant in process dict; a thin `MyApp.Repo` wrapper auto-opens the transaction and sets config (reusing an open transaction when inside one); dev/test override raises on repo calls outside a scope; Oban workers get a `use TenantedWorker` macro; a `TenantScope.spawn/3` helper for tasks | Low at call sites; discipline lives in one place | Same RLS guarantee; guardrails catch misses in dev |
| **C. Schema-per-tenant via Ecto prefixes** | Ecto's native `prefix` support | Migration/pool sprawl grows with tenants | Rejected (recorded ADR-0002 v1); revisit only as escape hatch |

Candidate B is the likely winner *provided* the spike proves the guardrails; the RLS
policies make even a missed guard a zero-row result, not a leak — the guardrails are for
debuggability, RLS is the guarantee.

### Spike definition (exit criteria for acceptance)

A minimal Phoenix + Ecto + Oban + LiveView app proving, for the chosen pattern:

1. Request-scoped read/write through the normal repo calls (no per-call wrapping in
   controllers/LiveViews).
2. Oban job scoping (tenanted worker macro) incl. a redelivery/idempotency scenario.
3. LiveView async (`Task.async`/`assign_async`) scoping via the spawn helper —
   explicitly passing tenant, no accidental inheritance gaps.
4. Cross-tenant leak test: a forged query with wrong/absent tenant returns zero rows;
   same test fails loudly if RLS is dropped.
5. Sandbox-compatible tests + factories seeded in tenant scope.
6. Pooled connections verified safe (`set_config` transaction-local; no session leakage).
7. Measured overhead of transaction-per-request on a representative endpoint.

Acceptance requires the founder's review of the spike's ergonomics.

## Consequences (anticipated, pending spike)

- Structural, testable tenancy safety independent of application discipline.
- Cost: transaction-per-request overhead + one integration module to maintain.
- Hosting stays swappable (see ADR-0008: Fly app hosting; Postgres hosting is a separate
  open decision — Fly Postgres vs Neon vs self-hosted — and will follow the spike).