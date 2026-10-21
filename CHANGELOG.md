# Changelog

All notable changes to Freelancer PocketCFO are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and the project will adhere to [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
from the first release. No release exists yet; before `1.0.0` a minor bump is used for
breaking changes as well, marked **Breaking** below.

## [Unreleased]

- **Decision pack 1 — ADRs 0002–0008, status after founder review (2026-10-21):**
  - **ADR-0002 database/multi-tenancy — Evaluating:** direction agreed (plain
    PostgreSQL, tenant_id columns + RLS as fail-closed enforcement, non-BYPASSRLS app
    role); acceptance blocked on a Phoenix integration spike — the founder found the
    transactional `with_tenant` pattern painful before, so the ADR now documents the
    known pain points (transaction-per-request, process-context leaks in LiveView/Oban,
    fail-closed debugging, sandbox friction) and defines spike exit criteria for two
    candidate patterns (explicit `with_tenant/2` vs process-dictionary TenantContext +
    Repo wrapper with dev/test guardrails).
  - **ADR-0003 auth/SSO — Accepted:** in-stack `phx.gen.auth` + Ueberauth (founder's
    known-good tool; Assent verified alive but slow — last release 2025-06 — kept as
    fallback); **WorkOS removed by founder: no US provider**; recorded insight: OAuth
    providers are a user choice, not ours; Zitadel is the single SAML-time IdP option;
    Keycloak and Auth0 excluded.
  - **ADR-0004 WhatsApp Business — Evaluating (client layer):** direct Meta Cloud API
    is settled ("not even a decision", no BSP); the two leading libraries —
    `whatsapp_sdk` (spec-generated, 79 modules/352 typed structs) and `anu`
    (idiomatic, Elixir 1.19+, explicit per-number client) — go through a Livebook-based
    live-API evaluation; own generated library is fallback only.
  - **ADR-0005 chatbot basics — Evaluating (provider/model):** architecture direction
    founder-endorsed (advice-only, human-confirmed action layer: propose → visualize →
    apply → track, the wmc-bexio pattern); provider choice runs a bounded evaluation —
    Mistral first, OpenRouter-with-ZDR fallback, Azure OpenAI EU as paper-trail
    alternative; ZDR contracts + DPA required of the winner; EU AI Act Art. 50
    disclosure posture stands.
  - **ADR-0006 payments — Accepted:** merchant of record via Paddle (EU-facing entity
    Paddle Payments Ltd, Dublin verified; English-law contracts); founder's argument:
    the B2C share (one-man shows, employees paying personally) makes EU consumer VAT
    via MoR much simpler; own-MoR/Stripe rejected for launch (OSS VAT compliance tail);
    dogfooding moved to the tenant-facing invoice engine.
  - **ADR-0007 customer support — Accepted:** WhatsApp-first in-product support —
    mandatory onboarding via WhatsApp Flows (native in-chat multi-step forms, verified
    with endpoint-connected builds), chatbot-first with founder escalation; Chatwoot
    self-hosted as the recorded step-2; Zendesk/HelpScout/Intercom excluded on privacy
    posture.
  - **ADR-0008 hosting — Accepted:** Fly.io on an EU region; stateless machines;
    custom hostname `pocket-cfo.titaniumcoder.com`; dedicated egress IP for provider
    allowlists and a clean processor story; release-artifact deploys. Postgres hosting
    (Fly Postgres vs Neon vs self-hosted) deliberately left to follow the ADR-0002
    spike.
  - docs/adr/README.md: **Evaluating** added to the ADR lifecycle (Proposed →
    Evaluating → Accepted).

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