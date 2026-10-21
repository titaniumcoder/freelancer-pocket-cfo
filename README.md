# Freelancer PocketCFO

The non-vibecoded rewrite of [titaniumcoder/pocket-cfo](https://github.com/titaniumcoder/pocket-cfo), built as a real SaaS.

> **Status: project start.** The technology stack is **not yet set** — this is a deliberate
> decision, not an omission. Architecture and implementation decisions will follow; nothing
> in this README below the "what it does" level is settled.

## What it does

In principle the same job as [pocket-cfo](https://github.com/titaniumcoder/pocket-cfo): a
freelancer's finance tracker and invoicing tool.

- **Finance tracker** — predicted income from an hourly rate, optionally from real
  tracked hours, superseded by real invoiced income once an invoice is issued; a planned
  expense budget; and the money actually spent, reconciled from bank statements and shown
  beside the plan.
- **Invoicing** — invoices with rendered PDFs, a read-only viewer, and per-client access.
- **Accountant-facing structure** — mandatory invoice wording checked against an
  accountant-owned catalog; validated, auditable data with a clear distinction between
  *planned* and *actual* numbers.
- **Uploads inbox** — statements (CSV, PDF, photos) parsed into transactions, reviewed,
  then committed.
- **Agent-facing API** — read tools so an agent can answer questions about the company's
  money, with writes kept behind the same auditable path as everything else.

## What changes: major new features and adaptations

This is a major rewrite. The single-tenant, git-as-database design of the original becomes
a multi-tenant SaaS product. Concretely:

- **Multi-tenant SaaS, safe by construction.** Tenancy isolation is a core design
  constraint from day one, not bolted on: the scoped context layer is the only data
  access surface, every query is tenant-scoped by construction (Phoenix scopes), and
  cross-tenant access is proven impossible by CI tests, not hoped away.
- **A database instead of a git data repository.** The original's two-repo split (code
  repo + private data repo, audited via git commits) is replaced by a real database as the
  main channel. Auditability and revertibility — the strengths of the git design — must be
  re-created inside the SaaS (immutability rules, change logs, traceable edits).
- **Technology not yet set.** The stack, database, hosting and frameworks are open
  questions. Everything above is technology-agnostic on purpose.
- **Time tracking becomes part of the problem.** The original consumed Toggl hours
  read-only; here, time tracking is an integrated feature, with abstractions planned so
  multiple time-tracking sources (and, later, external ones) can plug in.
- **Invoicing abstractions.** Invoice generation and bookkeeping will be abstracted so
  different legislations, formats and export targets can be added later.
- **WhatsApp Business as the main channel.** The app's primary selling feature is a
  WhatsApp Business channel: freelancers register through it and interact with a chatbot.
  The chatbot will be handled via a **ZDR** (Zero Data Retention) setup and a safe,
  explicitly-designed integration — chat traffic never becomes a data liability.
- **Stricter B2B rules, stronger data privacy.** As a SaaS it follows stricter B2B
  requirements than the original, with a bigger focus on data privacy: minimal data
  retention, clear processing purpose, and privacy-respecting defaults throughout.
- **Single Sign On.** SSO like the current GitHub-based login, but provider-agnostic —
  probably not only GitHub, even though the target audience clearly is software
  engineers.

## Origin

This is the **non-vibecoded version of
[titaniumcoder/pocket-cfo](https://github.com/titaniumcoder/pocket-cfo)**. That repo is
frozen and archived — a purely vibecoded codebase (written end to end with coding agents,
never restructured by hand) that served its four months in production and is kept
read-only there. Everything this project inherits conceptually comes from it; everything
it changes is listed above.