# Technology Proposal

Status: **Proposed — decision pending.** This document rates the candidate stacks against
the project's constraints and recommends one. Nothing here is decided until an explicit
decision is recorded (see [ADR-0001 placeholder](adr/0001-technology-decision.md), to be
written when the decision lands).

## Context

Solo-founder SaaS, written by [Rico Metzger](https://titaniumcoder.com). Profile summary
relevant to this decision:

- Strongest historical stack: **Java/Kotlin/Scala with Spring Boot**, Kafka, Kubernetes —
  all enterprise-scale, 5★, but used in employer contexts, not solo products.
- Recent momentum (2026): two **Elixir/Phoenix** projects (a Phoenix LiveView legacy
  migration with 321 tests; an agent-coordination platform) and the pocket-cfo rewrite
  candidate itself was a solo **Go** product.
- Frontend: Angular/React 4★, Svelte/Vue 3★ — capable, not passionate.
- Postgres 5★; agentic tooling and LLM integration deep and current.

## Requirements derived from the plan

In rough priority order — the product's shape, not the implementer's preferences:

1. **Multi-tenant SaaS, safe by construction.** Tenancy isolation must be enforced below
   the application layer: a forgotten `WHERE tenant_id = …` must not be able to leak
   another tenant's data.
2. **WhatsApp Business channel is the main channel.** Registration and daily interaction
   happen through WhatsApp (Meta Cloud API webhooks); the chatbot is the selling feature.
   This means: reliable webhook ingestion, idempotent message processing, background
   jobs, and a conversational AI integration — all first-class product machinery.
3. **Chatbot with ZDR.** LLM calls under a zero-data-retention arrangement; the LLM
   integration sits behind a provider-agnostic abstraction.
4. **Stricter B2B rules, bigger data-privacy focus.** EU data residency as the default;
   every third-party dependency that can see tenant data must be justified.
5. **Database is the main channel.** A relational model fits the domain (invoices,
   payment state, budget windows, actuals — the original's schemas were all shapes of
   this); auditability and revertibility are re-created in the database (append-only
   change logs, never-edit-issued-invoices invariants).
6. **Abstractions for time tracking and invoicing** (and legislations, export targets).
7. **SSO** — OIDC/OAuth2 (GitHub, others), B2B SAML later.
8. **Solo founder.** Iteration speed, one-person ops, low total moving-parts count, and
   synergy with AI coding tools weigh more than hyperscale read throughput. A finance
   SaaS for freelancers will be latency-tolerant and never Twitter-scale.

## Evaluation criteria and weights

| Criterion | Weight | Rationale |
|---|---|---|
| Solo iteration speed & ops burden | 25% | One person; every moving part is a liability |
| Product fit (webhooks, chat, jobs, admin UI) | 20% | WhatsApp-first product machinery must be first-class |
| Profile fit & velocity now | 20% | Recent, current practice — not résumé depth |
| Safety-by-construction support | 15% | Tenancy isolation, typing discipline for money |
| Ecosystem maturity for this domain | 10% | OIDC/SAML, PDF, LLM SDKs, integrations |
| Longevity & maintainability | 10% | The product should outlive fashion |

## Candidate stacks

### A. Elixir + Phoenix (+ LiveView) + PostgreSQL

- **Profile fit & velocity (4.5/5):** current daily practice; two 2026 projects; the
  dBase migration proved disciplined delivery with tests. Not the 5★ of Spring, but the
  freshest hand.
- **Product fit (5/5):** Phoenix is arguably the best HTTP/webhook server in any
  language (Bulletproof connection handling, supervision trees restart failed webhook
  handlers without dropping the server); Oban is best-in-class Postgres-backed job
  processing (retention/retries/queues — exactly what idempotent WhatsApp message
  processing needs); LiveView removes the entire SPA stack for the admin/web surface —
  one person ships interactive UI at backend speed; Phoenix channels for realtime.
- **Safety by construction (4/5):** immutability by default, pattern matching on message
  shapes, `with` pipelines; Postgres RLS for tenancy (see cross-cutting); no nulls.
  Dynamic typing is the weakness for money math — mitigated with schemas, typespecs,
  and property tests.
- **Solo ops (5/5):** one BEAM binary, one Postgres, one VM. No node_modules, no build
  pipeline zoo, releases are self-contained. The original repo's AGENTS.md
  "stdlib-first" instinct maps to Phoenix's batteries-included philosophy.
- **Ecosystem (3.5/5):** Assent (OIDC incl. GitHub/Google), Ecto, OpenAI/Anthropic HTTP
  clients trivial (raw HTTP is idiomatic here); WhatsApp has only community SDKs —
  irrelevant since the Cloud API is plain HTTPS webhooks. PDF: shell out to Typst.
  SAML: weaker than JVM.
- **Longevity (4/5):** Erlang/OTP stability measured in decades; hiring pool small but
  the founder *is* the hiring pool.
- **Weighted score: ≈ 4.5**

### B. Kotlin + Spring Boot 3 + PostgreSQL + React/Angular

- **Profile fit (5/5):** the 5★ core stack, certified, deepest experience.
- **Product fit (3/5):** everything exists (Spring Security OAuth2 is superb, Spring
  Integration/webhooks fine, Quartz) but nothing is *first-class*: the admin UI becomes
  a second frontend build with a second language, doubling the solo surface. Chat
  streaming is fine. The stack is engineered for teams, not one person.
- **Safety by construction (4/5):** strong typing, but nullable-by-default legacy of the
  platform; RLS equally available.
- **Solo ops (2.5/5):** JVM memory, Kubernetes muscle memory pulls toward infra weight;
  two deployables (API + SPA) minimum; iteration slower per feature.
- **Ecosystem (5/5):** unmatched; nothing is ever missing.
- **Longevity (5/5):** enterprise-forever.
- **Weighted score: ≈ 3.8**

### C. Go + server-rendered HTML (HTMX) + PostgreSQL

- **Profile fit (4.5/5):** pocket-cfo proves solo delivery end-to-end; the domain model
  (budget windows, actuals, invoice immutability) is already battle-tested in Go.
- **Product fit (3/5):** `net/http` is excellent for webhooks; River (Postgres jobs) and
  go-oidc are solid; but interactive admin UI, realtime, and chat streaming are all
  hand-built. Everything in the original was hand-built too — that was its charm and
  its cost.
- **Safety by construction (4.5/5):** explicit errors, no nulls, single binary;
  interface-based abstractions for time tracking/invoicing natural.
- **Solo ops (4.5/5):** cross-compiled static binary, trivial deploys.
- **Ecosystem (4/5):** all necessities exist, batteries not included.
- **Longevity (4.5/5):** Google-stable, minimal churn.
- **Weighted score: ≈ 4.0** — the runner-up, and the continuity argument (reuse of
  pocket-cfo's domain logic and the non-vibecoded discipline) is real.

### D. TypeScript — Next.js (or NestJS) + Prisma + PostgreSQL

- **Profile fit (3.5/5):** TS 4★, frontend is the weaker hand.
- **Product fit (4/5):** LLM/WhatsApp SDKs land here first; Vercel-style PaaS is fast.
- **Safety by construction (3/5):** the weakest for a finance product — runtime
  flexibility, `any` leakage, client/server boundary discipline is opt-in.
- **Solo ops (4/5):** PaaS-friendly but privacy posture (EU residency) narrows platform
  choice; dependency churn (npm) is a standing tax.
- **Ecosystem (5/5) / Longevity (4/5).**
- **Weighted score: ≈ 3.7**

### E. Scala + ZIO/http4s + PostgreSQL

- **Profile fit (3/5):** 5★ language on paper, but last daily use 2021; pure-FP solo
  velocity is a gamble.
- **Weighted score: ≈ 3.2** — brilliant type system for this domain, wrong moment.

## Recommendation

**Option A — Elixir + Phoenix + LiveView + PostgreSQL**, with **Option C (Go + HTMX)**
as the serious runner-up.

The deciding arguments:

1. **The product is a chat product with a finance backend, not the reverse.** Webhook
   reliability, job processing, and realtime UI are the core; Phoenix is the stack where
   those are not add-ons.
2. **One language, one deployable, one VM.** LiveView deletes the SPA question entirely
   for a solo founder — the admin portal, tenant dashboard and settings UI ship at
   backend speed with zero second-stack maintenance.
3. **Current practice beats résumé depth.** Spring is the deepest tool in the box, but
   the last two years of solo delivery ran through Elixir and Go; the freshest, most
   AI-tooling-synergistic hand is the Elixir one.
4. **Go stays viable** if continuity with pocket-cfo's domain code weighs more than the
   product-machinery argument — this is the one genuine trade-off worth the founder's
   explicit deliberation.

## Cross-cutting decisions (mostly stack-independent, rated)

These are recommended targets that hold for A and C alike; they become ADRs once the
stack decision lands.

### Database: PostgreSQL with Row-Level Security tenancy — recommended

Three models, rated against "safe by construction":

| Model | Isolation strength | Solo-ops cost | Verdict |
|---|---|---|---|
| Shared schema, `tenant_id` + **RLS enforced on every table** | Highest: even a query that forgot the tenant predicate cannot cross tenants; the app sets the tenant per transaction/session and RLS refuses the rest | Low | **Recommended** |
| Schema-per-tenant | High but migration sprawl grows linearly with tenants; N schemas × M migrations | Medium | Rejected |
| Database-per-tenant | Strongest blast radius, but connection/migration/backups × tenants; SaaS-grade bookkeeping burden | High | Rejected for now; escape hatch if a large B2B tenant demands it |

RLS alone is not the whole story: the app connects as a role that *cannot* bypass RLS
(`FORCE ROW LEVEL SECURITY`), sets `SET LOCAL app.tenant_id` per transaction, and has no
superuser credentials in production. Then tenancy is structural, matching README.md's
"cross-tenant access must be structurally impossible, not merely forbidden".

### Hosting: EU-first, boring, few moving parts — recommended: Fly.io or Hetzner

| Option | Privacy posture | Solo-ops | Verdict |
|---|---|---|---|
| **Fly.io (EU region)** | EU regions, app-level control | Low — app + managed Postgres in one platform | **Recommended default** |
| **Hetzner + Docker + managed Postgres (or self-hosted)** | Data center in DE/FI, cheapest | Medium — one more thing to run, maximum sovereignty | **Recommended alternative** |
| AWS/GCP + Kubernetes | Fine, EU regions | High — k8s is 5★ experience but solo-SaaS overkill | Rejected for now |
| US-first PaaS (Vercel/Railway) | Residency negotiable, not guaranteed | Low | Rejected on criterion 4 |

### Auth/SSO: in-stack OIDC now, dedicated IdP when B2B demands SAML

| Option | Fit now | Fit at B2B-SAML time | Verdict |
|---|---|---|---|
| In-stack OIDC (Assent / go-oidc) for GitHub, Google, email | Zero extra infrastructure, no third party sees tenant data | SAML later = real work | **Recommended start** |
| Zitadel (self-hosted, EU) | One more service to run | Full OIDC/SAML B2B, EU-native | Recommended upgrade path |
| WorkOS / Auth0 / Keycloak | Fast SAML | WorkOS = US processor (privacy trade-off); Keycloak = ops weight | Deferred; revisit at first B2B SAML customer |

### WhatsApp Business: Meta Cloud API direct

Direct Cloud API integration (plain HTTPS webhooks) beats BSP middlemen (Twilio,
360dialog) on both privacy (one fewer data processor; BSP is a fallback for number
provisioning problems) and ZDR simplicity. The message-processing layer must be
idempotent (webhook redelivery is guaranteed) — Oban/River-style jobs handle that.

### Chatbot / LLM: provider-agnostic adapter + contractual ZDR

The ZDR property is contractual and per-provider, not architectural; the architecture
contribution is the adapter boundary (one module defines the chatbot's LLM interface).
Candidates, all offering no-training/ZDR arrangements under appropriate agreements:
Azure OpenAI (EU residency, strongest paper trail), Anthropic API (no training on API
data by default), OpenAI API with ZDR terms, or a router (OpenRouter) with ZDR-modeled
providers. Decision deferred to an LLM-specific ADR; the stack decision does not depend
on it.

### PDF generation: Typst (deterministic, versioned input)

Invoice PDFs rendered from declarative templates by a pinned Typst binary —
reproducible, no headless-browser ops burden. Same reasoning for any language choice.

## Next steps after the decision

1. Write `docs/adr/0001-technology-decision.md` recording the choice and the rejected
   alternatives' reasons.
2. Follow-up ADRs: multi-tenancy model (RLS), auth/SSO, WhatsApp integration, LLM/ZDR
   adapter, hosting.
3. Only then: repository scaffolding — the first code arrives after the ADRs, not before.