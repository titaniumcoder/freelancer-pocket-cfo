# ADR-0002: Multi-tenancy — tenant columns on plain PostgreSQL, RLS as fail-closed enforcement

- **Status:** Accepted
- **Date:** 2026-10-21
- **Deciders:** Rico Metzger (founder)
- **Related:** ADR-0001 (stack: Elixir + Phoenix + Ecto + PostgreSQL), hosting ADR (follow-up)

## Context

The product is a multi-tenant SaaS holding freelancer finance data: invoices, payment
state, budget windows, actuals, tax-relevant documents. README.md requires that
cross-tenant access be *structurally impossible, not merely forbidden*.

The founder's framing (2026-10-21): multi-tenancy is first of all a *model* distinction —
"normally that is done with a tenant column" — and the database must remain a **normal
database** with no vendor-specific features, so the project can switch hosting around
freely. Safety must be structural: forgetting a `WHERE tenant_id = …` must not be able
to leak another tenant's data.

## Decision

**Shared-schema multi-tenancy on plain PostgreSQL. Every tenant-owned table carries a
`tenant_id` column (the model), and Row-Level Security is the enforcement layer on top
of it (the guarantee).** RLS is not an alternative to the tenant column — it is what
makes the tenant column un-forgettable.

Key properties:

- **Plain PostgreSQL.** RLS has been core Postgres since 9.5. It is not a cloud or
  vendor feature. It works identically on self-hosted Postgres, Fly Postgres, Neon,
  or Supabase's Postgres. Nothing in this decision locks hosting.
- **Fail-closed by construction.** The app connects as a dedicated non-superuser role
  without `BYPASSRLS`; tables are `ALTER TABLE … FORCE ROW LEVEL SECURITY` (so even the
  table owner cannot bypass); every policy reads the tenant from the transaction-local
  setting. If the setting is missing, the policy matches nothing — a forgotten tenant
  filter returns zero rows instead of another tenant's data.
- **Postgres flavors are hosting choices, not database choices.** Neon = pure Postgres
  (branching is attractive for ephemeral dev/CI databases; EU region available).
  Supabase = Postgres plus PostgREST/JWT conventions (`auth.uid()` patterns) that an
  Ecto app would not use — acceptable as *plain Postgres hosting*, but we deliberately
  adopt none of its app-level conventions. The hosting decision is a separate ADR.

## How it is done with normal Elixir tooling (the documented pattern)

1. **Roles.** Migrations run as an owner role. The runtime app role is separate, has no
   `BYPASSRLS`/superuser, and is the only role used in production (`FORCE ROW LEVEL
   SECURITY` keeps the owner path closed too).
2. **Per-request tenant binding.** A Phoenix plug resolves the tenant (from session/
   token) and opens a transaction that sets the tenant transaction-locally:

   ```elixir
   def with_tenant(tenant_id, fun) do
     Repo.transaction(fn ->
       # `true` = transaction-local, auto-cleared on commit/rollback
       Repo.query!("SELECT set_config('app.tenant_id', $1, true)", [tenant_id])
       fun.()
     end)
   end
   ```

   All queries inside inherit the scope. A convenience macro (`tenanted_schema` +
   `scope/2`) may generate tenant predicates for ergonomics — but the *invariant* is
   RLS; the macro is sugar, never the guarantee.
3. **Policy shape.** On every tenanted table:

   ```sql
   ALTER TABLE invoices FORCE ROW LEVEL SECURITY;
   CREATE POLICY tenant_isolation ON invoices
     USING (tenant_id = current_setting('app.tenant_id', true)::uuid);
   ```

4. **Guard rails.** `tenant_id UUID NOT NULL` with composite FKs and composite indexes
   `(tenant_id, …)` everywhere; migrations checklist item: every new tenanted table
   ships with its policy in the same migration. A dedicated test (and a Livebook
   recipe) proves a query with no tenant context returns zero rows and a foreign-tenant
   lookup is empty. No cross-tenant foreign keys, ever.

## Options considered

| Option | Isolation | Solo-ops cost | Verdict |
|---|---|---|---|
| Shared schema + `tenant_id` + RLS (this decision) | Highest: fail-closed at the DB | Low | **Accepted** |
| Shared schema + `tenant_id` column only (no RLS) | Lowest: correctness depends on every query remembering the filter | Lowest | Rejected — the one bad query is enough |
| Schema-per-tenant | High, but N schemas × M migrations sprawl | Medium | Rejected |
| Database-per-tenant | Strongest blast radius | High | Rejected for now |
| Supabase/Neon "as the database" | N/A — they *are* Postgres; not a tenancy model | — | N/A — hosting only; see hosting ADR |

## Consequences

- Tenancy leaks become impossible-by-construction, provable in tests, independent of
  application code discipline.
- Slight per-request transaction overhead (`set_config` inside a transaction); acceptable
  for a latency-tolerant finance SaaS.
- Hosting stays swappable; Neon's branching and EU regions make it the leading
  *hosting* candidate to be weighed in the hosting ADR (self-hosted/Hetzner remains the
  sovereignty alternative).
- Any future managed-Postgres migration keeps all RLS policies intact because they are
  plain SQL.
