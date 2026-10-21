# Changelog

All notable changes to Freelancer PocketCFO are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and the project will adhere to [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
from the first release. No release exists yet; before `1.0.0` a minor bump is used for
breaking changes as well, marked **Breaking** below.

## [Unreleased]

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