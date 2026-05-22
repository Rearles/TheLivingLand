# context/

Per-turn record of the design conversation between Ryan (the project owner) and Claude Code. This folder exists so that future Claude Code sessions can pick up the work without re-deriving decisions from scratch and without having to re-read the full transcript.

## File layout

```
context/
├── README.md          # this file
├── turn-001-*.md      # earliest turn
├── turn-002-*.md
└── turn-NNN-*.md      # most recent turn
```

Each turn file follows this template:

```markdown
# Turn NNN — <short title>

**Date:** YYYY-MM-DD
**User intent:** <paraphrased; not a verbatim paste of the prompt>

## Decisions
- <key decisions made this turn>

## Actions
- Created / edited <file path> — <what changed>
- Asked clarifying question(s): <summary>

## Open follow-ups
- <items deferred to a later turn>
```

## How Claude Code should use this folder

- **At the start of a new session**, read the highest-numbered turn file (and a few before it if context is missing) to recover where the project is.
- **At the end of every turn**, append a new turn file. Check the highest existing number first; do not overwrite earlier turns.
- Keep the files focused on *decisions and actions*, not transcript dumps. The PDF and the `docs/` folder are the source of truth for the game design; these files are the source of truth for the *meta* (what we agreed to, what we tried, what's deferred).

## Related

- Game design source: [../The Living Land.pdf](../The%20Living%20Land.pdf)
- API design: [../docs/api-design.md](../docs/api-design.md)
- Frontend design: [../docs/frontend-design.md](../docs/frontend-design.md)
- Persistent memory: `/Users/ryan/.claude/projects/-Users-ryan-Desktop-LivingLandGame-TheLivingLand/memory/`
