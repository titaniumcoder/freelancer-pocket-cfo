# ADR-0002: Multi-tenancy — tenant columns with Phoenix 1.8 app-layer scoping; RLS not adopted

- **Status:** Accepted
- **Date:** 2026-10-21
- **Deciders:** Rico Metzger (founder)
- **Related:** ADR-0001 (stack), ADR-0008 (hosting — Neon is the default Postgres host),
  ADR-0005 (chatbot context isolation builds on this)

## Context

Multi-tenancy is a model distinction: "normally that is done with a tenant column" —
the database stays a normal database with no vendor-specific features, so hosting stays
swappable. Cross-tenant access must be impossible by construction, not merely forbidden.

The founder **re-opened the earlier RLS direction** with a concrete technology
observation: **Phoenix 1.8 changed the calculus.** Generated contexts ship a `scope/2`
helper and every generated query automatically extends with the tenant predicate; the
LiveView generators thread a `Scope` struct through assigns; `phx.gen.auth` integrates
with it. Tenant scoping is no longer a hand-rolled discipline — it is generated,
ergonomic, and reviewable. The transactional `with_tenant` + `SET LOCAL` machinery
(whose Phoenix friction was already recorded) solves a problem the framework now solves
better: **"sounds like a need from the past."**

## Decision

**Tenanted tables carry `tenant_id` (the model). Scoping is enforced in the application
layer through Phoenix 1.8 scopes: every tenanted context function requires a `Scope` and
extends queries with the tenant predicate automatically. Row-Level Security is *not*
adopted.**

Concretely:

- **The context API is the only data access surface.** Controllers and LiveViews never
  call `Repo` directly for tenanted data; they call context functions that require a
  Scope. A query that reaches `Repo` without a tenant predicate is a code-review and CI
  failure, not a runtime hope.
- **Oban jobs carry the tenant explicitly** (job args / worker metadata) and re-build
  the Scope at `perform/2` — no process-dictionary magic.
- **Database hygiene stays:** app role without superuser, `tenant_id UUID NOT NULL`,
  composite indexes `(tenant_id, …)`, no cross-tenant foreign keys, migrations ship the
  column + index together. Tenant deletion = row deletion (GDPR erasure without schema
  surgery).
- **RLS explicitly rejected** — not deferred. Rationale: the database is accessed by
  exactly one application; the framework generates the scoping; the transactional
  machinery bought protection against bugs in a layer the framework now protects
  anyway, at real friction (transaction-per-request, process-context plumbing in
  LiveView/Oban, sandbox awkwardness).

### The honest difference (recorded, with mitigations)

RLS would additionally have protected against: (a) a forgotten scope in hand-written
(non-generated) queries, and (b) non-application access paths hitting the database
directly. Mitigations adopted in its place:

1. Context-API-only rule, enforced by convention and CI (a repo-calls-outside-contexts
   check).
2. A cross-tenant leak test in CI: asserting that scope functions never return foreign
   rows, seeded adversarially.
3. **Revisit trigger:** if a second access path to the database ever appears (BI tool,
   reporting service, microservice), the RLS evaluation reopens — that is the scenario
   it was built for.

## Options considered

| Option | Verdict | Why |
|---|---|---|
| **App-layer scoping via Phoenix 1.8 scopes** | **Accepted** | Generated, ergonomic, reviewable; zero runtime machinery; matches how the framework now wants apps built |
| RLS (`with_tenant` + `SET LOCAL`) | Rejected | Real Phoenix friction, transaction-per-request, protects a layer the framework now scopes by construction; the founder's call: a need from the past |
| Schema-per-tenant (Ecto prefixes) | Rejected | Migration/pool sprawl grows linearly with tenants |
| Database-per-tenant | Rejected | Ops cost × tenants; escape hatch only if a large B2B tenant demands it |

## Consequences

- **Easier:** no transaction wrapping anywhere; no tenant-setting plumbing in
  LiveView/Oban/Task processes; the DB stays boring, portable SQL — pairing perfectly
  with managed Neon (ADR-0008).
- **Risk accepted:** fail-open instead of fail-closed — a bug that bypasses the scope
  layer *can* leak, where RLS would have zeroed it. Bounded by the context-API-only
  rule, the CI leak test, and the revisit trigger.
- **README.md alignment:** "structurally impossible" is anchored to the architecture:
  the scoped context layer is the only data access surface — cross-tenant access is
  impossible *by construction of the app*, enforced in CI.
- **Downstream effects:** ADR-0005's chatbot context isolation assembles prompts
  exclusively through this scoped API (one mechanism for the whole product).