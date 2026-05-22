# Turn 008 — Design questions for the 3D engine doc

**Date:** 2026-05-22
**User intent:** Ryan asked me to record this turn, create `story/3d-engine-design-doc` (done at end of turn 007), continue with the 3D engine design doc, and ask as many clarifying questions as I need to make the doc the best it can be. I asked four rounds of four questions each. Ryan's answers leaned consistently toward the more ambitious / higher-fidelity options.

## Decisions locked in

### Round 1 — foundations
1. **Engine architecture:** Hybrid — OOP for individual actors (Player, each Enemy), ECS for swarms (projectiles, particles, audio sources).
2. **Pixel-rendering intensity:** Medium — 640×360 internal render target, nearest-neighbor upscale, nearest-filtered textures, modern lighting model.
3. **Movement feel:** Half-Life-ish — smooth accel/decel, walk ~3 m/s, run ~5 m/s, no air control.
4. **MVP "done" bar:** "Playable for fun" — 5–10 min run feels like a real game, tension and satisfaction, sound tuned, HUD readable.

### Round 2 — gameplay systems
5. **Pathfinding:** Full navmesh via recast-navigation-js. (Heaviest option; "right" long-term answer.)
6. **Sweeper logic:** Pathfind through the room graph; always knows the player's room; you can hide briefly but it always catches up.
7. **Procgen connectivity:** Branching tree per PDF spec — 1–4 doors per room, tree/DAG layout. (More complex than linear chain but matches PDF.)
8. **Audio fidelity:** Full 3D spatial audio via Web Audio API PannerNode with distance falloff. Music ducks under sfx.

### Round 3 — combat/UI/movement extras/assets
9. **Damage feedback:** Full physics ragdoll on enemy death via Rapier. Adds rigging complexity but maximally visceral.
10. **HUD rendering:** Angular DOM + Tailwind, styled with pixel-art icons, `image-rendering: pixelated`, and a bitmap-feel webfont. Naturally accessible.
11. **Jump/crouch:** Both in MVP — jump (Space, ~0.5m hop) and crouch (LCtrl, half-height capsule).
12. **Asset packs:** Name primary picks (Quaternius modular dungeon + zombies, Kenney mini dungeon kit, etc.) plus document the swap-out criteria (CC0, glTF or convertible, retextureable, 1u=1m).

### Round 4 — determinism/build/hands/testing
13. **Determinism:** Loose — procgen layout is seeded and reproducible; per-frame randomness uses Math.random. No replays.
14. **Build layout:** Standalone engine npm package in its own GitHub repo, published to npm. Angular client consumes via `npm install @rearles/livingland-engine` (or similar). Most isolation, most overhead.
15. **First-person hands:** Visible hands + weapon by default, toggle in settings to hide.
16. **Testing depth in the doc:** 2–3 representative test snippets per major engine module as ready-to-paste TypeScript. ~10% of doc volume.

## Scope implications

MVP is now larger than the initial "vertical slice" I proposed in turn 003. Ryan's consistent picks of higher-fidelity options expand MVP to include: full navmesh pathfinding, branching procgen, full ragdoll death, jump + crouch, separate npm-published engine package, 3D spatial audio, FP hands rig. I'll honor these choices in the design doc and note where they push MVP beyond the original thin slice.

## Actions
- Asked four AskUserQuestion rounds (16 questions total), recorded all answers above.
- Wrote this turn file.
- About to write `docs/3d-engine-design.md` in full on this story branch.

## Open follow-ups
- **Create the standalone engine GitHub repo** (`livingland-engine` or `@rearles/livingland-engine`). Out of scope for this turn; happens at implementation time.
- **The 3D engine doc itself** — next action.
- **PR back to `develop`** — after the doc is written, committed, and Ryan reviews.
