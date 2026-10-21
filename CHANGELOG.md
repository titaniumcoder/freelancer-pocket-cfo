# Changelog

All notable changes to Freelancer PocketCFO are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and the project will adhere to [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
from the first release. No release exists yet; before `1.0.0` a minor bump is used for
breaking changes as well, marked **Breaking** below.

## [Unreleased]

- Technology proposal (docs/technology-proposal.md): five candidate stacks rated against
  the plan's constraints (WhatsApp-first product, multi-tenant safety by construction,
  solo-founder ops, B2B privacy); recommendation Elixir + Phoenix + LiveView + PostgreSQL
  with Go + HTMX as runner-up, plus rated cross-cutting proposals (Postgres RLS tenancy,
  EU-first hosting, in-stack OIDC now / IdP later, Meta Cloud API direct, ZDR LLM adapter,
  Typst PDF). Decision pending — recorded as ADR when it lands.
- Technology proposal re-examined after founder feedback: Elixir 1.19's gradual
  set-theoretic types, Tidewave, and the realtime WhatsApp → app story added to the
  Elixir case (with second-developer cost and LLM-codegen evaluation noted as open
  risks); Spring Boot 4 + Kotlin 2.2 + Spring AI 2.0/MCP re-rated B; Scala dropped;
  TypeScript reframed as a vendor-lock-free framework investigation (TanStack Start,
  Remix/RR7, Hono, NestJS). New comparison table added; no real blocker found for the
  founder's Elixir-leaning gut decision.
- Founder constraint recorded: the JVM candidate is **Kotlin-only, Java excluded**
  (LLM-corpus argument: Java corpus aging for AI-assisted work, Kotlin large, current
  and K2-native; plus founder preference).

- Project scope defined in README.md: the SaaS rewrite of pocket-cfo — what it does in
  principle, and the major adaptations (multi-tenant safety, database instead of git data
  repo, technology deliberately unset, time tracking and invoicing abstractions, WhatsApp
  Business channel with a ZDR chatbot, stricter B2B and data-privacy rules, SSO beyond
  GitHub).
- AGENT.md establishes the architecture-phase working agreement: documents only, no code,
  no technology assumptions, conventional commits and Keep a Changelog apply from the
  start despite the project sitting effectively at `0.0.0`.