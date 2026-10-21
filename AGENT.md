# AGENT.md

**Current phase: pure architecture.** We write documents, not code. Do not assume,
pick, or scaffold any technology — no language, framework, database, hosting or protocol
decision is made yet; "the technology is not yet set" is a project principle (see
README.md). Every deliverable in this phase is a document: architecture notes, decision
records, domain models, requirement definitions. Code, configs, CI pipelines, Dockerfiles
and dependency files do not belong in this phase.

## What this project is

The non-vibecoded SaaS rewrite of
[titaniumcoder/pocket-cfo](https://github.com/titaniumcoder/pocket-cfo). The original's
ARCHITECTURE.md and docs are useful domain references, but its implementation decisions
(single Go binary, git data repo, two-repo split) are *inputs to rethink*, not defaults.
README.md carries the canonical feature scope and the major adaptations (multi-tenant
safety, database instead of git repo, time tracking and invoicing abstractions, WhatsApp
Business channel with a ZDR chatbot, stricter B2B and data-privacy rules, SSO beyond
GitHub).

## Conventions

- **Conventional Commits on `main`.** `feat:`, `fix:`, `feat!:` (or a
  `BREAKING CHANGE:` footer), etc. In this phase a "feature" is a document that adds or
  settles architecture knowledge — `docs:` is fine for work-in-progress, but anything
  that lands a real decision is `feat:` (or `fix:` when it corrects one). Commits that
  land on `main` are the versioning signal, so write them correctly from the start.
- **CHANGELOG.md follows Keep a Changelog 1.1.0.** Every user-visible step (here:
  every doc that adds or changes a decision) gets a line under `[Unreleased]` at commit
  time. Releases turn that section into the version's entry.
- **Semantic versioning applies from the start.** There are no releases yet, so the
  project sits effectively at `0.0.0` — below the first tag. That does not suspend the
  logic: version proposals come from the conventional commits since the last tag. Before
  `1.0.0`, a minor bump is used for breaking changes as well; those are marked
  **Breaking** in the changelog. Pre-releases carry a suffix (`0.1.0-rc.1`), each
  candidate gets its own changelog section, and promoting a series folds its
  `[X.Y.Z-rc.N]` sections into the single `[X.Y.Z]` entry.
- **Releases are cut locally, on request, never automatically.** Analyze commits since
  the last tag, categorize them, propose a version — and only cut the release after
  explicit confirmation of the proposal. For the first release the proposal is `v0.1.0`
  unless the user names the version themselves; the proposal says so either way.
- **Plan in small, complete subtasks.** Break any non-trivial task into independently
  verifiable units; one commit per subtask; push at the end of the plan, not per step.
- **Merging a PR is a founder action.** Branches and PRs are the working state; the
  agent never merges — not its own PRs, not reverts, not admin fixes. A PR is merged
  when the founder says "merge" in conversation or merges it himself in the UI.
  During the architecture phase this collapses to: **one standing PR stays open until
  the phase is finished**; work lands on its branch.
- **Documents explain the why.** A fact from outside the code — a legal requirement, a
  vendor behaviour, a domain rule — belongs in a document under `docs/`, written once,
  so that the later implementation can be the mechanism. During the architecture phase
  that document is the product.