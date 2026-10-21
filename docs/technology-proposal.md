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
- **Safety by construction (4.5/5):** immutability by default, pattern matching on message
  shapes, `with` pipelines; Postgres RLS for tenancy (see cross-cutting); no nulls.
  The former dynamic-typing weakness is being patched by the language itself: **Elixir
  1.19 (October 2025) ships gradual set-theoretic types in the compiler** — sound and
  gradual, with an explicit `dynamic()` escape hatch, type checking of anonymous
  functions and of protocol dispatch, and up to 4× faster compilation. For a finance
  product this is the right trajectory: the type system will catch money-math mistakes
  the compiler previously ignored, and it is delivered by the language, not by bolting
  on a third-party type layer.
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
- **Founder additions (2026-10, decision discussion):**
  - **Tidewave.** An MCP server with runtime-level tools for Phoenix: the coding agent
    sees the *running* app — precise framework traces, browser evaluation, LiveView
    diagnostics, in-browser agent (active project, v0.9.x as of September 2026). The
    founder uses and loves it in coding sessions. No other candidate stack has an
    equivalent: this is a real development-experience moat for Option A.
  - **Realtime updates, WhatsApp → app.** Phoenix PubSub + LiveView make the
    founder's scenario a one-liner of architecture: a message arrives via the WhatsApp
    webhook, the chatbot updates the database, and every open LiveView (the founder's,
    the accountant's browser) re-renders *live*. Accountant says something on the
    phone, founder fixes it in the app, accountant watches it update — no refresh, no
    polling, no extra realtime stack. In every other candidate this is hand-built.
  - **Negative — second developer.** The Elixir hiring pool is small and
    expensive; onboarding a future second developer onto Phoenix/OTP will be harder
    and costlier than onto JVM or TS stacks. Accepted trade-off for a solo-founder
    start; revisit if/when hiring.
  - **Open — LLM code-gen quality for Elixir.** LLM training corpora skew towards
    mainstream languages (TS, Java, Python); Elixir assistance quality is plausibly
    weaker and must be *evaluated*, not assumed — a small spike (build a slice of the
    domain with Claude Code / Tidewave and judge) belongs in the decision follow-ups.
    Tidewave mitigates on the runtime/feedback side regardless.

### B. Kotlin + Spring Boot 4 + PostgreSQL + React/Angular — Kotlin only, Java excluded

**Founder constraint (2026-10, critical):** this candidate is **Kotlin, never Java**. If
the JVM wins at all, it wins as Kotlin. Rationale: the founder's LLM assessment — the
Java training corpus is showing its age for AI-assisted work, while Kotlin's corpus is
large (Android-driven), current, and K2-native — and the founder's clear preference.
Spring Boot 4 makes this clean: it rebased on a **Kotlin 2.2 baseline** with K2 compiler
support, ships a first-class `kotlinx-serialization` starter, and the Spring AI 2.0
surfaces (chatbot, MCP server starters, annotations) are usable idiomatically from
Kotlin. Java interop remains a safety net, not the working mode.

- **Profile fit (5/5):** the 5★ core stack in its preferred dialect, certified, deepest
  experience; and the founder's question — "aren't we on Spring Boot 4 now?" — is
  right, and it matters: **Spring AI 2.0 went GA in June 2026** with dedicated MCP
  Server Boot Starters and MCP annotations. The AI/agent story that used to be the JVM's
  weak spot is now first-class and superbly documented. The runtime LLM integration
  (chatbot, MCP surface) would be *easier* here than anywhere else.
- **Product fit (3/5):** everything exists (Spring Security OAuth2 is superb, webhooks
  fine, Spring AI 2.0 for the chatbot) but the UI story is unchanged: the admin portal
  becomes a second frontend build with a second language, doubling the solo surface.
  The realtime WhatsApp → app scenario needs extra machinery (WebSocket/SSE wiring or
  an added realtime layer) where Phoenix has it in its bones.
- **Safety by construction (4/5):** strong typing, but nullable-by-default legacy of the
  platform; RLS equally available.
- **Solo ops (2.5/5):** JVM memory, Kubernetes muscle memory pulls toward infra weight;
  two deployables (API + SPA) minimum; iteration slower per feature.
- **Ecosystem (5/5):** unmatched — now explicitly including the LLM/agent axis
  (Spring AI 2.0 GA, MCP starters, MCP Java SDK — consumed from Kotlin; Android-scale
  Kotlin corpus for AI-assisted coding).
- **Longevity (5/5):** enterprise-forever.
- **Weighted score: ≈ 3.9** (LLM-support gains bought by the update, solo-ops and
  two-surface costs unchanged)

### C. Go + server-rendered HTML (HTMX) + PostgreSQL — the definitive safe runner-up

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
  pocket-cfo's domain logic and the non-vibecoded discipline) is real. Nothing discovered
  in the 2026-10 re-examination moved it; "safe runner-up" stands.

### D. TypeScript — framework under investigation

The language is rated a good one by the founder; the prior framing (Next.js + Prisma)
was rejected by the founder outright — "Next.js is a disaster" and the Vercel coupling
is a vendor-lock concern. The *investigation* is therefore narrower: which
**vendor-lock-free TypeScript full-stack setups** exist, and can one avoid the setup
pain the founder dreads? Shortlist for that investigation:

- **TanStack Start** — Vite-based, self-hostable anywhere, React Server Components
  without Next.js; young but backed by the TanStack ecosystem.
- **Remix / React Router v7 (framework mode)** — batteries-included full-stack React,
  deploys to any Node/Deno/Bun host; the most "no setup pain" React option.
- **Hono (backend) + Vite + React SPA** — runtime-agnostic, minimal, deploy-anywhere;
  but the setup is hand-rolled, which collides with the founder's setup-pain aversion.
- **NestJS (backend) + Vite SPA** — Angular-style structure the founder knows;
  batteries included, but heavier and the SPA remains a second surface.

None of these changes the fundamental ratings: the finance-safety concern and the
client/server boundary discipline are language-runtime properties, not framework ones.
Re-estimate after the framework investigation.

### D. TypeScript — Next.js (or NestJS) + Prisma + PostgreSQL

- **Profile fit (3.5/5):** TS 4★, frontend is the weaker hand.
- **Product fit (4/5):** LLM/WhatsApp SDKs land here first; Vercel-style PaaS is fast.
- **Safety by construction (3/5):** the weakest for a finance product — runtime
  flexibility, `any` leakage, client/server boundary discipline is opt-in.
- **Solo ops (4/5):** PaaS-friendly but privacy posture (EU residency) narrows platform
  choice; dependency churn (npm) is a standing tax.
- **Ecosystem (5/5) / Longevity (4/5).**
- **Weighted score: ≈ 3.7**

### E. Scala + ZIO/http4s + PostgreSQL — dropped

Dropped by founder decision (2026-10): the language is fading and the project will not
spend effort keeping it alive. Kept here for the record only.

## Comparison table (2026-10 re-examination)

Stack fit reuses the proposal's weighted score. Onboarding a second developer, LLM
support (both AI-assisted coding *and* the runtime chatbot/agent story), and complexity
(running parts × solo setup pain) are rated 1–5, higher is better. **Overall** folds the
new criteria into the founder's decision lens.

| Criterion | A: Elixir/Phoenix | B: Kotlin/Spring 4 | C: Go/HTMX | D: TypeScript |
|---|---|---|---|---|
| Stack fit for founder (weighted) | **4.5** | 3.9 | 4.0 | 3.5 (pending framework pick) |
| Onboarding a 2nd developer | 2.5 — small, expensive pool | **5** — largest enterprise pool | 4 — big pool, boring stack | 4.5 — biggest pool overall |
| LLM support (AI coding + runtime) | 3.5 — code-gen corpus weaker, **needs evaluation**; Tidewave mitigates runtime-side | **4.5** — huge corpus + Spring AI 2.0/MCP | 4 — huge corpus, thin agent tooling | **5** — SDKs and corpora first-class |
| Complexity (moving parts, setup pain) | 2 — one language, one surface, one VM | 3 — framework weight + SPA second build | 2 — few parts, but machinery hand-built | 3 — setup pain real; npm churn standing tax |
| Realtime WhatsApp → app story | **5** — PubSub/LiveView in the bones | 3 — extra wiring | 3 — hand-built | 3.5 — framework-dependent |
| **Overall** | **4.2** | 3.9 | 4.0 | 3.6 |

Read as: the founder's gut (Elixir) has no *real blocker* in the table — the two open
risks are explicit and bounded: the 2nd-developer cost (accepted for now, revisit at
hiring) and the LLM-for-Elixir evaluation (a spike, scheduled before final commitment;
a bad result does not sink the stack, it informs how much of the code is hand-written
vs agent-generated).

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
3. **LLM-for-Elixir evaluation spike** (before final commitment): build one bounded
   slice of the domain (e.g. the budget-window module) with Claude Code + Tidewave on
   Elixir, judge assistance quality honestly. A bad outcome does not sink the stack; it
   sets the hand-written vs agent-generated mix.
4. Only then: repository scaffolding — the first code arrives after the ADRs, not before.