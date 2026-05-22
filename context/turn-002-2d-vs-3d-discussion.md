# Turn 002 — 2D vs 3D technical discussion

**Date:** 2026-05-21
**User intent:** Ryan asked two questions: what are the technological advantages of 2D over 3D, and why is first-person 3D in the browser "a real undertaking"? This was a discussion turn — no code changes expected.

## Decisions
None made this turn; it was an information-gathering exchange so Ryan could decide between 2D and 3D in the next turn.

## Actions
- No files changed.
- Replied with a structured answer covering:
  - **Why 3D in the browser is hard:** Three.js is a renderer not an engine; asset pipeline (rigged models, animations, PBR textures, modular room geometry) is expensive; first-person specific pitfalls (camera bob, pointer-lock quirks, near-plane clipping); procedural 3D dungeon generation is harder than 2D tilemap procgen; browser perf ceilings; debugging difficulty; small-team realities.
  - **What 2D buys you:** faster time to playable, cheaper assets, mobile-friendly, simpler collision, mature tooling (Tiled, Aseprite, Phaser), easier procgen, smaller team.
  - **Tradeoffs against 2D for this PDF specifically:** loses first-person claustrophobia, loses Sweeper fog-of-war tension, less visceral ranged combat, less environmental texture.
  - **Recommendation:** for a real small-team product, 2D with Phaser. For a passion/learning project, 3D with Three.js. Middle path: 2.5D / Doom-style billboarded sprites in a 3D grid.

## Open follow-ups
- Waiting on Ryan's choice between 2D, 3D, or 2.5D.
