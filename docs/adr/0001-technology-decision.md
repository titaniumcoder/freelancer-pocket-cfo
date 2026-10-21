# ADR-0001: Elixir + Phoenix (+ LiveView) + PostgreSQL as the technology stack

- **Status:** Accepted
- **Date:** 2026-10-21
- **Deciders:** Rico Metzger (founder)
- **Supersedes:** —
- **Superseded by:** —

## Context

The rewrite of pocket-cfo is a solo-founder, multi-tenant B2B SaaS whose main channel is
a WhatsApp chatbot with a zero-data-retention LLM arrangement, backed by a relational
domain model (invoices, payment state, budget windows, actuals) with EU-first data
privacy. The full constraint analysis is in
[docs/technology-proposal.md](../technology-proposal.md); its weighted evaluation scored
five candidate stacks and found no remaining blocker for the Elixir option after the
founder withdrew the LLM-codegen concern and corrected the simplicity rating.

The requirements that shaped this decision, in priority order: multi-tenancy that is
safe by construction (isolation below the application layer), WhatsApp webhook
ingestion with idempotent message processing and background jobs as first-class product
machinery, realtime updates from the WhatsApp channel into the app (the founder ↔
accountant live-update scenario is the demo moment), one-person ops, and synergy with
AI coding tools.

## Decision

**Elixir + Phoenix + LiveView + PostgreSQL** is the project's technology stack.

Rejected alternatives (from docs/technology-proposal.md, weighted scores 2026-10):

- **Kotlin + Spring Boot 4 (never Java)** — ≈ 3.9. Deepest founder experience and now a
  superb LLM/agent story (Spring AI 2.0 GA, MCP starters), but the admin surface becomes
  a second frontend build with a second language, and it has no realtime backbone for
  the WhatsApp → app story.
- **Go + HTMX (server-rendered)** — ≈ 3.7 overall / ≈ 4.0 weighted stack fit, the safe
  runner-up with the continuity argument (pocket-cfo's battle-tested domain code).
  Structurally loses the realtime criterion; webhook UI, jobs, and chat streaming would
  all be hand-built.
- **TypeScript (TanStack Start / SvelteKit)** — ≈ 3.6. Best LLM-tooling ecosystem, but
  no background-job story at all (external worker required) and WebSockets remain in
  limbo in SvelteKit — critical gaps for a webhook-first chat product.
- **Scala + ZIO/http4s** — dropped by founder decision (2026-10): fading language, not
  worth keeping alive.

The accepted trade-off: Elixir's hiring pool is small and expensive, so onboarding a
future second developer will cost more than on a JVM or TS stack. Accepted for a
solo-founder start; to be revisited if/when hiring begins.

## Options considered

| Option | Verdict | Why |
|---|---|---|
| Elixir + Phoenix + LiveView + PostgreSQL | **Accepted** | Product machinery (webhook reliability, Oban jobs, PubSub/LiveView realtime) is in the stack's bones; one language, one deployable, one VM; founder's freshest daily practice; Elixir 1.19 gradual set-theoretic types close the money-math typing gap; Tidewave gives the coding agent a runtime-inside view no other stack has |
| Kotlin + Spring Boot 4 | Rejected | Two deployables / two languages for one solo founder; no realtime backbone; strongest only on ecosystem and résumé depth |
| Go + HTMX | Rejected (runner-up) | Everything product-critical hand-built; HTMX painful for WS/SSE; kept as the continuity fallback |
| TypeScript (TanStack Start / SvelteKit) | Rejected | No jobs story, WS in limbo, npm churn tax |
| Scala + ZIO/http4s | Rejected | Fading language; maintenance liability |

## Consequences

- **Easier:** the realtime WhatsApp → app scenario is a one-liner of architecture
  (webhook → chatbot → database → PubSub → every open LiveView re-renders live). Solo
  ops are one BEAM binary, one Postgres, one VM — no node_modules, no build pipeline
  zoo, self-contained releases.
- **Harder:** smaller, costlier talent pool if a second developer is hired; major-version
  Elixir upgrades are the one standing rough edge (mitigated by LLM-written upgrade
  guidance and the 1.19 type trajectory).
- **Neutral / deferred:** cross-cutting decisions (Postgres RLS tenancy, EU-first
  hosting, in-stack OIDC, Meta Cloud API direct, ZDR LLM adapter, Typst PDF) are
  recommended in the technology proposal and become their own ADRs (0002+); an optional
  bounded Elixir tooling spike (Claude Code + Tidewave) remains a confidence-building
  sanity check, not a gate. Repository scaffolding — the first code — starts only after
  the follow-up ADRs land.