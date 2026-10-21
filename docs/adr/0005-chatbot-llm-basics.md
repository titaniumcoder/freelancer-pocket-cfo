# ADR-0005: Chatbot integration basics — provider-agnostic adapter, ZDR, EU AI Act posture

- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** Rico Metzger (founder) — decision pending
- **Related:** ADR-0001 (stack), ADR-0004 (WhatsApp channel), ADR-0002 (tenancy)

## Context

The chatbot reads tenant finance data (invoices, payments, budgets) and answers through
WhatsApp. Two hard constraints from the plan: **ZDR** (zero data retention — prompt
content must not be stored/used for training by the LLM vendor) and **EU rules**. The
founder is a Bulgarian company; the chatbot processes tenants' personal finance data,
so GDPR applies, and as a deployer of an AI system the EU AI Act applies.

**Verified facts (checked 2026-10):**

- **EU AI Act, Article 50 (transparency) is enforceable since 2026-08-02.** Deployers
  of chatbots must inform users that they are interacting with an AI system (unless it
  is obvious to a reasonably informed, observant user). Penalties scale to €15M/3%
  turnover. A finance-advice chatbot on WhatsApp is *not obviously* AI → **disclosure
  is mandatory**: onboarding message and/or first AI reply must state it. The system is
  **limited-risk** (not high-risk Annex III: no decisions on creditworthiness, no
  employment/essential services) as long as it *advises* and does not autonomously
  execute consequential financial/legal decisions — human confirmation stays in the
  loop for anything that changes invoices, payments or filings.
- **GDPR note:** automated decisions with legal/significant effects are Art. 22
  territory; the chatbot recommends, the human decides — this posture also keeps AI Act
  exposure at limited-risk.
- **ZDR is contractual, per provider.** No-training-on-API-data by default: Anthropic
  API, OpenAI API (with ZDR terms / enterprise), Mistral (EU provider, EU-hosted, DPA
  published). Azure OpenAI adds EU Data Boundary residency. A DPA with each used
  provider is required (Art. 28 GDPR).
- **Mistral** is the EU-native candidate (EU-hosted inference, published DPA, AI Act
  compliance center) — a natural "EU-sovereign" default and strong optics for B2B
  customers.

## Options considered

| Provider | EU residency option | ZDR/no-training | Notes |
|---|---|---|---|
| **Mistral API** | Yes — EU-hosted | Yes (DPA, no training on customer data) | EU sovereign story; model quality for this task to be evaluated in a spike |
| Azure OpenAI (EU Data Boundary) | Yes | Yes (no training; ZDR options) | Strongest paper trail; enterprise-accepted; US-owned |
| Anthropic API | EU endpoint available; residency limits to verify per region | Yes (API data not trained on by default; ZDR tiers) | Strong reasoning models; founder-favored ecosystem |
| OpenAI API | EU residency option exists | Yes with ZDR terms | Big corpus; ZDR must be contractually pinned |
| OpenRouter (ZDR-modeled providers) | Depends on backend | Router-level ZDR configuration | Adds a processor; flexibility for spikes |

**Architectural decision (independent of provider choice):** one adapter module defines
the chatbot's LLM interface; providers plug in behind it. Cost/rate/log fields and the
system prompt stay ours. This is the ADR-0001 "ZDR adapter" promise made concrete.

### What the basic (v1) integration is

1. **Disclosure** (AI Act Art. 50): onboarding message states the assistant is AI;
   footer/first-token disclosure in AI replies.
2. **Scope:** read-only Q&A over the tenant's own data (budgets, invoice status,
   what-if math). No autonomous actions in v1 — mutations (e.g. "mark invoice paid")
   are proposed by the bot and confirmed by the human in chat.
3. **Data minimization:** prompts carry structured tenant context selected by us (not
   raw dumps); no cross-tenant context ever (RLS, ADR-0002).
4. **Logging:** store our own conversation log in the tenant database (needed for
   support/audit); the provider stores nothing (ZDR). Retention policy for chat logs to
   be set with the founder.
5. **Escalation:** handoff path to human support (ADR-0007).

## Decision (proposed)

Provider-agnostic adapter (one module, one behaviour); start with **Mistral or
Azure OpenAI (EU)** as the first provider — final pick in a bounded spike comparing
answer quality on finance-domain tasks — with Anthropic/OpenAI kept switchable behind
the adapter. Limited-risk AI Act posture: mandatory AI disclosure, advice-only + human
confirmation for actions, own conversation log, provider ZDR contracts (DPA) in place
before launch.

## Consequences (if accepted)

- **Easier:** provider can change without touching chat flows; EU-sovereign option
  available; compliance story is short and provable to B2B customers.
- **Harder:** prompt-engineering and context-selection live with us; disclosure copy and
  action-confirmation UX are product surfaces; per-token costs must be metered per
  tenant (feeding ADR-0006).
- **Deferred:** voice/multimodal, agentic tool-calling beyond read-only, fine-tuning —
  all future ADRs if wanted.

## Open questions for the founder

1. First provider preference: Mistral (EU sovereign) vs Azure OpenAI (paper trail) — or
   run the spike and decide on quality?
2. Confirm the v1 scope: read-only advice + human-confirmed actions?
3. Chat-log retention period in our own DB (GDPR storage limitation — e.g. 12 months?
   tied to accounting periods?)