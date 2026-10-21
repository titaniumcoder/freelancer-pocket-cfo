# Architecture Decision Records

This folder records the architecture decisions of Freelancer PocketCFO as short,
self-contained documents.

## Format

We follow the **MADR** pattern ([Markdown Any Decision Records][madr],
`v3.0.0-rc1` template): each ADR is a markdown file with the sections *Context*,
*Decision*, *Options considered* (with pros/cons), and *Consequences*, and carries a
`Status` header (Proposed → Accepted → Superseded by ADR-XXXX). ADRs are immutable
once Accepted — a changed decision gets a new ADR that supersedes the old one.

## Numbering

ADRs are numbered sequentially starting from `0001`; the next decision gets the next
higher number. Numbers are never reused, so the sequence only ever grows bottom-up:
`0001-…`, `0002-…`, `0003-…`. The filename is
`NNNN-kebab-case-title.md`.

## Lifecycle

- A decision is **Proposed** while the discussion is ongoing.
- It becomes **Accepted** when the founder confirms it (here: in conversation with the
  agent, recorded in the ADR and landed on the standing architecture-phase PR).
- Superseded ADRs keep their number and get a `Superseded by ADR-XXXX` status.

[madr]: https://github.com/adr/madr