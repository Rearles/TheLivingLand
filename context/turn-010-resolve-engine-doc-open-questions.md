# Turn 010 — Resolve §24 open questions in the 3D engine design doc

**Date:** 2026-05-22
**User intent:** Continue the doc review per Option D (§24 first, then deep-dive). Ryan answered two rounds of four AskUserQuestion items covering all eight §24 open questions, then asked me to continue the edits after a Claude subscription upgrade interruption.

## Decisions (from review rounds)

| §24 item | Decision |
| -------- | -------- |
| Pipe durability per hit | Data-driven via API; MVP default 1, override via `Item` record |
| Sweeper release timing | Per-day API-driven via `RunDescriptor.sweeperReleaseSeconds`; default 90 s if API omits |
| Ragdoll configuration | Per-enemy configs (`PIPE_ZOMBIE_RAGDOLL` 9 bones, `MINI_ZOMBIE_RAGDOLL` 5 bones, `SWEEPER_RAGDOLL` 11 bones) |
| Pause Sweeper audio | Duck Sweeper proximity loop to 30 %, mute other spatial loops, music -6 dB |
| Mid-run save | Confirmed off |
| Stamina UX | Always visible on HUD |
| FP hands fallback | MVP ships with weapon-only if pack doesn't ship the four per-weapon poses |
| Sweeper spawn metric | BFS-farthest AND ≥ 8 m world distance; fall back to pure BFS otherwise |

## Actions

Edits applied to `docs/3d-engine-design.md`:

1. **§9.4** — Added FP hands fallback paragraph (weapon-only if pack doesn't ship per-weapon poses).
2. **§10.4** — Replaced single `HUMANOID_RAGDOLL` with three per-enemy configs plus a `RAGDOLLS_BY_KIND` lookup. Mini Zombie uses simpler 5-bone (single-segment limbs) skeleton; Sweeper gets bulkier 11-bone (with separate head + neck) skeleton.
3. **§12.2** — Updated Sweeper release narrative to describe BFS-farthest + 8 m world floor with fallback.
4. **§14.5** — Updated `farthestRoomFrom` to compute both BFS-best-with-floor and fallback BFS-only, returning fallback if no room clears the floor.
5. **§17** — Rewrote `TimerService` to take `initialRemainingSeconds` in the constructor; engine reads from `RunDescriptor`. Updated trailing prose to make the API-driven nature explicit.
6. **§18.3** — Replaced the one-line pause description with a full audio policy table (music -6 dB, Sweeper duck 30 %, other spatial loops mute, one-shots finish).
7. **§19** — Renamed `PIPE_DURABILITY_LOSS_PER_HIT` to `..._DEFAULT` with a comment that the live value is data-driven from the API. Renamed `SWEEPER_RELEASE_S` to `SWEEPER_RELEASE_DEFAULT_S` with API-fallback note. Added `SWEEPER_MIN_WORLD_DIST_M = 8`.
8. **§24** — Rewrote "Open Questions" as "Resolved Decisions" — each item briefly states the decision and points at the section that implements it.

Also configured `.claude/settings.local.json` with a `Write(context/turn-*.md)` permission allow rule so future per-turn context files don't surface a permission prompt. Created `.gitignore` excluding the local settings file. Ryan's instruction: "You never have to ask me to allow you to write to the turn-NNN...md files."

Also captured the credit-balance context: Ryan ran out of Claude credits mid-edit and upgraded the subscription. The interruption created a series of small clarifying prompts ("How do I fix my credit balance?", "What agents are available?") between the first edit (§9.4) and the resumption of the remaining seven edits.

## Open follow-ups

- **Deep-dive phase of Option D** — Ryan may want to drill into specific sections of the doc next. Offering this after committing the §24 resolution work.
- **Commit + push** — staging this turn's edits, the new `.claude/settings.local.json` (only the `.gitignore` is tracked since the settings file itself is gitignored), and the turn-010 file together on `story/3d-engine-design-doc`. Branch already tracks origin.
- **PR** — held until deep-dive phase concludes or Ryan signals the doc is ready.
- **Memory update** — `feedback-context-recording.md` now correctly reflects what's expected, but worth noting in there that the permission is wired up in `.claude/settings.local.json` so future Claude Code sessions don't re-create it.
