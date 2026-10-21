# ADR-0003: Authentication and SSO — in-stack Ueberauth now, Zitadel at SAML time

- **Status:** Accepted
- **Date:** 2026-10-21
- **Deciders:** Rico Metzger (founder)
- **Related:** ADR-0001 (stack), ADR-0002 (tenancy — identity maps to tenant)

## Context

Users are freelancers and accountants (EU). Login needs: email/password baseline, social
login (GitHub, Google) via OAuth2/OIDC, and — later, at the first B2B SAML customer —
SAML SSO. Identity data is personal data under GDPR; every third party that sees logins
must be justified.

Founder constraints (2026-10-21):

- Auth0 is **too bloated** — excluded.
- Keycloak is out: running a Java IdP in the open is exactly the ops weight this project
  avoids.
- The candidate IdPs are **Zitadel** and ~~WorkOS~~ — WorkOS removed (2026-10-21):
  **no US provider.**
- Library question: is Assent still alive? The founder's known-good experience is
  Ueberauth.
- Founder insight recorded: OAuth providers (GitHub, Google, …) behind Ueberauth are
  **user-oriented** — the choice of OAuth is a user choice, not ours. Our responsibility
  is the platform baseline, not the provider list.

## Decision

**Start in-stack: `phx.gen.auth` baseline + Ueberauth (or Assent) for social OIDC.
Identity lives in our own Postgres — no third party sees logins. At the first B2B SAML
customer, Zitadel (self-hosted, EU) is the only recorded IdP option — WorkOS is removed,
no US provider.**

The OAuth-vs-IdP asymmetry, made explicit: social login strategies are a *user* choice —
tenants bring whichever identity they prefer, and strategies are added on user demand,
not by roadmap. A dedicated IdP (Zitadel) enters only when a *tenant organization*
requires SAML federation — a genuinely different requirement.

### Library status (verified 2026-10-21, hex.pm)

| Library | State | Notes |
|---|---|---|
| **Ueberauth** | Stable, long-established plug-based OAuth/OIDC framework; strategy ecosystem (`ueberauth_github`, `ueberauth_google`) | Founder's known-good tool; default choice |
| **Assent** | Alive but slow-moving: last release **June 2025**, ~2.3M total downloads; strategy framework usable independently of Pow | Solid OIDC correctness (PKCE, JWKS); alternative if Ueberauth's model chafes |

Both are acceptable; **Ueberauth is the default** (founder familiarity, strategy
ecosystem), Assent is the documented fallback. Either way the strategy layer sits behind
a thin internal module so swapping strategies is not an app-wide event.

### What "in-stack" means concretely

- `phx.gen.auth` generates email/password accounts, session tokens, and mail — audited
  and owned.
- Social login = OAuth2/OIDC callbacks creating/linking local identities. Email is the
  identity join key; verification required.
- Sessions as server-side tokens in Postgres (revocable, auditable).
- Magic-link login optional, reusing the same token machinery.
- Per ADR-0002: the user record belongs to a tenant (freelancer) or maps into one
  (accountant) — the plug resolves tenant before `with_tenant/2` opens scope.

### IdP options for the SAML-time upgrade (recorded, not decided)| Option | Fit | Trade-off |
|---|---|---|
| **Zitadel** (self-hosted EU, or Zitadel cloud EU region) | Organizations as a first-class concept — natural for B2B tenants; OIDC+SAML; EU-native | One more service to run (but Go binary, lighter than Keycloak) |
| **WorkOS** | ~~Fastest SAML~~ | **Removed by founder (2026-10-21): no US provider.** Identity is the most sensitive tenant data; a US processor fails the privacy bar by default |
| Keycloak | — | Excluded by founder (ops weight in the open) |
| Auth0 | — | Excluded by founder (bloated) |

## Consequences

- Zero extra infrastructure at launch; no third party sees logins (privacy criterion).
- SAML is real work later — accepted, because B2B SAML customers do not exist yet.
- If/when Zitadel is added, in-stack identities migrate via OIDC linking; the
  in-stack account remains the fallback login.
