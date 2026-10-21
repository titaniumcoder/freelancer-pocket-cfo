# ADR-0005: Chatbot basics — advice-only, human-confirmed action layer, ZDR + EU AI Act posture

- **Status:** Evaluating — architecture direction (advice-only + action layer) is
  founder-endorsed; **the evaluation is about tenant-context isolation**: data from one
  tenant must never spill into another tenant's prompts, context, or logs — under any
  circumstance
- **Date:** 2026-10-21
- **Deciders:** Rico Metzger (founder)
- **Related:** ADR-0001 (stack), ADR-0002 (tenancy), ADR-0004 (WhatsApp channel)

## Context

The chatbot reads tenant finance data (invoices, payments, budgets) and answers through
WhatsApp. Hard constraints: **ZDR** (the LLM vendor must not retain or train on prompt
content), **EU rules** — the founder is a Bulgarian company and the **EU AI Act is
directly in force for transparency obligations since 2026-08-02** — and the founder's
conviction that the product's actions must be *human-advised*, never autonomously
applied.

Founder direction (2026-10-21), modeled on his wmc-bexio work: build a **layer on top
for the actions** — visualize proposed actions, apply them explicitly, keep track of
what was applied. And on providers: Mistral is naturally promising if it can compete;
models must be evaluated; fallback is **OpenRouter with ZDR**.

## Decision

**The chatbot is advice-only. It proposes; it never applies. Every state-changing action
flows through an explicit, tracked application layer with a human confirmation step.**
Provider access goes through a thin, provider-agnostic adapter; every provider in the
candidate list must carry a contractual ZDR/no-training commitment.

### The action layer (the architectural center of this ADR)

1. **Propose.** The chatbot (or a scheduled agent) produces structured *action
   records*: type (e.g., categorize payment, draft invoice correction, set budget
   window), payload, reasoning trace, confidence.
2. **Visualize.** Proposed actions surface in a LiveView review queue (and as an
   in-chat summary card). The user sees exactly what would change — before/after
   states, not just prose.
3. **Apply.** An explicit user confirmation applies the action through the same domain
   modules the UI uses — the chatbot has no privileged write path.
4. **Track.** Applied actions land in an append-only log with actor (user vs. agent),
   timestamp, before/after, and the originating conversation. Domain invariants stand
   (issued invoices are never edited; corrections are new records).

This design simultaneously delivers: human oversight (AI Act), auditability (finance
SaaS), reversibility bookkeeping, and a clean test surface (the action layer is pure
domain logic, chatbot-independent).

### EU AI Act posture (limited-risk system)

- The product's AI is a **limited-risk AI system** under Regulation (EU) 2024/1689:
  no Annex III high-risk category applies (it informs, it does not decide or score).
- **Article 50 transparency is enforceable since 2026-08-02** and applies regardless of
  risk class: users must be informed that they are interacting with an AI system.
  Concretely: the chatbot discloses it is AI-assisted at chat start and on demand; the
  app labels AI-generated content. This is product copy, not infrastructure.
- Provider due diligence: we require a DPA and no-training commitments from the model
  vendor (Art. 50's supply-chain expectations; providers must in turn disclose AI in
  their interfaces).
- Reassessment trigger: if the chatbot's role expands toward autonomous decisions or
  scoring, the classification review reopens (recorded here as the tripwire).

### Provider candidates (adapter behind, all under ZDR contracts)

| Provider | EU posture | ZDR | Assessment |
|---|---|---|---|
| **Mistral** | EU-native, EU residency, published DPA, no training on customer data (verified 2026-10) | Yes, contractually | **Default first candidate** — founder's lean; evaluate model quality on finance-domain tasks |
| **OpenRouter (ZDR-modeled providers)** | Routing layer; ZDR depends on the routed provider's terms | Configurable per provider | **Fallback** if Mistral's models are too weak on our tasks |
| **Azure OpenAI (EU region)** | EU residency, strongest enterprise paper trail | Enterprise terms | Alternative with the best compliance story |
| **Anthropic / OpenAI APIs** | US processors; Anthropic trains not on API data by default | Terms-dependent | Usable only with signed ZDR terms |

**Open evaluation (before acceptance):** the same finance-domain task set — Q&A over
sample tenant data plus action-proposal quality — is run against **Mistral's current
models**, **one OpenRouter-routed alternative**, and **Azure OpenAI (EU)**. The winner
becomes the launch provider behind the adapter; whichever wins must sign ZDR terms and
a DPA. The adapter makes the outcome a config change, not a rewrite. Founder's prior:
Mistral is naturally promising — *if it can compete* on quality; OpenRouter with ZDR is
the stated fallback.

### The evaluation that matters: tenant-context isolation

The founder reframed the open evaluation: **not primarily model quality — the question
is how the chatbot's context machinery makes cross-tenant interference impossible.**
Data from tenant A must not be able to reach tenant B's conversation — not in prompts,
not in retrieved context, not in logs, not via model-side caching side channels.

Requirements the evaluation must validate (design, then adversarial test):

1. **Context assembly is scoped by construction.** Everything the prompt carries is
   built exclusively through the ADR-0002 scoped context API — the chatbot cannot
   assemble context it could not query as the tenant. The Scope is derived from the
   WhatsApp sender → tenant mapping, never from conversation content.
2. **One mechanism, whole product.** The chatbot gets no privileged query path; if a
   human UI can't see it, neither can the bot. Isolation is reviewable in one place.
3. **Per-tenant context snapshots.** Conversation history and retrieved documents are
   stored tenant-scoped (own Postgres) and re-read through the scope per turn — no
   shared process state, no cross-tenant memoization.
4. **Provider-side caching awareness.** Model providers may cache prompt prefixes;
   caching never crosses tenants by construction (prefixes are tenant-specific), and
   ZDR terms cover retention. Recorded, then verified against the winning provider's
   docs.
5. **An isolation red-team test suite.** Adversarial prompts attempting cross-tenant
   recall ("show me invoices from [other tenant's name]"), mixed-tenant seeding, and
   log inspection asserting no foreign data appears in prompts, responses, or stored
   conversations. These tests gate the integration.

Model-quality comparison (Mistral vs OpenRouter-routed vs Azure OpenAI) runs *inside*
this harness — on tasks built per-tenant — so the winner is evaluated under the same
isolation guarantees it will ship with. ZDR terms + DPA remain a hard gate for any
provider.

### Data flows

- Chat logs live in **our Postgres** only (tenant-scoped, per ADR-0002). The LLM vendor
  receives prompt content per request under ZDR; no vendor-side storage.
- Retention of our own chat logs is a GDPR storage-limitation question — **open item**:
  proposal is 12 months rolling, tied to accounting periods; founder to confirm.
- Context assembly (tenant docs, summaries) happens in-stack, through the scoped
  context API (see the isolation requirements above); only the composed prompt leaves
  the boundary.

## Consequences

- The AI Act posture is cheap and durable: disclosure + human oversight + contractual
  ZDR covers the limited-risk obligations.
- No autonomous writes means the worst a hallucination can do is a wrong *proposal* —
  bounded by the review queue.
- Provider swaps are configuration, enabling the evaluation spike to be rerun cheaply
  as models improve (Mistral's trajectory is the founder's stated bet).
- Slightly more engineering than "just call the API" — the action layer is the product,
  not overhead.
