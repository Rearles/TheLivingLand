# Turn 004 — Context folder + recurring recording behavior

**Date:** 2026-05-21
**User intent:** Ryan rejected one in-flight edit and instructed: (1) make the rejected edit, (2) create a `context/` folder at the repo root to hold the chat history for future Claude Code reference, (3) going forward, every prompt must be logged to that folder.

## Decisions
- A `context/` folder is now the durable log of the design conversation. One file per turn, numbered monotonically.
- The recurring recording behavior is captured as a feedback memory so future sessions follow it automatically.
- Skipped TodoWrite this turn despite reminders — the work was a flat list of file writes, plainly enumerable in the prose, and a todo list would have added friction without adding clarity.

## Actions
- Saved memory `feedback-context-recording.md` to the project's memory directory — codifies the per-prompt logging rule, with rationale and how to apply it.
- Saved memory `project-living-land.md` — captures the project's purpose (Claude Code stress-test for game-from-design-doc), tech stack commitments, asset sourcing, visual style, MVP staging.
- Created `memory/MEMORY.md` indexing both memory files.
- Completed the two pending edits to `docs/frontend-design.md`:
  - §9 Assets — rewrote to reflect pixel-textured 3D (nearest-filtered low-res textures, free CC0 packs + primitives, glb + Draco, pointer to the 3D engine doc).
  - §14 Open Questions — removed the now-resolved 2D vs 3D question; replaced with concrete proposed answers for scaling formula, procgen determinism, mid-run saves, admin CMS, tutorial, and replays.
- Created `context/README.md` — explains the folder's purpose, file naming, per-file template, and how Claude Code should use it at session start/end.
- Backfilled `context/turn-001-initial-design-request.md`, `context/turn-002-2d-vs-3d-discussion.md`, `context/turn-003-commit-to-3d.md`.
- Created this file (`context/turn-004-context-folder-setup.md`).

## Open follow-ups
- `docs/3d-engine-design.md` is still unwritten. It was the planned next major artifact when Ryan redirected to the context-folder setup. Next turn should pick this up — confirming with Ryan before launching since the doc is sizable.
- Frontend doc updates are now consistent with the 3D + pixel-texture decisions; no further reconciliation needed there.
- API design doc (`docs/api-design.md`) has not been re-read against the MVP staging decision. Worth a future pass to note which endpoints/entities are MVP-required vs deferred (annotation, not removal).
