# Claude Code instructions — The Living Land

## Session startup (required)

At the start of every new session, read this file first, then read:

```
context/PROJECT-SNAPSHOT.md
```

That file is the authoritative summary of all decisions made, the current project state, and what's next. It replaces needing to read the individual `context/turn-NNN-*.md` files. Only read turn files if you need the detailed history behind a specific decision.

## Context recording rule (required, every turn)

At the end of every prompt/response, create a new `context/archive/turn-NNN-<short-title>.md` file. The file numbering is monotonically increasing — check the highest existing number first inside `context/archive/`. Never overwrite an earlier turn file.

Template:

```markdown
# Turn NNN — <short title>

**Date:** YYYY-MM-DD
**User intent:** <paraphrased; not a verbatim paste of the prompt>

## Decisions
- <key decisions made this turn>

## Actions
- Created / edited <file path> — <what changed>

## Open follow-ups
- <items deferred to a later turn>
```

Write permissions for `context/` are pre-approved in `.claude/settings.local.json`.

**Also update `context/PROJECT-SNAPSHOT.md`** if this turn's decisions, completed work, or current state differ from what the snapshot says. The snapshot must stay current so the next session starts with accurate information.

## What this project is

A first-person 3D dungeon crawler called "The Living Land". See `context/PROJECT-SNAPSHOT.md` for the full picture: tech stack, architecture decisions, MVP scope, visual style, git workflow, and current build status.

## Key design docs

- `docs/api-design.md` — REST API (ASP.NET Core 8)
- `docs/frontend-design.md` — Angular 17+ frontend
- `docs/3d-engine-design.md` — 3D engine (Three.js + Rapier, 2 511 lines)
- `CONTRIBUTING.md` — git branching and workflow
