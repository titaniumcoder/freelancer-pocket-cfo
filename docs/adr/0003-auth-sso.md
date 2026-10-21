# ADR-0003: Authentication and SSO — in-stack OIDC first, IdP at B2B-SAML time

- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** Rico Metzger (founder) — decision pending
- **Related:** ADR-0001 (stack), ADR-0002 (tenancy — identity must map to tenant)

## Context

Users are freelancers and accountants (B2B, EU). Login needs: email/password or magic
link as baseline, social login (GitHub, Google) via OAuth2/OIDC, and — later, when the
first B2B SAML customer appears — SAML SSO. All identity data is personal data under
GDPR; every third party that sees credentials or identity claims must be justified.
EU AI Act is irrelevant here; EU data-residency posture matters for the identity store.

## Options considered

| Option | Fit now | Fit at B2B-SAML time | Privacy posture | Verdict |
|---|---|---|---|---|
| **In-stack: Phoenix `phx.gen.auth` + Assent (OIDC/OAuth2)** | Zero extra infrastructure; identity lives in the same Postgres, same RLS regime, same backups; Assent is the maintained Elixir multi-provider framework (OAuth 1.0/2.0, OIDC strategies; GitHub, Google work out of the box) | SAML = real work later | Best: no third party sees tenant identity data | **Recommended start** |
| Zitadel (self-hosted, EU, built-in orgs/tenancy) | One more service to run (it *is* a whole product) | Full OIDC + SAML, multi-tenant orgs as a first-class concept, EU-native | Good (self-hosted) | Recommended upgrade path when SAML is real |
| Keycloak / authentik (self-hosted) | Heavy ops for a solo founder; Java/JVM sidecar | Full OIDC/SAML/federation | Good (self-hosted) | Rejected now — ops weight without current need |
| Auth0 / WorkOS / Clerk (US SaaS) | Fastest SAML | Fast SAML | US processors; SCCs needed; identity is the most sensitive data in the system | Rejected on privacy posture for now |
| Managed EU IdP SaaS | — | — | Varies | Not needed at current scale; revisit at SAML time |

### Feasibility notes

- `phx.gen.auth` generates the password/session/magic-link baseline with LiveView
  support — this is the Phoenix-batteries-included path, consistent with the stack
  rationale in ADR-0001.
- Assent handles the OIDC/OAuth2 strategies; it needs no separate service.
- Session model: server-side sessions (ETS/database) in the same database; MFA can be
  added in-stack later (TOTP) — no external dependency required.
- The SAML bridge later is bounded work: Zitadel can sit in front *only* for SAML
  customers (proxy OIDC) without migrating everyone.

## Decision (proposed)

Start in-stack (`phx.gen.auth` + Assent), identity stored in the product database under
the same tenancy and privacy regime. When the first SAML customer materializes, stand up
Zitadel (self-hosted, EU) as the dedicated IdP for B2B federation — not a migration of
the existing user base.

## Consequences (if accepted)

- **Easier:** one service to run; identity data never leaves the product's Postgres;
  password reset, sessions and audit logging share the app's existing machinery.
- **Harder:** no SAML on day one (accepted — no customer demands it yet); in-stack auth
  code is the app's responsibility (phx.gen.auth covers the dangerous parts).
- **Trade-off recorded:** if B2B SAML demand arrives early, the Zitadel path is the
  planned answer, not a re-architecture.

## Open questions for the founder

1. Social providers at launch: GitHub + Google + email/magic link — is that the set?
2. Password + social coexistence (recommended), or social-only?
3. Confirm Zitadel as the *named* future IdP, or leave the SAML-time choice open?
