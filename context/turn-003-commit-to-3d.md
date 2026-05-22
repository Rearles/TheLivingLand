# Turn 003 — Commit to 3D + scope/style decisions

**Date:** 2026-05-21
**User intent:** Ryan declared the project a passion + learning project explicitly testing how far Claude Code can build a game from a design document. Chose full 3D with Claude Code handling all the 3D work given Ryan's detailed design notes.

## Clarifying questions asked
Three questions covering: asset source, scope staging, and visual style.

## Decisions (from Ryan's answers)
- **Asset source:** Free CC0 asset packs (Kenney, Quaternius, Sketchfab) plus procedural primitive geometry where no pack asset fits. No bespoke Blender modeling by Ryan — Claude Code handles all 3D wiring.
- **Scope staging:** Vertical-slice MVP first. The MVP is: register/login → shop with one weapon (Pipe) → one zone (Dungeon) → one room template (Locker Room) → 2 enemy types (Pipe Zombie + Mini Zombie) → Sweeper release → die-or-escape → summary. Subsequent slices expand outward.
- **Visual style:** Pixel-textured 3D — 3D geometry with pixel-art textures (nearest-filtered, low-res). Optional low-res render target upscaled with nearest-neighbor for chunky retro feel.

## Actions
- Updated `docs/frontend-design.md`:
  - §1 Overview: removed 2D hedging; committed to pixel-textured first-person 3D; added pointer to the forthcoming 3d-engine-design.md.
  - §1.1 (new): MVP scope section enumerating the vertical slice.
  - §2 Tech Stack table: added justification for Three.js over Babylon.js; added Rapier physics row.

## Open follow-ups
- Several edits started but not yet completed at the end of this turn:
  - §9 Assets section still referred to "PBR sets" (mismatch with pixel-textured commitment).
  - §14 Open Questions still listed the now-resolved 2D vs 3D question.
- New doc to create next: `docs/3d-engine-design.md` — deep technical doc covering coordinate system, scene graph, procgen algorithm, modular room kit pieces, lighting strategy for procgen scenes, asset pipeline for pixel-textured 3D, first-person controller, combat hit detection, enemy AI state machines, animation, audio architecture, performance budgets, and combat tuning numbers.
