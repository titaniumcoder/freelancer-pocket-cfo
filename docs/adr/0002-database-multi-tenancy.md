# ADR-0002: Multi-tenancy model — PostgreSQL Row-Level Security

- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** Rico Metzger (founder) — decision pending
- **Related:** ADR-0001 (stack: Elixir + Phoenix + Ecto + PostgreSQL)

## Context

The product is a multi-tenant B2B SaaS holding freelancer finance data: invoices, payment
state, budget windows, actuals, tax-relevant documents. A forgotten `WHERE tenant_id = …`
must not be able to leak another tenant's data — README.md requires that cross-tenant
access be *structurally impossible, not merely forbidden*. The stack decision (ADR-0001)
already fixed PostgreSQL with Ecto; this ADR decides *how tenancy is modelled and
enforced*.

EU context: tenant data is personal and finance data under GDPR; the less redundancy and
 sprawl the tenancy model creates, the smaller the GDPR surface (per-tenant backups,
deletion, export).

## Options considered

| Model | Isolation strength | Ops cost at N tenants | Verdict |
|---|---|---|---|
| **Shared schema, `tenant_id` + RLS on every table** | Highest: isolation lives in the database. Even a query that forgot the tenant predicate cannot cross tenants. App connects as a non-superuser role with `FORCE ROW LEVEL SECURITY`; every transaction sets `SET LOCAL app.tenant_id` | Constant: 1 schema, N rows | **Recommended** |
| Shared schema, application-layer scoping only (Ecto query prefixes, no RLS) | Weakest: one missing `where: [tenant_id: …]` in any code path (including future ad-hoc SQL, reporting queries, admin tools) leaks across tenants | Constant | Rejected — fails the "structurally impossible" bar |
| Schema-per-tenant | High: hard boundary per tenant; but Ecto migrations and connection pooling (`pgbouncer`, prepared statements) get messy with dynamic schema paths | Linear: N schemas × M migrations | Rejected — migration and pool sprawl |
| Database-per-tenant | Strongest blast radius; cleanest deletion story | Linear: N databases × M migrations × backup/monitoring × connection budget | Rejected for now; escape hatch if a large B2B tenant demands it |

### What the RLS model requires (feasibility, verified against Ecto/Postgres practice)

- Postgres RLS is mature (since 9.5) and fully supported through Ecto: the app sets
  `tenant_id` per transaction via a custom Ecto connection hook; RLS policies reference
  `current_setting('app.tenant_id')`.
- The application DB role must have **no** `BYPASSRLS` and **no** superuser credentials
  in production; `FORCE ROW LEVEL SECURITY` also binds the table owner.
- Migrations and seeding run as a separate role that *can* bypass RLS (or with policies
  disabled in the migration path) — a normal, well-understood pattern.
- Pooling: use a transaction-pooled setup (Postgrex/pool size or pgbouncer in
  transaction mode) with `SET LOCAL` (per-transaction, auto-reset) — never `SET`
  (session state leaks across pooled checkouts).
- Soft-delete/audit layers (append-only change log) sit naturally on top: the audit
  table gets the same RLS policy.

## Decision (proposed)

Shared schema with `tenant_id` on every tenant-owned table, RLS policies on every
tenant-owned table, app role without BYPASSRLS, `SET LOCAL app.tenant_id` per
transaction, migrations via a privileged role.

## Consequences (if accepted)

- **Easier:** structural tenancy safety; single migration pipeline; single backup story;
  tenant deletion = row deletion (satisfies GDPR erasure without schema surgery).
- **Harder:** every table must carry `tenant_id` and an RLS policy from day one
  (discipline: a schema-check in CI); connection hooks add a small per-transaction cost;
  debugging requires remembering the RLS role in use.
- **Deferred:** hosting/Postgres provider choice (Fly/Hetzner/managed EU provider) is a
  separate ADR; vector/extension needs (pgvector) unknown until the chatbot ADR-0005
  lands.

## Open questions for the founder

1. Accept the RLS model as proposed? (Alternative: schema-per-tenant for perceived
   simplicity of per-tenant export — rejected above on ops cost.)
2. Should a *large enterprise tenant* isolation escape hatch (database-per-tenant) be
   kept explicitly in the roadmap or dropped from the record?
