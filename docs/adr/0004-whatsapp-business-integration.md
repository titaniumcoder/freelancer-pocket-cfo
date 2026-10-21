# ADR-0004: WhatsApp Business integration — Meta Cloud API direct

- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** Rico Metzger (founder) — decision pending
- **Related:** ADR-0001 (stack: Oban for idempotent ingestion), ADR-0005 (chatbot)

## Context

WhatsApp is the product's main channel: registration, daily interaction, and the
chatbot all run through it. The founder is a Bulgarian company; the product's tenants
(freelancers, accountants) chat with the product's WhatsApp Business number.

**Verified facts (Meta developer docs, checked 2026-10):**

- Meta charges **per message delivered** (not per conversation) since **2025-07-01**.
  Categories: marketing, utility, authentication, service. Rates vary by market ×
  category; there is **no platform access fee** — partners (BSPs) add their own fees on
  top.
- **Business-initiated messages require pre-approved templates.** Free-form replies are
  only allowed inside the **24-hour customer-service window** opened by an inbound user
  message. Utility templates sent *inside* an open customer-service window are free.
- Since **2026-10-01** free-form messages in an open window are billed as *service*
  messages (a new pricing page effective 2026-10-01; exact EU market rates to be
  verified at implementation).
- Authentication templates are restricted to one-time codes/verification flows.
- Redelivery of webhooks is guaranteed → ingestion must be **idempotent** (message IDs
  deduplicated) — already an ADR-0001 requirement, handled by Oban-style jobs.

## Options considered

| Option | Cost structure | Privacy/data | Verdict |
|---|---|---|---|
| **Meta Cloud API direct** | Pure Meta per-message rates, no middleman margin | One data processor (Meta) — GDPR processor agreement via WhatsApp Business Solution terms; Meta is an independent/joint controller for delivery metadata (unavoidable on WhatsApp) | **Recommended** |
| BSP (Twilio, Vonage, 360dialog, SleekFlow…) | Meta rates + per-message partner fee (e.g. Twilio adds $0.005/message) + BSP platform fees | Two processors (BSP sees message content) | Rejected as default; kept as fallback if number provisioning or Meta business verification stalls |
| WhatsApp Business App (the consumer-ish app) | Free | No API, no chatbot, no idempotent ingestion; manual only | Rejected — cannot be the product's channel |

### What is possible (feasibility constraints)

- **One product number, many tenants.** Tenants chat *with* the product's number; the
  message's phone number maps to a tenant. A per-tenant business number is *not*
  possible (Meta verification per number; cost; the product is the WhatsApp presence).
  Multi-tenant routing happens by WhatsApp user identity → tenant mapping in our DB.
- **Inbound-initiated first:** the natural flow is the freelancer messaging the product
  number first (opens the 24h window; service messages free in-window); outbound
  nudges/summaries outside the window need utility/marketing templates (approval flow
  via Meta template review; utility in-window is free, so proactive templates should be
  timed near user activity where possible).
- **Number provisioning:** a dedicated Bulgarian SIM-less number can be registered to
  the Cloud API; display name and Meta Business verification are required before scale
  limits rise. Business verification takes days — schedule it early.
- **Meta Business Agent:** a new category appeared on the 2026-10-01 pricing page (AI
  agents on WhatsApp); its rules/rates are new — tracked, not load-bearing for this
  decision.
- **Shared inbox/team features do not exist in the Cloud API** — any multi-agent human
  inbox is built by us (LiveView) or comes from a third-party helpdesk (ADR-0007
  territory; Chatwoot would wrap the same Cloud API number — i.e. direct integration
  there too, not a BSP).

## Decision (proposed)

Integrate the Meta Cloud API directly: plain HTTPS webhooks (ingested idempotently via
Oban), templates for business-initiated messages, single product number, inbound-first
flows. No BSP in the path.

## Consequences (if accepted)

- **Easier:** lowest per-message cost; one processor fewer than any BSP path; the
  webhook/jobs machinery is exactly what Phoenix+Oban are best at (ADR-0001).
- **Harder:** template approval and Meta business verification are real process work
  with lead times; Meta pricing page changes (e.g. 2026-10-01) must be tracked — the
  billing category of every outbound message becomes part of the domain model
  (cost-per-message accounting per tenant).
- **Risk recorded:** if Meta verification or number provisioning stalls, a BSP
  (360dialog or Twilio) is the documented fallback, not a redesign.

## Open questions for the founder

1. Accept single-product-number + inbound-first as the interaction model?
2. Which outbound message types do you want at launch (summary nudges? invoice status
   pushes?) — this determines the template set to submit for approval.
3. Is the per-message cost pass-through to tenants wanted (usage-based billing line,
   see ADR-0006)?