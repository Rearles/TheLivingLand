# Turn 011 — §5 deep-dive (Hybrid OOP + ECS architecture)

**Date:** 2026-05-22
**User intent:** Begin the Option-D deep-dive of the 3D engine design doc, section by section. Start with §5 (Hybrid OOP + ECS architecture) and apply Claude's recommended fixes.

## Issues surfaced in review

Five issues found in §5 + the supporting §6.5:

| # | Issue                                                                                                   | Stakes   |
| - | ------------------------------------------------------------------------------------------------------- | -------- |
| A | Physics → render sync missing; ragdolls would render at their spawn pose, not their simulated position. | critical |
| B | §6.5 falsely claimed Rapier auto-substeps; Rapier's `step()` integrates only by `world.timestep`.        | critical |
| C | Camera ownership undefined; tick referenced `this.camera` with no clear setter.                          | medium   |
| D | Input flow not in tick(); the host-latch model wasn't documented.                                       | medium   |
| E | Services were a footnote in §5.4's table, not a first-class category alongside actors and ECS.          | low      |

Ryan green-lit applying all five (and deferring lower-stakes ragdoll-ownership and audio-loop-ownership questions to §10).

## Decisions

- **Dynamic-body sync:** `GameEngine` owns a `dynamicBodies: Map<number, { body, mesh }>` registry. Step 5 of `tick()` walks it and copies Rapier transforms → THREE meshes. New public methods `registerDynamicBody(body, mesh)` and `unregisterDynamicBody(handle)`.
- **Fixed-timestep accumulator:** `tick()` step 4 accumulates the variable render `dt`, then steps Rapier at `T.PHYSICS_FIXED_DT` (1/60 s) up to `T.PHYSICS_MAX_STEPS_PER_FRAME` (5) per render frame. Excess accumulator is dropped (no spiral of death).
- **Camera ownership:** `PlayerController.onAttach()` sets `engine.activeCamera = this.camera`. The renderer reads from `engine.activeCamera`.
- **Input model:** push-from-host (Angular calls `engine.setInputs()` / `engine.applyMouseDelta()` per frame; the engine never polls). `tick()` step 1 is just a documenting comment.
- **Services as first-class category:** §5.4 now opens with an intro paragraph naming three categories (Actors, ECS components, Services). Decision table updated. Long-lived infrastructure (PhysicsWorld, AudioService, NavMeshService, Scheduler, AssetLoader) is a first-class category, created in `init()` and disposed in `dispose()`.

## Actions

Edits to `docs/3d-engine-design.md`:

1. **§5.1** — rewrote the `GameEngine` code block (~70 lines). Added the dynamic-body registry, `activeCamera` field, fixed-timestep accumulator with substep cap, sync step, and the two new public methods.
2. **§5.4** — promoted services from a table row to a first-class category. Added an intro paragraph naming the three categories with one bullet each. Updated the decision-table row for "long-lived infrastructure".
3. **§6.5** — full rewrite. Renamed to "Time — two clocks." Explained the render clock (variable, clamped) and the physics clock (fixed, accumulator) with consequence notes. Killed the wrong claim about Rapier auto-substepping.
4. **§9.2** — added `engine.activeCamera = this.camera` to `PlayerController.onAttach()` so §5's contract is honored.
5. **§19** — added a new "Physics timestep" section to `tuning.ts` with `PHYSICS_FIXED_DT = 1/60` and `PHYSICS_MAX_STEPS_PER_FRAME = 5`, with the derived "max ~83 ms of sim per render frame" annotation.

Also widened `.claude/settings.local.json` to auto-approve Write and Edit anywhere under `context/`. The original narrow rule (`Write(context/turn-*.md)`) only worked because the settings watcher had already picked up the file once; the wider rule required Ryan to `/hooks`-reload before it took effect (the watcher only watches directories that existed at session start, so a freshly-created `.claude/` is invisible to it until reload).

## Deferred to §10 review

- **Ragdoll-bone ownership** — actors (one Ragdoll actor per dead enemy holding a list of bones) vs ECS entities (one `RagdollBone` component per bone). The §5 dynamic-body registry doesn't force either; both work.
- **Audio-loop ownership** — for Sweeper footsteps, who holds the `SpatialEmitter` handle: the actor (started in §12.1) or the ECS world (where §5.3's `EcsEntity` puts `audio`)?

## Open follow-ups

- Continue the deep-dive with §10 (combat + ragdolls).
- Eventually open the PR from `story/3d-engine-design-doc` → `develop` once all five sections are reviewed.
