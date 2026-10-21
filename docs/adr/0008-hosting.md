# ADR-0008: Hosting — Fly.io on an EU region, behind our own hostname and egress IP

- **Status:** Accepted
- **Date:** 2026-10-21
- **Deciders:** Rico Metzger (founder)
- **Related:** ADR-0001 (stack), ADR-0002 (tenancy — Postgres hosting is a separate open
  decision), ADR-0004 (WhatsApp — stable egress IP matters), ADR-0005 (LLM providers)

## Context

Solo-founder ops and EU data residency shape hosting. The founder's call: **Fly.io**,
because the machines are mostly stateless and the deployment story for Elixir
applications is too easy to ignore — a release artifact deploys as-is, and horizontal
scale is a machine count, not an architecture change.

The founder also pinned two operational specifics:

- The app sits **behind a dedicated egress IP** (outbound traffic to Meta, LLM
  providers, Paddle presents a stable source address).
- The app serves **our own hostname: `pocket-cfo.titaniumcoder.com`**.

## Decision

**Host the Phoenix application on Fly.io in an EU region (e.g. `fra` Frankfurt — exact
region picked at setup from Fly's EU list), with:**

- **Stateless machines** — all durable state lives in Postgres and object storage;
  machines boot from the release, nothing local is trusted.
- **Custom hostname** `pocket-cfo.titaniumcoder.com` with Fly-managed certificates
  (Let's Encrypt); the apex `titaniumcoder.com` DNS delegates the subdomain.
- **Dedicated egress IP** for outbound traffic — stable source IP for provider
  allowlists (Meta Cloud API, LLM APIs, Paddle webhooks callbacks) and a cleaner GDPR
  processor story (fixed, identifiable source).
- **Deploys via release artifact** (`fly deploy` from CI or locally) — no Kubernetes,
  no build-pipeline zoo; ADR-0001's "one BEAM binary, one VM" rationale continues here.

### Postgres hosting: Neon is the default candidate, Fly Postgres rejected

The founder decided the direction (2026-10-21): **Fly Postgres is out** — "ways too
expensive for our usecase." Even as a SaaS, the app will do nothing ~90% of the time;
keeping a clustered/always-on database running is not useful for that profile.
Managed serverless Postgres brings more features at the right cost shape:

| Option | Verdict | Why |
|---|---|---|
| **Neon (EU region)** | **Default candidate** | Founder has worked with it many times; **database branching** for ephemeral dev/test environments is "really nice"; scale-to-zero fits the idle-heavy profile; pure Postgres (pairs with ADR-0002's plain-SQL tenancy) |
| Supabase (EU region) | Evaluated alternative | Similar feature set (branching/backups) — kept as the comparison point; final pick at scaffolding time |
| Fly Postgres | **Rejected** | Cost shape wrong for an idle-heavy solo SaaS; clustered always-on DB unneeded |
| Self-hosted (Hetzner) | Parked | Maximum sovereignty, most ops — reopen only if the managed options chafe |

## Options considered

| Option | Verdict | Why |
|---|---|---|
| **Fly.io, EU region** | **Accepted** | Stateless machines, trivial Elixir deploys, EU regions, dedicated egress IP, custom domains |
| Hetzner + Docker | Rejected for now | Cheapest and most sovereign, but one more layer to run; revisit only if Fly's model chafes |
| AWS/GCP + Kubernetes | Rejected | Solo-SaaS overkill (ADR-0001 already rejected this) |
| US-first PaaS | Rejected | Data residency (criterion 4) |

## Consequences

- **Easier:** deploys are minutes; scale-out is `fly machines run`; the Elixir cluster
  story (libcluster/DNS clustering) is first-class on Fly if LiveView PubSub needs
  multi-node later; EU region pinning satisfies the residency default.
- **Harder / accepted:** vendor concentration on Fly — mitigated by keeping the app
  stateless and the release container portable (boring reverse-proxy escape hatch
  documented); Fly Postgres isn't a fully managed database — the DB hosting decision
  (above) weighs that.
- **Operational:** DNS delegation + cert issuance at setup; egress IP is a paid add-on
  (billed per IP — factored into unit economics); region `fra` is chosen once at setup
  and never mixed.