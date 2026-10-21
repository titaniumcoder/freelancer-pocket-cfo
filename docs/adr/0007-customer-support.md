# ADR-0007: Customer support — WhatsApp-first in-product, Chatwoot as the recorded step-2

- **Status:** Accepted
- **Date:** 2026-10-21
- **Deciders:** Rico Metzger (founder)
- **Related:** ADR-0004 (WhatsApp Cloud API — shared channel), ADR-0005 (chatbot escalation)

## Context

Solo-founder support for a B2B SaaS whose users already live in the product's WhatsApp
chat. The founder's framing: support that matches the product's philosophy — **guide
users with the mandatory initial forms, then handle the rest conversationally** — is
naturally in-product, because the support conversation and the product's context are the
same channel.

## Decision

**Build support into the product's WhatsApp channel first: structured onboarding via
WhatsApp Flows, conversational support after, chatbot-first with escalation to the
founder. No helpdesk product at launch.**

### How good is in-product WhatsApp support, honestly?

**Strong for this product, by construction:**

- The user never leaves the channel they already use for the product itself — zero
  new-context cost for a support request.
- **WhatsApp Flows** (verified 2026-10-21, Meta developer docs) are *native, structured,
  multi-step forms that run inside the chat* — text inputs, dropdowns, checkboxes,
  date pickers — instead of linking out to a web page. Endpoint-connected Flows submit
  structured form data to our own endpoint. This is exactly the "mandatory initial
  forms" mechanism: onboarding (business data, tax setup, invoicing preferences) becomes
  a guided in-chat flow with validation, not a PDF-of-questions.
- Support requests arrive with the tenant's full context (plan, onboarding state, recent
  actions) attached — something a standalone helpdesk has to be fed.
- The chatbot (ADR-0005) handles first-line answers from product docs; unresolved
  threads escalate to the founder's human inbox with full transcript.

**Known limits (accepted at launch scale):**

- No SLA tooling, no ticket state machine, no per-agent analytics beyond our own log.
- The founder is the only human agent — the "inbox" is a LiveView over our conversation
  records, replies via the same Cloud API client.
- Historical transcript UI beyond chat is ours to build — but conversation records are
  already first-class data in our Postgres.

These limits are exactly why step 2 is recorded rather than dismissed:

### Step 2 (when volume justifies it): Chatwoot, self-hosted

- **Chatwoot** (open source, can self-host on EU infra, WhatsApp Cloud API integration
  native) becomes the agent surface: team inbox, assignment, SLAs, canned responses.
- **Zammad** is the fallback if ticket-centric workflows fit better later.
- **Crisp** (French SaaS) is the SaaS alternative if self-hosting weight is the
  bottleneck.
- Zendesk / Help Scout / Intercom: **excluded** — US processors, personal data of EU
  users off-EU by default; the privacy criterion applies to support conversations too
  (they carry the same tenant data as the product itself).

### Onboarding design (the Flows layer)

- Flow 1 — account setup: business identity, VAT status, invoicing details (the
  "mandatory initial forms").
- Flow 2 — first invoice/budget walkthrough (guided, form-validated).
- Flow payloads submit to our Flows endpoint → validated → stored tenant-scoped →
  confirmed in chat. Failed validation re-prompts in-flow, not out-of-band.
- Flows require Meta business verification and maintained message quality (operational
  prerequisites, tracked in ADR-0004's Meta-side requirements).

## Consequences

- Support is a product feature, not a procurement: the conversation log is tenant data
  under our tenancy model (ADR-0002), the chatbot escalates per ADR-0005, and every
  support exchange is auditable like the rest of the domain.
- Near-zero marginal cost and no new processors at launch.
- The Chatwoot step-2 is a deliberate, pre-agreed expansion path — the migration is
  data-shaped (conversations out of our log into Chatwoot), so the in-product version
  must keep conversations exportable from day one.
