# Turn 013 — Condense context folder + CLAUDE.md auto-load

**Date:** 2026-05-22
**User intent:** Ryan was concerned the growing number of `turn-NNN-*.md` files would become unwieldy for Claude Code to consume at session start. Requested: (1) condense them into a smaller set of documents that are easier to read in one go, and (2) set up a rule so Claude Code automatically reads them at session start without Ryan having to ask.

## Decisions
- **`context/PROJECT-SNAPSHOT.md`** is the new authoritative session-start document. It replaces reading all individual turn files. Captures: project identity, MVP scope, visual style, tech stack, all locked-in architecture decisions (engine, API, frontend), repo workflow, current state, and what's next.
- **`CLAUDE.md`** at the repo root is the auto-load mechanism. Claude Code reads CLAUDE.md automatically at every session start (it is the project's instruction file). CLAUDE.md instructs Claude to read `context/PROJECT-SNAPSHOT.md` and documents the context recording rule.
- Turn files are **not deleted** — they remain as a historical archive. New sessions load only the snapshot.
- **`context/README.md`** updated to reflect the new structure (snapshot-first, turn files as archive).
- Snapshot must be updated at the end of any session where decisions, completed work, or current state change.

## Actions
- Created `context/PROJECT-SNAPSHOT.md` — comprehensive condensed summary of all decisions and state from turns 001–012 (~200 lines).
- Created `CLAUDE.md` at repo root — session-startup instructions + context recording rule.
- Edited `context/README.md` — updated the "How Claude Code should use this folder" section to point at the snapshot first.
- Created this turn file.

## Open follow-ups
- Same as turn-012:
  - Decide whether to merge open PR or continue doc deep-dive (§10 ragdolls/combat suggested next)
  - After merge: scaffold implementation
  - Wire up Tier 1 security scanning when code exists
  - Create standalone engine npm package repo
  - CLT update (non-blocking)
