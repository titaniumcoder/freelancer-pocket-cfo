# ADR-0007: Customer support — WhatsApp-first, no heavyweight helpdesk

- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** Rico Metzger (founder) — decision pending
- **Related:** ADR-0004 (WhatsApp Cloud API — the channel support would share),
  ADR-0005 (chatbot escalation path)

## Context

Solo-founder support for a B2B SaaS whose users already live in the product's WhatsApp
chat. Support volume at launch is small and personal; the customers' natural support
surface is the channel they already use — WhatsApp. Any helpdesk that *doesn't* speak
WhatsApp would force a second channel; any US-hosted helpdesk sees tenant support
content (privacy posture from ADR-0001: every third party that can see tenant data must
be justified).

**Verified facts (checked 2026-10):**

- Open-source/self-hosted helpdesks that integrate **WhatsApp natively**: **Chatwoot**
  (built around messaging; WhatsApp via the same Meta Cloud API — self-host free,
  cloud per-agent pricing, "Captain" AI add-on) and **Zammad** (mature full ticketing;
  hosted in Germany available). **FreeScout** (lightweight shared inbox, email-first).
- Hosted/EU SaaS: **Crisp** (France), hosted **Chatwoot**. Hosted/US: Zendesk,
  Intercom, Help Scout ($25–75/user/mo) — all strong, all US processors, all priced per
  seat.
- **Peppermint**, a common self-hosted pick, was archived by its owner 2026-07-17 —
  excluded from the shortlist.

## Options considered

| Option | WhatsApp fit | Privacy | Cost | Verdict |
|---|---|---|---|---|
| **In-product support (v0): support = the product's own WhatsApp chat** | Native — support conversations *are* product conversations; chatbot answers known issues, escalates to the founder's inbox | Best: zero new processors; nothing leaves our DB | €0 | **Recommended start** — for a solo founder, support volume is tiny; the chatbot handoff (ADR-0005) already exists |
| **Chatwoot, self-hosted** (on the same EU infra) | Native WhatsApp integration; team inbox, labels, SLAs when staff grows | Good: self-hosted, no new processor if we run it | Free + ops (~one more container) | **Planned step 2** when support volume justifies a team inbox |
| Zammad, self-hosted | Email+channels; WhatsApp integration less central | Good | Free + heavier ops | Alternative at step 2 if ticketing (over messaging) fits better |
| Crisp (FR SaaS) | WhatsApp channel available | EU processor, DPA | Per-seat SaaS | Reasonable zero-ops middle ground |
| Zendesk / Intercom / Help Scout (US SaaS) | Good | US processors of support content | $20–90/agent/mo | Rejected for now on privacy posture and cost |

### Feasibility of the v0 path

- The WhatsApp number is ours (ADR-0004); inbound support messages already flow through
  our webhook with idempotent ingestion. Support routing is a message-classification
  rule ("this user needs a human" / bot self-identifies limits) plus a LiveView
  "founder inbox" surface — the realtime WhatsApp → LiveView story from ADR-0001 makes
  this cheap.
- Migration path is clean: Chatwoot can attach to the *same* Cloud API number later;
  history stays ours either way (export from our DB).

## Decision (proposed)

Start in-product: support lives in the same WhatsApp conversation, chatbot first,
human escalation into a minimal founder inbox (LiveView). Add **Chatwoot self-hosted**
as the recorded step-2 upgrade when a second person or real ticket volume appears. No
US helpdesk SaaS for now.

## Consequences (if accepted)

- **Easier:** zero new processors, zero new channel, zero seat fees; the support
  transcript is tenant data we already hold under the same retention rules.
- **Harder:** support UX (labels, macros, SLAs, multi-agent) must grow with us — the
  Chatwoot step-2 plan exists precisely for that moment.
- **Trade-off recorded:** self-hosting Chatwoot later adds one more service; accepted
  as it arrives with a real second person.

## Open questions for the founder

1. Confirm v0 in-product support (vs starting with Chatwoot right away)?
2. Should the support inbox also be reachable by email (most SaaS users expect
   support@ — small addition via SMTP in-stack)?