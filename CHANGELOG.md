# Changelog

All notable changes to Freelancer PocketCFO are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and the project will adhere to [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
from the first release. No release exists yet; before `1.0.0` a minor bump is used for
breaking changes as well, marked **Breaking** below.

## [Unreleased]

- **Decision pack 1 — ADRs 0002–0008, status after founder review (2026-10-21):**
  - **ADR-0002 database/multi-tenancy — Accepted (revised):** Phoenix 1.8 changed the
    calculus — generated `scope/2` context helpers and phx.gen.auth/LiveView scope
    threading make app-layer tenant scoping automatic, so **RLS is not adopted**
    (founder: "a need from the past"); the transactional `with_tenant` machinery is
    dropped. Tenant columns + composite indexes stay on plain Postgres; the context API
    is the only data access surface; CI carries a cross-tenant leak test; RLS reopens
    only if a second DB access path ever appears. README.md's tenancy wording aligned
    (safe by construction via the scoped context layer).
  - **ADR-0003 auth/SSO — Accepted:** in-stack `phx.gen.auth` + Ueberauth (Assent kept
    as fallback); **WorkOS removed by founder: no US provider**; recorded insight: OAuth
    providers are a user choice, not ours; Zitadel is the single SAML-time IdP option;
    Keycloak and Auth0 excluded.
  - **ADR-0004 WhatsApp Business — Evaluating (client layer):** direct Meta Cloud API
    settled (no BSP); **three equally viable options for evaluation** — `whatsapp_sdk`,
    `anu` (founder's small-edge prior: closest to what we need), and our own thin
    generated client — exercised via a Livebook against the live API.
  - **ADR-0005 chatbot basics — Evaluating (tenant-context isolation):** architecture
    founder-endorsed (advice-only, human-confirmed action layer: propose → visualize →
    apply → track); **the evaluation was reframed by the founder: the question is how
    context machinery makes cross-tenant interference impossible** — scoped context
    assembly via the ADR-0002 API, per-tenant snapshots, provider-cache awareness, and
    an isolation red-team test suite gate the integration; model comparison (Mistral
    first, OpenRouter-ZDR fallback, Azure EU) runs inside that harness; ZDR + DPA are
    hard gates.
  - **ADR-0006 payments — Accepted:** merchant of record via Paddle (EU-facing entity
    Paddle Payments Ltd, Dublin verified); founder's argument: the B2C share makes EU
    consumer VAT via MoR much simpler; own-MoR/Stripe rejected for launch.
  - **ADR-0007 customer support — Accepted:** WhatsApp-first in-product support —
    onboarding via WhatsApp Flows (native in-chat multi-step forms, verified),
    chatbot-first with founder escalation; Chatwoot self-hosted as recorded step-2.
  - **ADR-0008 hosting — Accepted:** Fly.io EU region, stateless machines, hostname
    `pocket-cfo.titaniumcoder.com`, dedicated egress IP; **Postgres hosting: Fly
    Postgres rejected by founder** (cost shape wrong for an idle-heavy SaaS —
    "nothing 90% of the time"); **Neon (EU) is the default candidate** (founder
    experience + database branching for testing), Supabase the evaluated alternative;
    final pick at scaffolding time.
  - docs/adr/README.md: **Evaluating** added to the ADR lifecycle.

- **Technology decision accepted (ADR-0001, docs/adr/):** the stack is **Elixir + Phoenix
  + LiveView + PostgreSQL**, Go + HTMX as runner-up fallback; accepted trade-off:
  small, expensive 2nd-developer pool (revisited on hiring). The former
  docs/technology-proposal.md is absorbed into ADR-0001 in full — five candidate stacks
  rated against the plan's constraints (WhatsApp-first product, multi-tenant safety by
  construction, solo-founder ops, B2B privacy), evaluation criteria and weights, the
  corrected comparison table, and the rated cross-cutting proposals (Postgres RLS
  tenancy, EU-first hosting, in-stack OIDC now / IdP later, Meta Cloud API direct,
  ZDR LLM adapter, Typst PDF) — and the proposal file removed. docs/adr/README.md fixes
  the ADR convention (MADR format, sequential bottom-up numbering starting at 0001,
  immutable once accepted). Cross-cutting decisions (tenancy, auth, hosting, WhatsApp,
  LLM adapter, PDF) follow as ADR-0002+.

- AGENTS.md working agreement extended: hard rule — merging a PR is a founder action —
  the agent never merges on its own initiative.

- Evaluation history of the technology decision (now recorded inside ADR-0001): Elixir
  1.19's gradual set-theoretic types, Tidewave, and the realtime WhatsApp → app story
  added to the Elixir case; Spring Boot 4 + Kotlin 2.2 + Spring AI 2.0/MCP re-rated B;
  Scala dropped; TypeScript narrowed to TanStack Start vs SvelteKit (SvelteKit backend
  status recorded: Kit 3.0, `.live(...)` remote queries, WebSockets in limbo, no
  background-job story). Comparison table corrected after founder review (complexity
  row inverted → now *simplicity*, higher = fewer moving parts; Elixir LLM support
  raised to 4.5 on the founder's assessment — early-adopter ecosystem, Tidewave's
  inside-the-runtime view eliminating the debugging loop; evaluation spike downgraded
  to an optional sanity check; Kotlin and Go realtime ratings lowered to 2 each).
  Overall: A 4.5, B 3.8, C 3.7, D 3.6 — no real blocker remained for Option A.
  Founder constraint recorded: the JVM candidate is **Kotlin-only, Java excluded**
  (LLM-corpus argument: Java corpus aging for AI-assisted work, Kotlin large, current
  and K2-native; plus founder preference).

- Project scope defined in README.md: the SaaS rewrite of pocket-cfo — what it does in
  principle, and the major adaptations (multi-tenant safety, database instead of git data
  repo, technology deliberately unset, time tracking and invoicing abstractions, WhatsApp
  Business channel with a ZDR chatbot, stricter B2B and data-privacy rules, SSO beyond
  GitHub).
- AGENTS.md establishes the architecture-phase working agreement: documents only, no code,
  no technology assumptions, conventional commits and Keep a Changelog apply from the
  start despite the project sitting effectively at `0.0.0`.