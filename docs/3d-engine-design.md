# The Living Land — 3D Engine & Asset Pipeline Design

**Status:** Draft v1
**Owner:** Bolder Games, LLC
**Companion docs:** [api-design.md](./api-design.md), [frontend-design.md](./frontend-design.md)
**Stack:** Three.js r160+, Rapier 3D (WASM), recast-navigation-js, miniplex (ECS), Howler.js, alea (PRNG)

---

## 1. Overview

This document specifies the 3D game engine that powers the in-dungeon gameplay of *The Living Land*. The engine is a **standalone npm package** (`@rearles/livingland-engine`) consumed by the Angular client as a dependency. It is pure TypeScript, has no Angular imports, and is independently testable.

Where the [frontend-design.md](./frontend-design.md) describes Angular's shell (auth, routing, shop, HUD, summary), this document describes the engine that runs inside the `<canvas>` element when the player enters the dungeon: rendering, physics, AI, procgen, audio, and the contract between the engine and the Angular shell.

The reader should already have skimmed the frontend doc and have access to the original design PDF.

### 1.1 MVP scope

This document targets the **vertical-slice MVP** for the dungeon experience. Anything not in this list is out of scope for v0.1 and will be specced in a follow-on doc.

- **One room template:** Locker Room.
- **Procedural connectivity:** branching tree, 1–4 doors per room, 5–8 rooms per run.
- **One weapon:** Pipe (melee).
- **Two standard enemies:** Pipe Zombie, Mini Zombie.
- **One boss enemy:** Sweeper Zombie (invincible, hunts the player).
- **Player movement:** walk, run (stamina-limited), jump, crouch, mouse-look.
- **Combat:** melee swing with hit detection, full ragdoll death on enemies, damage feedback (player vignette + screen shake, enemies flash + knockback + particle puff).
- **Pathfinding:** full navmesh, baked from the generated dungeon geometry.
- **Audio:** 3D spatial sfx + non-spatial music, music ducks under sfx.
- **Run lifecycle:** start → play → die or escape → emit terminal event for the Angular shell to call the API.

### 1.2 Quality bar

The MVP target is **"playable for fun"** — a 5–10 minute run feels like a real game. Tension when the Sweeper drops. Satisfaction connecting a pipe swing. Sound tuned, HUD readable, no game-breaking jank. Not showcase polish, but you'd happily play it yourself.

### 1.3 Non-goals

- Multiplayer, voice chat, friends, leaderboards beyond the existing API surface.
- Server-side combat authority. Combat is resolved in the engine; the API only sees aggregate run results.
- Replays. We choose **loose determinism** in §6.6 — the layout is reproducible from a seed but per-frame randomness is not.
- Mobile / touch. Desktop keyboard + mouse only.
- VR / AR.

---

## 2. Repository & Package Boundaries

The engine lives in its **own GitHub repository**, separate from the Angular client.

```
github.com/Rearles/TheLivingLand          ← this repo (Angular client + design docs + API)
github.com/Rearles/livingland-engine      ← engine repo (described in this doc)
```

The engine is published to npm as `@rearles/livingland-engine`. The client depends on it like any other npm package:

```jsonc
// packages/client/package.json (the Angular client)
{
  "dependencies": {
    "@rearles/livingland-engine": "^0.1.0",
    "@angular/core": "^17.0.0",
    ...
  }
}
```

Local development uses `npm link` (or pnpm workspaces if both repos are checked out side-by-side) so engine changes are seen by the client without a publish round-trip.

### 2.1 Why a separate package?

- **Independent testability** — the engine runs under Vitest in a pure Node + jsdom context, no Angular Test Bed.
- **Independent versioning** — engine semver tracks engine API, not Angular feature work.
- **Reusability** — a future admin/wiki could pull in a small subset of the engine (e.g. the room renderer) for previews.
- **Clean dependency direction** — the engine has zero knowledge of Angular, the API, or the player's session. It just runs the game.

### 2.2 Engine repo layout

```
livingland-engine/
├── package.json                # name: @rearles/livingland-engine
├── tsconfig.json
├── vite.config.ts              # library mode build
├── vitest.config.ts
├── src/
│   ├── index.ts                # public re-exports
│   ├── engine/
│   │   ├── GameEngine.ts       # top-level lifecycle (init/loadRun/tick/dispose)
│   │   ├── EventBus.ts         # typed event emitter for the host (Angular)
│   │   ├── events.ts           # GameEvent type union
│   │   └── time.ts             # clock + fixed/variable timestep helpers
│   ├── rendering/
│   │   ├── Renderer.ts         # Three.js renderer + low-res target + upscale
│   │   ├── materials.ts        # pixel-texture material factory
│   │   └── postfx.ts           # subtle vignette only (MVP)
│   ├── physics/
│   │   ├── PhysicsWorld.ts     # Rapier wrapper
│   │   └── colliders.ts        # capsule, box, mesh collider helpers
│   ├── ecs/
│   │   ├── world.ts            # miniplex world singleton
│   │   ├── components.ts       # Particle, AudioSource, Projectile (future)
│   │   └── systems/            # update systems
│   ├── actors/                 # OOP actors
│   │   ├── Entity.ts           # base
│   │   ├── Player.ts
│   │   ├── PlayerController.ts # input + movement
│   │   ├── enemies/
│   │   │   ├── Enemy.ts        # state machine base
│   │   │   ├── PipeZombie.ts
│   │   │   ├── MiniZombie.ts
│   │   │   └── SweeperZombie.ts
│   │   └── weapons/
│   │       ├── Weapon.ts
│   │       └── Pipe.ts
│   ├── ai/
│   │   ├── NavMeshService.ts   # recast-navigation wrapper
│   │   └── StateMachine.ts     # generic FSM helper
│   ├── procgen/
│   │   ├── DungeonGenerator.ts # branching tree
│   │   ├── RoomBuilder.ts      # room kit assembler
│   │   ├── LockerRoom.ts       # MVP room template
│   │   └── prng.ts             # alea wrapper
│   ├── audio/
│   │   ├── AudioService.ts     # AudioContext + listener
│   │   └── SpatialEmitter.ts   # PannerNode wrapper
│   ├── assets/
│   │   ├── AssetLoader.ts      # GLTFLoader + DRACOLoader
│   │   └── manifest.ts         # the assets the engine expects to find
│   └── tuning.ts               # all magic numbers + formulas in one place
├── tests/                      # vitest
└── README.md
```

### 2.3 Public surface

Only what's exported from `src/index.ts` is part of the public API. The minimal v0.1 surface is small on purpose:

```typescript
// @rearles/livingland-engine — public API
export { GameEngine } from './engine/GameEngine'
export type { GameEngineOptions, GameEvent } from './engine/events'
export type { RunDescriptor, RunOutcome } from './engine/types'
export { tuning } from './tuning'           // for tests + admin tools
```

Everything else (actors, ECS components, internal systems) is package-internal. The Angular client never imports a `Player` class directly; it only calls `engine.loadRun(...)` and listens to events.

### 2.4 Versioning

- Semver. Engine version is **independent** of the client version.
- `0.x.y` until the MVP ships; breaking changes allowed on minor bumps.
- After MVP: `1.0.0`. Then breaking changes go on major bumps.
- CI publishes to npm on `git tag v*` pushes (GitHub Actions; out of scope for this doc).

---

## 3. Library Stack & Versions

| Library                        | Version | Role                                                           |
| ------------------------------ | ------- | -------------------------------------------------------------- |
| `three`                        | ^0.160  | Renderer, scene graph, glTF loading                            |
| `@dimforge/rapier3d-compat`    | ^0.13   | Physics (WASM bundled, no async init pain in browsers)         |
| `@recast-navigation/three`     | ^0.30   | Navmesh baking + queries; the `three` adapter for recast       |
| `miniplex`                     | ^2.0    | ECS for swarms (particles, future projectiles, audio sources)  |
| `howler`                       | ^2.2    | Music playback + sfx (sprite-friendly); spatial audio uses raw Web Audio (§16) |
| `alea`                         | ^1.0    | Seedable PRNG for procgen                                      |
| `vite`                         | ^5.0    | Library-mode build                                             |
| `vitest`                       | ^1.0    | Unit testing                                                   |
| `typescript`                   | ^5.3    | `strict: true`                                                 |

### 3.1 Why these specifically

- **Three.js over Babylon.js** — bigger ecosystem and example corpus, lightweight (~580 KB minified core), explicit control of the render loop. Babylon is more "engine-like" but locks you into its abstractions.
- **Rapier over Cannon.js / Ammo.js** — Cannon is in maintenance mode, Ammo is a port of an old C++ engine with a clunky API. Rapier is modern (Rust + WASM), actively developed, and the `compat` build sidesteps WASM async-init headaches.
- **recast-navigation-js over three-pathfinding** — three-pathfinding is grid-based and limited; recast is the same library that ships in Unity/Unreal/etc. The `three` adapter handles glTF → navmesh conversion cleanly.
- **miniplex over bitecs** — bitecs is faster but uses raw memory and is awkward in TypeScript. miniplex is the most ergonomic TS-first ECS. The hybrid architecture (§5) means we only use ECS for swarms, so absolute perf isn't critical.
- **Howler for music, Web Audio for spatial sfx** — Howler is the cleanest API for music tracks and 2D sfx; for the spatial sfx that drive horror tension (footsteps, Sweeper proximity) we use Web Audio `PannerNode` directly since Howler's spatial support is shallower than what we need.
- **alea over Math.random** — Math.random isn't seedable. alea is a tiny (~1 KB) seedable PRNG with good statistical properties; PRNG state is one number.

---

## 4. Coordinate System & Conventions

| Property           | Value                              | Notes                                              |
| ------------------ | ---------------------------------- | -------------------------------------------------- |
| Handedness         | Right-handed                       | Three.js default                                   |
| Up axis            | +Y                                 | Three.js default                                   |
| Forward            | -Z (camera looks down -Z)          | Three.js default                                   |
| Units              | 1 unit = 1 meter                   |                                                    |
| Time               | Seconds (floats), `dt` in seconds  | Engine never deals with ms internally              |
| Angles             | Radians                            |                                                    |
| World origin       | Center of the run's starting room  | Procgen lays rooms relative to origin              |

### 4.1 Naming conventions

- **Files**: `PascalCase.ts` for classes, `camelCase.ts` for modules of functions.
- **Components (ECS)**: PascalCase nouns: `Particle`, `AudioSource`.
- **Systems (ECS)**: camelCase verbs: `particleSystem`, `audioFadeSystem`.
- **Actors (OOP)**: PascalCase nouns: `Player`, `PipeZombie`.
- **Events**: PascalCase past tense: `EnemyKilled`, `RoomEntered`, `PlayerDied`.
- **Tuning constants**: SCREAMING_SNAKE in `tuning.ts`, e.g. `PIPE_SWING_DAMAGE`.

### 4.2 Hot paths

The render and physics tick are the hot loops. In hot-path code:

- **No allocations per frame.** Reuse `Vector3`, `Quaternion`, `Matrix4` instances. Each actor owns its scratch vectors.
- **No `Math.random()`** in deterministic code paths (only allowed in particle / non-gameplay code per §6.6).
- **No string-keyed property access** in tight loops; use numeric indices into typed arrays where possible.
- **Avoid Three.js `.copy()` chains** that allocate on assignment; prefer in-place math.

---

## 5. Architecture: Hybrid OOP + ECS

The engine uses **OOP for individual actors** (Player, each Enemy, each Weapon) and **ECS for swarms** (particles, audio source pool, future projectiles). This balance lets the gameplay code read like the design doc (one class per design entity) while keeping per-frame work cheap for high-cardinality stuff.

### 5.1 The two paradigms side by side

```typescript
// engine/GameEngine.ts — the seam between OOP and ECS
import RAPIER from '@dimforge/rapier3d-compat'
import { Object3D, PerspectiveCamera } from 'three'
import { World } from 'miniplex'
import { particleSystem, audioFadeSystem } from '../ecs/systems'
import type { Entity } from '../actors/Entity'
import { tuning as T } from '../tuning'

export class GameEngine {
  // OOP path: individual, behavior-rich actors.
  private actors: Set<Entity> = new Set()

  // ECS path: swarms, particles, transient effects.
  private ecsWorld: World<EcsEntity> = new World()

  // Dynamic-body registry — bodies whose THREE mesh transforms must be copied
  // from Rapier each frame so they render at their simulated position
  // (e.g. ragdoll bones — see §10.4 — and any future physics-driven projectiles).
  private dynamicBodies: Map<number, { body: RAPIER.RigidBody; mesh: Object3D }> = new Map()

  // The camera the renderer draws with. Registered by PlayerController.onAttach()
  // — the engine doesn't construct its own camera.
  activeCamera!: PerspectiveCamera

  // Fixed-timestep accumulator. Rapier integrates at T.PHYSICS_FIXED_DT (1/60 s)
  // regardless of the render frame rate, so simulation is frame-rate-independent
  // (critical for stable ragdolls — see §6.5).
  private physicsAccumulator = 0

  tick(dt: number): void {
    // 1) Input was already latched by the host (Angular DungeonPage) via
    //    setInputs() / applyMouseDelta() before this tick. The player consumes
    //    those fields during its own update() in step 2.

    // 2) Actors update their behavior + position. Kinematic bodies (the Player)
    //    stage their next translation here — the physics step in (4) applies it.
    for (const actor of this.actors) actor.update(dt)

    // 3) ECS systems sweep the swarm components.
    particleSystem(this.ecsWorld, dt)
    audioFadeSystem(this.ecsWorld, dt)

    // 4) Step Rapier at a fixed rate using a classic accumulator. Rapier's
    //    world.step() integrates by world.timestep (= T.PHYSICS_FIXED_DT),
    //    NOT by the variable dt we pass at the render layer. A per-frame cap
    //    of T.PHYSICS_MAX_STEPS_PER_FRAME prevents a "spiral of death" if
    //    the tab is restored from backgrounding.
    this.physicsAccumulator += dt
    let steps = 0
    while (
      this.physicsAccumulator >= T.PHYSICS_FIXED_DT &&
      steps < T.PHYSICS_MAX_STEPS_PER_FRAME
    ) {
      this.physics.step()
      this.physicsAccumulator -= T.PHYSICS_FIXED_DT
      steps++
    }
    if (this.physicsAccumulator >= T.PHYSICS_FIXED_DT) {
      // Hit the substep cap. Drop the rest so we don't accumulate forever.
      this.physicsAccumulator = 0
    }

    // 5) Sync dynamic-body transforms (Rapier → THREE) so ragdolls render where
    //    physics put them, not where they were last spawned.
    for (const { body, mesh } of this.dynamicBodies.values()) {
      const t = body.translation()
      const q = body.rotation()
      mesh.position.set(t.x, t.y, t.z)
      mesh.quaternion.set(q.x, q.y, q.z, q.w)
    }

    // 6) Render with the camera the player registered.
    this.renderer.render(this.scene, this.activeCamera)
  }

  spawnActor<E extends Entity>(actor: E): E {
    this.actors.add(actor)
    actor.onAttach(this)
    return actor
  }

  despawnActor(actor: Entity): void {
    actor.onDetach(this)
    this.actors.delete(actor)
  }

  spawnParticle(opts: ParticleSpawnOptions): void {
    this.ecsWorld.add({
      position: opts.position.clone(),
      velocity: opts.velocity.clone(),
      ttl: opts.ttl,
      sprite: opts.sprite,
    })
  }

  // Register a dynamic body whose THREE mesh should follow the physics
  // simulation each frame. Called by §10.4's spawnRagdoll() once per bone.
  // Returns the body handle so the caller can unregister on cleanup.
  registerDynamicBody(body: RAPIER.RigidBody, mesh: Object3D): number {
    this.dynamicBodies.set(body.handle, { body, mesh })
    return body.handle
  }

  unregisterDynamicBody(bodyHandle: number): void {
    this.dynamicBodies.delete(bodyHandle)
  }
}
```

### 5.2 The `Entity` base (OOP path)

```typescript
// actors/Entity.ts
import type { Object3D } from 'three'
import type { GameEngine } from '../engine/GameEngine'

export abstract class Entity {
  abstract readonly kind: string             // 'player' | 'pipe-zombie' | ...
  position: Vector3 = new Vector3()
  protected mesh?: Object3D                  // optional visual representation
  protected engine?: GameEngine              // back-reference set in onAttach

  onAttach(engine: GameEngine): void {
    this.engine = engine
    if (this.mesh) engine.scene.add(this.mesh)
  }

  onDetach(engine: GameEngine): void {
    if (this.mesh) engine.scene.remove(this.mesh)
    this.engine = undefined
  }

  abstract update(dt: number): void
}
```

### 5.3 The ECS world

```typescript
// ecs/world.ts
import { World } from 'miniplex'
import type { Vector3, Texture } from 'three'

export type EcsEntity = {
  position?: Vector3
  velocity?: Vector3
  ttl?: number                  // seconds remaining until despawn
  sprite?: Texture              // particle billboard texture
  audio?: SpatialEmitter        // spatial audio handle (see §16)
}

export const createEcsWorld = () => new World<EcsEntity>()
```

### 5.4 Three categories: actors, ECS components, services

Engine code lands in one of three categories. Each has a different lifecycle and a different home in the file tree.

- **Actors (OOP)** — `actors/`. One instance per design entity (Player, each Enemy, each Weapon, each Locker, each ragdoll). Owns its own state, has rich behavior, updates itself in the actor loop in §5.1 step 2. Best when instance count is low and each behaves differently.
- **ECS components** — `ecs/`. Tiny per-row data in a `miniplex` world; mutated by systems that sweep over all entities matching a component query. Best for high-cardinality, uniform-per-tick effects (particles, audio source pool, future projectiles).
- **Services (singletons)** — created in `GameEngine.init()`, disposed in `GameEngine.dispose()`. Long-lived infrastructure: `PhysicsWorld` (Rapier), `AudioService` (Web Audio), `NavMeshService` (recast), `Scheduler` (one-shot delayed callbacks), `AssetLoader` (glTF + textures). Actors and ECS systems call into them; they own no actors and no components themselves.

The decision when adding something new:

| If the thing has...                                          | It's...                |
| ------------------------------------------------------------ | ---------------------- |
| Distinct behavior per instance, AI, complex state            | An **actor (OOP)**     |
| High instance counts, uniform per-tick math, throwaway       | An **ECS component**   |
| Both (e.g. an enemy emits particles)                         | Actor owns the spawn; particles live in ECS |
| Long-lived infrastructure (physics, audio, navmesh, assets)  | A **service** on `GameEngine` |

---

## 6. Engine Lifecycle & Event Bus

### 6.1 Lifecycle states

```
[ Uninitialized ] ──init(canvas)──► [ Ready ] ──loadRun(descriptor)──► [ Running ]
                                                                            │
                                          pause() ◄──────────────────► resume()
                                                                            │
                                                                       endRun(outcome)
                                                                            │
                                                                            ▼
                                                                      [ Finished ]
                                                                            │
                                                                       dispose()
                                                                            │
                                                                            ▼
                                                                      [ Disposed ]
```

### 6.2 Public engine API

```typescript
// engine/GameEngine.ts (public surface)
export interface GameEngineOptions {
  canvas: HTMLCanvasElement
  pixelTarget?: { width: number; height: number }   // default 640x360
  showFirstPersonHands?: boolean                    // default true
  mouseSensitivity?: number                         // radians per pixel
  audioListener?: AudioListener                     // injected for testing
}

export interface RunDescriptor {
  runId: string
  seed: string                  // server-provided; feeds procgen PRNG
  player: PlayerSnapshot        // hp, level, equipped weapon, inventory
  content: ContentBundle        // enemy stats, room templates, items
}

export class GameEngine {
  constructor(opts: GameEngineOptions)
  async init(): Promise<void>                       // load WASM, build renderer
  async loadRun(run: RunDescriptor): Promise<void>  // generate dungeon, spawn player
  pause(): void
  resume(): void
  endRun(outcome: RunOutcome): void                 // emits RunFinished
  dispose(): void                                   // releases GPU/audio resources

  readonly events: EventBus<GameEvent>              // typed event stream
}
```

### 6.3 The event bus

The engine emits events for everything Angular needs to react to. The bus is a **typed pub/sub** with no RxJS dependency (Angular wraps it in a signal/observable on its side).

```typescript
// engine/EventBus.ts
export class EventBus<T extends { type: string }> {
  private listeners = new Map<string, Set<(e: T) => void>>()

  on<K extends T['type']>(type: K, fn: (e: Extract<T, { type: K }>) => void): () => void {
    const set = this.listeners.get(type) ?? new Set()
    set.add(fn as (e: T) => void)
    this.listeners.set(type, set)
    return () => set.delete(fn as (e: T) => void)
  }

  emit(event: T): void {
    this.listeners.get(event.type)?.forEach(fn => fn(event))
    this.listeners.get('*')?.forEach(fn => fn(event))   // wildcard for logging
  }
}
```

### 6.4 The `GameEvent` union (MVP)

```typescript
// engine/events.ts
export type GameEvent =
  | { type: 'RunStarted';     runId: string; seed: string }
  | { type: 'RoomEntered';    roomIndex: number; roomKind: string }
  | { type: 'EnemyKilled';    enemyKind: EnemyKind; position: Vec3; pointsDelta: number; goldDelta: number; multiplier: number }
  | { type: 'EnemyDamaged';   enemyId: number; remainingHp: number }
  | { type: 'PlayerHit';      damage: number; remainingHp: number; sourceKind: string }
  | { type: 'PlayerHealed';   amount: number; remainingHp: number }
  | { type: 'ItemPickedUp';   itemSlug: string; quantity: number }
  | { type: 'AmmoChanged';    weaponSlug: string; remaining: number }
  | { type: 'WeaponBroke';    weaponSlug: string }
  | { type: 'ClockFound';     remainingSeconds: number }
  | { type: 'SweeperReleased' }
  | { type: 'StaminaChanged'; current: number; max: number }
  | { type: 'PlayerDied';     killer: string }
  | { type: 'RunCompleted';   outcome: 'died' | 'escaped' | 'beat-mega'; totals: RunTotals }
  | { type: 'LowHealth' }
  | { type: 'EngineError';    error: { message: string; stack?: string } }

export interface RunTotals {
  zombiesKilled: number
  goldEarned: number
  xpEarned: number
  deepestRoom: number
  topMultiplier: number
  durationSeconds: number
}
```

### 6.5 Time — two clocks

Two clocks run inside the engine. They serve different masters.

**Render clock** (variable `dt`). The render loop is driven by `requestAnimationFrame`, so frame intervals vary with the host: 16.7 ms on a 60 Hz display, 6.9 ms on a 144 Hz display, and arbitrarily long after a tab is restored from backgrounding. We clamp `dt` to bound the worst case — if a tab was hidden for 30 seconds, we don't want the engine trying to "catch up" 30 seconds of simulation in one frame.

```typescript
// engine/time.ts
const MAX_DT = 1 / 30          // never advance more than ~33 ms in one render tick

export class Clock {
  private last = performance.now() / 1000
  tick(): number {
    const now = performance.now() / 1000
    const raw = now - this.last
    this.last = now
    return Math.min(raw, MAX_DT)
  }
}
```

**Physics clock** (fixed `dt`). Rapier's `world.step()` advances the world by its `integrationParameters.dt` property (set to `T.PHYSICS_FIXED_DT` = 1/60 s); it does NOT consume the render-side `dt`. There is no automatic substepping inside Rapier — if you call `step()` once per render frame at 144 Hz, you'd be running physics at 2.4× real time on that machine and 1× real time on a 60 Hz machine.

To stay frame-rate-independent, we use a classic **accumulator pattern** in `GameEngine.tick()` (§5.1 step 4): every render tick we add the variable `dt` to the accumulator, then step Rapier 1/60 s at a time until the accumulator drains. A per-frame cap of `T.PHYSICS_MAX_STEPS_PER_FRAME` (5 steps = 83 ms of simulation) prevents a "spiral of death" if the host stalls badly — if we hit the cap, we **drop the remainder** rather than try to catch up.

Consequences worth knowing:

- Physics behavior is **deterministic in step count** (good — ragdolls fall the same way regardless of frame rate).
- Visual representation may lag the simulation by up to one fixed step (~16 ms). At 60 Hz render this is invisible; at higher refresh it can produce mild judder. If we ever observe it in playtests, we'll add render-time **interpolation** between the two most recent physics states in the dynamic-body sync pass.
- Kinematic bodies (the Player) stage their next translation during the actor update (§5.1 step 2); Rapier applies it on the next physics step. With one render frame where physics happens to step 0 times (because the accumulator hasn't drained yet), the player's queued movement is held until the next step. In practice with a 60 Hz physics rate and ≤144 Hz render, this is below perceptual threshold.

### 6.6 Determinism policy

The engine uses **loose determinism**:

- **Seeded (uses the run's PRNG)**: dungeon layout, room contents (loot picks, enemy spawn positions, obstacle placements), navmesh seed.
- **Not seeded (uses `Math.random()` is fine)**: particle jitter, audio pitch variation, hit-flash duration jitter, screen-shake direction, ragdoll initial impulse jitter.

The rule is: **anything that affects gameplay outcomes goes through the seeded PRNG; anything that only affects look/feel can use `Math.random`.** This gives us reproducible level layouts (good for bug reports) without forcing us to seed every cosmetic flourish.

```typescript
// procgen/prng.ts
import alea from 'alea'

export class RunPrng {
  private rng: () => number
  constructor(seed: string) {
    this.rng = alea(seed)
  }
  next(): number { return this.rng() }
  range(min: number, max: number): number { return min + this.rng() * (max - min) }
  int(min: number, maxInclusive: number): number {
    return Math.floor(min + this.rng() * (maxInclusive - min + 1))
  }
  pick<T>(arr: readonly T[]): T { return arr[this.int(0, arr.length - 1)] }
}
```

---

## 7. Rendering Pipeline

### 7.1 The pixel-textured 3D recipe

We render the 3D scene at **640×360** into an offscreen render target, then upscale it to the canvas with **nearest-neighbor filtering**. This gives a chunky, deliberate pixel feel without sacrificing modern lighting.

```
       60–144 Hz                    1× CSS upscale
[ Scene + camera ] ──renderTarget──► [ 640×360 RT ] ──fullscreen quad──► [ canvas (any size) ]
                       (3D pass)         (NearestFilter on RT)            ( image-rendering: pixelated )
```

### 7.2 Renderer setup

```typescript
// rendering/Renderer.ts
import {
  WebGLRenderer, WebGLRenderTarget, NearestFilter, OrthographicCamera,
  Scene, PlaneGeometry, ShaderMaterial, Mesh, LinearSRGBColorSpace,
} from 'three'

const PIXEL_W = 640
const PIXEL_H = 360

export class Renderer {
  private gl: WebGLRenderer
  private rt: WebGLRenderTarget
  private upscaleScene: Scene
  private upscaleCam: OrthographicCamera

  constructor(canvas: HTMLCanvasElement) {
    this.gl = new WebGLRenderer({ canvas, antialias: false, alpha: false })
    this.gl.setPixelRatio(1)              // we control sampling ourselves
    this.gl.outputColorSpace = LinearSRGBColorSpace

    this.rt = new WebGLRenderTarget(PIXEL_W, PIXEL_H, {
      minFilter: NearestFilter,
      magFilter: NearestFilter,
      generateMipmaps: false,
    })

    // Fullscreen quad that samples the RT with a nearest-filtered pass-through shader.
    this.upscaleScene = new Scene()
    this.upscaleCam = new OrthographicCamera(-1, 1, 1, -1, 0, 1)
    const quad = new Mesh(
      new PlaneGeometry(2, 2),
      new ShaderMaterial({
        uniforms: { tDiffuse: { value: this.rt.texture } },
        vertexShader: PASS_VS,
        fragmentShader: PASS_FS,
        depthTest: false,
        depthWrite: false,
      }),
    )
    this.upscaleScene.add(quad)
  }

  resize(width: number, height: number): void {
    // The RT stays at 640×360; only the canvas backing buffer changes.
    this.gl.setSize(width, height, false)
  }

  render(scene: Scene, camera: Camera): void {
    this.gl.setRenderTarget(this.rt)
    this.gl.render(scene, camera)
    this.gl.setRenderTarget(null)
    this.gl.render(this.upscaleScene, this.upscaleCam)
  }
}

const PASS_VS = `
  varying vec2 vUv;
  void main() { vUv = uv; gl_Position = vec4(position, 1.0); }
`
const PASS_FS = `
  uniform sampler2D tDiffuse;
  varying vec2 vUv;
  void main() {
    vec3 col = texture2D(tDiffuse, vUv).rgb;
    // subtle vignette (MVP post-fx)
    float v = smoothstep(0.85, 0.45, length(vUv - 0.5));
    gl_FragColor = vec4(col * mix(0.85, 1.0, v), 1.0);
  }
`
```

### 7.3 Material factory

Every material in the engine goes through a factory that pins nearest-neighbor filtering and disables mipmaps so textures don't smooth with distance.

```typescript
// rendering/materials.ts
import { MeshStandardMaterial, NearestFilter, RepeatWrapping, Texture } from 'three'

export function pixelStandard(map: Texture, opts?: { metalness?: number; roughness?: number }): MeshStandardMaterial {
  map.magFilter = NearestFilter
  map.minFilter = NearestFilter
  map.generateMipmaps = false
  map.wrapS = RepeatWrapping
  map.wrapT = RepeatWrapping
  return new MeshStandardMaterial({
    map,
    metalness: opts?.metalness ?? 0,
    roughness: opts?.roughness ?? 1,
  })
}
```

### 7.4 Lighting

Modern lighting model, kept cheap:

- **One hemisphere light** for sky/ground ambient — provides base illumination.
- **One directional light** with no shadow casting (we'll add cascaded shadow maps in v0.2 if needed) — provides directional shading.
- **Per-room point lights** for atmosphere — 2–4 per room (locker overhead bulbs, flickering torches).
- **No real-time shadows in MVP.** Dynamic shadows are expensive in a procgen scene; we'll fake floor contact with darkened decal sprites underneath actors.

```typescript
// rendering/lighting.ts
import { HemisphereLight, DirectionalLight, PointLight, Scene } from 'three'

export function buildBaseLighting(scene: Scene): void {
  scene.add(new HemisphereLight(0x9090a0, 0x202020, 0.4))
  const sun = new DirectionalLight(0xffffff, 0.5)
  sun.position.set(2, 5, 3)
  scene.add(sun)
}

export function addRoomLight(scene: Scene, position: Vec3, color = 0xffe0a0, intensity = 0.8): PointLight {
  const light = new PointLight(color, intensity, 8, 2)
  light.position.copy(position)
  scene.add(light)
  return light
}
```

### 7.5 Post-fx

MVP runs only the subtle vignette in the upscale shader (§7.2). Bigger post-fx (chromatic aberration, scanlines, CRT curvature) are deferred — they're easy to add later and we don't want to commit to a "look" before playtesting.

---

## 8. Asset Pipeline

### 8.1 Named primary packs

| Asset domain               | Primary pack                                                    | License | Notes                                                |
| -------------------------- | --------------------------------------------------------------- | ------- | ---------------------------------------------------- |
| Modular dungeon kit pieces | **Quaternius — "Ultimate Modular Dungeon"**                     | CC0     | Floors, walls, doors, props. Retexture to pixel.     |
| Lockers + interior props   | **Kenney — "Mini Dungeon Kit" + "Furniture Kit"**               | CC0     | Locker meshes, beds, tables. May need light retopo.  |
| Zombies (Pipe + Mini)      | **Quaternius — "Modular Characters" + "Zombies"**               | CC0     | Mid-poly humanoids, basic anim rigs. Rescale to 1u=1m. |
| Sweeper boss               | Composite: enlarged Quaternius zombie + bespoke palette         | CC0     | Recolor + scale 1.5× for menace.                     |
| First-person hands         | **Kenney — "FPS Pack"** OR itch.io pixel-arms                   | CC0     | Generic arms holding weapons.                        |
| Weapons (Pipe)             | **Kenney — "Weapon Pack"** OR primitive cylinder + texture      | CC0     | Pipe is a simple cylinder; can be procedural.        |
| Pixel textures             | Authored in **Aseprite** OR sourced from **itch.io CC0 pixel texture packs** | CC0 | 16–32 px palette-locked.                       |

### 8.2 Pack swap criteria

Any replacement pack must:

1. Be **CC0** or compatible permissive license.
2. Ship in **glTF/.glb** or be trivially convertible (Blender → glTF export).
3. Be **retextureable** — UV-mapped, not vertex-colored, so we can swap textures for the pixel-art look.
4. Be **at scale** (or rescalable to) 1 unit = 1 meter without manual rig fixes.
5. Have **a rest pose** and standard humanoid bone names if it's a character (Mixamo-compatible naming preferred so we can pull Mixamo anims).

### 8.3 Authoring conventions

- **Models**: `.glb` (glTF binary). Apply Draco compression at export.
- **Textures**: `.png`, indexed or low-color, 16/32/64 px on the long side. Stored in `assets/textures/<material>/`.
- **Animations**: bundled into the source `.glb` where possible. If anims must be separate (Mixamo), name them `<actor>.<anim>.glb`.
- **Atlases**: small static UI icons get atlased; in-world textures stay separate so palettes can be edited per-prop.

### 8.4 Asset manifest

The engine declares the assets it expects to find. This lets the client (or a CDN tool) prefetch and warn if any are missing.

```typescript
// assets/manifest.ts
export const ENGINE_ASSETS = {
  models: {
    locker:       'models/locker.glb',
    floor:        'models/dungeon-floor.glb',
    wall:         'models/dungeon-wall.glb',
    doorFrame:    'models/dungeon-door.glb',
    pipeZombie:   'models/pipe-zombie.glb',
    miniZombie:   'models/mini-zombie.glb',
    sweeper:      'models/sweeper.glb',
    pipe:         'models/pipe.glb',
    fpHands:      'models/fp-hands.glb',
  },
  textures: {
    floor:        'textures/floor.png',
    wall:         'textures/wall.png',
    locker:       'textures/locker.png',
    zombieSkin:   'textures/zombie-skin.png',
    zombieClothes:'textures/zombie-clothes.png',
  },
  audio: {
    music: {
      darksides:  'audio/music/darksides.mp3',
      relapse:    'audio/music/relapse.mp3',
    },
    sfx: {
      pipeSwing:  'audio/sfx/pipe-swing.ogg',
      pipeHit:    'audio/sfx/pipe-hit-flesh.ogg',
      zombieGroan:'audio/sfx/zombie-groan.ogg',
      sweeperRoar:'audio/sfx/sweeper-roar.ogg',
      sweeperSteps:'audio/sfx/sweeper-footsteps-loop.ogg',
      clockTick:  'audio/sfx/clock-tick.ogg',
      lockerOpen: 'audio/sfx/locker-open.ogg',
    },
  },
} as const
```

### 8.5 Loader

```typescript
// assets/AssetLoader.ts
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js'
import { TextureLoader, NearestFilter } from 'three'

export class AssetLoader {
  private gltf = new GLTFLoader()
  private tex = new TextureLoader()
  constructor(baseUrl: string) {
    const draco = new DRACOLoader()
    draco.setDecoderPath(`${baseUrl}/draco/`)
    this.gltf.setDRACOLoader(draco)
  }
  async loadModel(path: string) { return (await this.gltf.loadAsync(path)).scene }
  async loadTexture(path: string) {
    const t = await this.tex.loadAsync(path)
    t.magFilter = t.minFilter = NearestFilter
    t.generateMipmaps = false
    return t
  }
}
```

---

## 9. Player Controller

The player is an OOP actor that wraps a Rapier `KinematicCharacterController` (capsule collider) and a camera. Movement is **Half-Life-ish**: smooth acceleration, ground friction, no air control, no strafe-jumping.

### 9.1 The shape of the player

```
        camera (eye)
        ┌──┐
        │  │  eye height = 1.6 m
   FP arms (toggleable)
        │  │
        ├──┤
        │  │  capsule top
        │  │
        │  │  capsule total height standing = 1.7 m
        │  │  capsule total height crouched = 0.85 m
        │  │  radius = 0.35 m
        └──┘
```

### 9.2 PlayerController

```typescript
// actors/PlayerController.ts
import { Vector3, PerspectiveCamera } from 'three'
import RAPIER from '@dimforge/rapier3d-compat'
import { tuning as T } from '../tuning'
import { Entity } from './Entity'

export class PlayerController extends Entity {
  readonly kind = 'player'

  camera = new PerspectiveCamera(T.PLAYER_FOV_DEG, 16 / 9, 0.05, 200)
  velocity = new Vector3()
  isCrouched = false
  isGrounded = false
  stamina = T.STAMINA_MAX                 // seconds remaining

  private yaw = 0
  private pitch = 0
  private input = { fwd: 0, right: 0, jump: false, sprint: false, crouch: false }

  // Set up by onAttach
  private body!: RAPIER.RigidBody
  private collider!: RAPIER.Collider
  private cct!: RAPIER.KinematicCharacterController

  onAttach(engine: GameEngine) {
    super.onAttach(engine)
    const rapier = engine.physics.rapier
    const bodyDesc = rapier.RigidBodyDesc.kinematicPositionBased()
      .setTranslation(0, 1.0, 0)
    this.body = engine.physics.world.createRigidBody(bodyDesc)
    const colliderDesc = rapier.ColliderDesc.capsule(
      (T.PLAYER_HEIGHT_STANDING - 2 * T.PLAYER_RADIUS) / 2,
      T.PLAYER_RADIUS,
    )
    this.collider = engine.physics.world.createCollider(colliderDesc, this.body)
    this.cct = engine.physics.world.createCharacterController(0.05)
    this.cct.enableSnapToGround(0.2)
    this.cct.setMaxSlopeClimbAngle(50 * Math.PI / 180)

    // Register our camera as the engine's render camera (see §5.1).
    engine.activeCamera = this.camera
  }

  setInputs(input: typeof this.input) { this.input = input }

  applyMouseDelta(dx: number, dy: number, sensitivity: number) {
    this.yaw   -= dx * sensitivity
    this.pitch -= dy * sensitivity
    this.pitch = Math.max(-Math.PI / 2 + 0.01, Math.min(Math.PI / 2 - 0.01, this.pitch))
  }

  update(dt: number): void {
    // 1) Sprint stamina logic
    const wantsSprint = this.input.sprint && this.input.fwd > 0 && !this.isCrouched
    if (wantsSprint && this.stamina > 0) {
      this.stamina = Math.max(0, this.stamina - dt)
    } else if (!wantsSprint && this.stamina < T.STAMINA_MAX) {
      // regen after a delay would require a timer; for MVP we regen immediately when not sprinting
      this.stamina = Math.min(T.STAMINA_MAX, this.stamina + dt * T.STAMINA_REGEN_PER_S)
    }
    const sprinting = wantsSprint && this.stamina > 0

    // 2) Desired velocity from inputs (in player's local frame)
    const speed = this.isCrouched
      ? T.PLAYER_CROUCH_SPEED
      : sprinting ? T.PLAYER_RUN_SPEED : T.PLAYER_WALK_SPEED
    const dx = Math.sin(this.yaw) * this.input.fwd + Math.cos(this.yaw) * this.input.right
    const dz = Math.cos(this.yaw) * this.input.fwd - Math.sin(this.yaw) * this.input.right
    const wishVel = new Vector3(dx, 0, dz).normalize().multiplyScalar(speed)

    // 3) Smooth accel/decel toward wishVel (Half-Life feel)
    const accel = this.isGrounded ? T.PLAYER_GROUND_ACCEL : T.PLAYER_AIR_ACCEL
    this.velocity.x += (wishVel.x - this.velocity.x) * Math.min(1, accel * dt)
    this.velocity.z += (wishVel.z - this.velocity.z) * Math.min(1, accel * dt)

    // 4) Gravity + jump
    if (this.isGrounded && this.input.jump) {
      this.velocity.y = T.PLAYER_JUMP_VELOCITY
    }
    this.velocity.y -= T.GRAVITY * dt

    // 5) Crouch toggle handles collider resize
    if (this.input.crouch !== this.isCrouched) {
      this.setCrouched(this.input.crouch)
    }

    // 6) Move via CCT
    this.cct.computeColliderMovement(this.collider, this.velocity.clone().multiplyScalar(dt))
    const corrected = this.cct.computedMovement()
    const t = this.body.translation()
    this.body.setNextKinematicTranslation({ x: t.x + corrected.x, y: t.y + corrected.y, z: t.z + corrected.z })
    this.isGrounded = this.cct.computedGrounded()
    if (this.isGrounded && this.velocity.y < 0) this.velocity.y = 0

    // 7) Update camera transform
    const head = this.body.translation()
    const eyeY = this.isCrouched ? T.PLAYER_EYE_CROUCH : T.PLAYER_EYE_STANDING
    this.camera.position.set(head.x, head.y + eyeY - T.PLAYER_HEIGHT_STANDING / 2, head.z)
    this.camera.rotation.set(this.pitch, this.yaw, 0, 'YXZ')

    // 8) Emit stamina change so HUD reacts
    this.engine!.events.emit({ type: 'StaminaChanged', current: this.stamina, max: T.STAMINA_MAX })
  }

  private setCrouched(crouched: boolean) {
    this.isCrouched = crouched
    // Resize the capsule by swapping the collider's half-height. Rapier doesn't allow live edit,
    // so we recreate the collider attached to the same body.
    const physics = this.engine!.physics
    physics.world.removeCollider(this.collider, false)
    const h = crouched ? T.PLAYER_HEIGHT_CROUCHED : T.PLAYER_HEIGHT_STANDING
    const desc = physics.rapier.ColliderDesc.capsule((h - 2 * T.PLAYER_RADIUS) / 2, T.PLAYER_RADIUS)
    this.collider = physics.world.createCollider(desc, this.body)
  }
}
```

### 9.3 Input mapping

| Action       | Key            | Notes                                                     |
| ------------ | -------------- | --------------------------------------------------------- |
| Forward      | `W`            |                                                           |
| Back         | `S`            |                                                           |
| Strafe left  | `A`            |                                                           |
| Strafe right | `D`            |                                                           |
| Jump         | `Space`        | Only fires while grounded                                 |
| Sprint       | `Shift` (hold) | Drains stamina; auto-cancels at stamina 0                 |
| Crouch       | `LCtrl` (toggle) | Press to crouch, press again to stand                   |
| Look         | Mouse          | Pointer-lock; sensitivity from settings                   |
| Attack       | `LMB`          | Pipe swing; weapon-specific cooldown                      |
| Interact     | `E`            | Pickups, locked boxes, clocks                             |
| Pause        | `Esc`          | Releases pointer-lock; opens Angular pause modal          |

Rebinding is owned by the Angular shell; the engine receives an `InputState` snapshot per frame.

### 9.4 First-person hands rig

When `showFirstPersonHands` is true (the default), the engine spawns an `FPArms` mesh as a child of the camera at a fixed offset. The mesh is animated by a per-weapon `AnimationMixer`:

```typescript
// actors/FpArms.ts
import { AnimationMixer, Group, Vector3 } from 'three'

export class FpArms {
  group = new Group()
  private mixer?: AnimationMixer
  private currentAnim?: string

  attach(camera: Camera) {
    camera.add(this.group)
    this.group.position.set(0.15, -0.35, -0.4)   // tuned per asset
  }

  loadWeapon(weaponSlug: string, modelRoot: Object3D, anims: AnimationClip[]) {
    this.group.clear()
    this.group.add(modelRoot)
    this.mixer = new AnimationMixer(modelRoot)
    for (const clip of anims) this.mixer.clipAction(clip).play()
  }

  play(animName: string, fadeMs = 60) { /* … crossfade between named clips … */ }
  update(dt: number) { this.mixer?.update(dt) }
}
```

When `showFirstPersonHands` is false, the `FpArms` is omitted and only the weapon model is added to the camera (`weapon.position.set(0.25, -0.25, -0.5)` or similar).

**Asset fallback (MVP only).** The FP hands rig assumes the chosen asset pack ships four per-weapon poses: `idle`, `swing-windup`, `swing-hit`, and `hide`. If the Kenney FPS Pack (or whichever pack we land on) does *not* ship these, MVP ships with `showFirstPersonHands: false` as the visible default and the settings toggle becomes a no-op for the slice. Authoring the four poses in Blender against a generic arm rig is a v0.2 task. This keeps MVP unblocked on asset availability.

---

## 10. Combat — Pipe Melee + Ragdoll Death

### 10.1 Weapon base

```typescript
// actors/weapons/Weapon.ts
export abstract class Weapon {
  abstract readonly slug: string
  abstract readonly maxDurability: number
  currentDurability: number = this.maxDurability
  protected lastUseAt = -Infinity

  abstract use(engine: GameEngine, originator: Entity, dt: number): void

  canUse(now: number, cycleSeconds: number): boolean {
    return now - this.lastUseAt >= cycleSeconds && this.currentDurability > 0
  }
}
```

### 10.2 The Pipe

```typescript
// actors/weapons/Pipe.ts
import { Raycaster, Vector3 } from 'three'
import { Weapon } from './Weapon'
import { tuning as T } from '../../tuning'

export class Pipe extends Weapon {
  readonly slug = 'pipe'
  readonly maxDurability = T.PIPE_DURABILITY

  use(engine: GameEngine, player: PlayerController, dt: number): void {
    const now = engine.timeNow
    if (!this.canUse(now, T.PIPE_SWING_CYCLE_S)) return
    this.lastUseAt = now

    // Play swing anim; the hit frame is at swing-anim-time ≈ 0.18s
    engine.fpArms?.play('pipe.swing')
    engine.audio.playOneShot('pipeSwing')

    // The hit check is delayed to the animation's hit frame so the visual matches
    engine.scheduler.after(T.PIPE_SWING_HIT_FRAME_S, () => this.resolveHit(engine, player))

    this.currentDurability -= T.PIPE_DURABILITY_LOSS_PER_HIT
    if (this.currentDurability <= 0) {
      engine.events.emit({ type: 'WeaponBroke', weaponSlug: this.slug })
    }
  }

  private resolveHit(engine: GameEngine, player: PlayerController) {
    // Capsule-cast a short distance in front of the camera
    const origin = new Vector3().setFromMatrixPosition(player.camera.matrixWorld)
    const forward = new Vector3(0, 0, -1).applyQuaternion(player.camera.quaternion)
    const reach = T.PIPE_REACH_M

    const hit = engine.physics.castShape({
      shape: { type: 'capsule', halfHeight: 0.05, radius: T.PIPE_HIT_RADIUS_M },
      origin,
      direction: forward,
      distance: reach,
      groups: T.PHYSICS_GROUP_ENEMY,
    })

    if (!hit) return

    const enemy = engine.findEnemyByColliderHandle(hit.colliderHandle)
    if (!enemy) return

    enemy.takeDamage(T.PIPE_SWING_DAMAGE, forward)
    engine.audio.playOneShotAt('pipeHit', enemy.position)
    engine.spawnParticleBurst('blood', enemy.position, 8)
  }
}
```

### 10.3 Enemy hit reaction

```typescript
// actors/enemies/Enemy.ts (excerpt — full class in §11)
takeDamage(amount: number, fromDirection: Vector3) {
  this.hp -= amount
  this.mesh.userData.flashUntil = performance.now() / 1000 + T.HIT_FLASH_S
  this.applyKnockback(fromDirection, T.HIT_KNOCKBACK_IMPULSE)
  this.state.transition('Stagger')
  if (this.hp <= 0) {
    this.die(fromDirection)
  } else {
    this.engine!.events.emit({ type: 'EnemyDamaged', enemyId: this.id, remainingHp: this.hp })
  }
}
```

### 10.4 Ragdoll death

On death, we **replace** the animated character with a Rapier-driven ragdoll: a small set of capsule bodies joined by spherical joints. Each enemy archetype gets its **own** ragdoll config so deaths feel distinct (a Mini Zombie should crumple differently from a Pipe Zombie, which crumples differently from a Sweeper). The configs share a single `RagdollConfig` type.

```typescript
// actors/enemies/ragdoll.ts
import RAPIER from '@dimforge/rapier3d-compat'

export interface RagdollConfig {
  bones: { name: string; halfHeight: number; radius: number; localOffset: Vec3 }[]
  joints: { parent: string; child: string; anchorParent: Vec3; anchorChild: Vec3 }[]
}

// Pipe Zombie — adult humanoid. 9 bones (torso + segmented arms + segmented legs).
export const PIPE_ZOMBIE_RAGDOLL: RagdollConfig = {
  bones: [
    { name: 'torso',   halfHeight: 0.30, radius: 0.18, localOffset: { x: 0,     y: 0.90, z: 0 } },
    { name: 'lUpArm',  halfHeight: 0.14, radius: 0.07, localOffset: { x: -0.25, y: 1.15, z: 0 } },
    { name: 'lLoArm',  halfHeight: 0.14, radius: 0.06, localOffset: { x: -0.45, y: 1.05, z: 0 } },
    { name: 'rUpArm',  halfHeight: 0.14, radius: 0.07, localOffset: { x:  0.25, y: 1.15, z: 0 } },
    { name: 'rLoArm',  halfHeight: 0.14, radius: 0.06, localOffset: { x:  0.45, y: 1.05, z: 0 } },
    { name: 'lUpLeg',  halfHeight: 0.20, radius: 0.09, localOffset: { x: -0.10, y: 0.45, z: 0 } },
    { name: 'lLoLeg',  halfHeight: 0.20, radius: 0.07, localOffset: { x: -0.10, y: 0.10, z: 0 } },
    { name: 'rUpLeg',  halfHeight: 0.20, radius: 0.09, localOffset: { x:  0.10, y: 0.45, z: 0 } },
    { name: 'rLoLeg',  halfHeight: 0.20, radius: 0.07, localOffset: { x:  0.10, y: 0.10, z: 0 } },
  ],
  joints: [
    { parent: 'torso',  child: 'lUpArm', anchorParent: { x: -0.18, y:  0.25, z: 0 }, anchorChild: { x: 0, y:  0.14, z: 0 } },
    { parent: 'lUpArm', child: 'lLoArm', anchorParent: { x: 0,     y: -0.14, z: 0 }, anchorChild: { x: 0, y:  0.14, z: 0 } },
    { parent: 'torso',  child: 'rUpArm', anchorParent: { x:  0.18, y:  0.25, z: 0 }, anchorChild: { x: 0, y:  0.14, z: 0 } },
    { parent: 'rUpArm', child: 'rLoArm', anchorParent: { x: 0,     y: -0.14, z: 0 }, anchorChild: { x: 0, y:  0.14, z: 0 } },
    { parent: 'torso',  child: 'lUpLeg', anchorParent: { x: -0.10, y: -0.30, z: 0 }, anchorChild: { x: 0, y:  0.20, z: 0 } },
    { parent: 'lUpLeg', child: 'lLoLeg', anchorParent: { x: 0,     y: -0.20, z: 0 }, anchorChild: { x: 0, y:  0.20, z: 0 } },
    { parent: 'torso',  child: 'rUpLeg', anchorParent: { x:  0.10, y: -0.30, z: 0 }, anchorChild: { x: 0, y:  0.20, z: 0 } },
    { parent: 'rUpLeg', child: 'rLoLeg', anchorParent: { x: 0,     y: -0.20, z: 0 }, anchorChild: { x: 0, y:  0.20, z: 0 } },
  ],
}

// Mini Zombie — short, light, simpler 5-bone skeleton (single-segment limbs).
// Tumbles fast and floppy; doesn't fold at elbows/knees.
export const MINI_ZOMBIE_RAGDOLL: RagdollConfig = {
  bones: [
    { name: 'torso', halfHeight: 0.18, radius: 0.12, localOffset: { x: 0,     y: 0.55, z: 0 } },
    { name: 'lArm',  halfHeight: 0.18, radius: 0.05, localOffset: { x: -0.20, y: 0.65, z: 0 } },
    { name: 'rArm',  halfHeight: 0.18, radius: 0.05, localOffset: { x:  0.20, y: 0.65, z: 0 } },
    { name: 'lLeg',  halfHeight: 0.22, radius: 0.06, localOffset: { x: -0.08, y: 0.22, z: 0 } },
    { name: 'rLeg',  halfHeight: 0.22, radius: 0.06, localOffset: { x:  0.08, y: 0.22, z: 0 } },
  ],
  joints: [
    { parent: 'torso', child: 'lArm', anchorParent: { x: -0.12, y:  0.15, z: 0 }, anchorChild: { x: 0, y:  0.18, z: 0 } },
    { parent: 'torso', child: 'rArm', anchorParent: { x:  0.12, y:  0.15, z: 0 }, anchorChild: { x: 0, y:  0.18, z: 0 } },
    { parent: 'torso', child: 'lLeg', anchorParent: { x: -0.08, y: -0.18, z: 0 }, anchorChild: { x: 0, y:  0.22, z: 0 } },
    { parent: 'torso', child: 'rLeg', anchorParent: { x:  0.08, y: -0.18, z: 0 }, anchorChild: { x: 0, y:  0.22, z: 0 } },
  ],
}

// Sweeper — bulky humanoid + separate neck + head. 11 bones. Heavier radii give it
// presence; the loose head joint sells the "thing collapsing" silhouette on death.
// (The Sweeper is invincible during gameplay; this config only kicks in if the run
// ends and we want a cinematic death — e.g. a future "destroy the mothership" beat.)
export const SWEEPER_RAGDOLL: RagdollConfig = {
  bones: [
    { name: 'head',    halfHeight: 0.10, radius: 0.16, localOffset: { x: 0,     y: 2.10, z: 0 } },
    { name: 'neck',    halfHeight: 0.06, radius: 0.10, localOffset: { x: 0,     y: 1.85, z: 0 } },
    { name: 'torso',   halfHeight: 0.42, radius: 0.28, localOffset: { x: 0,     y: 1.30, z: 0 } },
    { name: 'lUpArm',  halfHeight: 0.20, radius: 0.11, localOffset: { x: -0.38, y: 1.65, z: 0 } },
    { name: 'lLoArm',  halfHeight: 0.20, radius: 0.10, localOffset: { x: -0.65, y: 1.40, z: 0 } },
    { name: 'rUpArm',  halfHeight: 0.20, radius: 0.11, localOffset: { x:  0.38, y: 1.65, z: 0 } },
    { name: 'rLoArm',  halfHeight: 0.20, radius: 0.10, localOffset: { x:  0.65, y: 1.40, z: 0 } },
    { name: 'lUpLeg',  halfHeight: 0.30, radius: 0.14, localOffset: { x: -0.15, y: 0.65, z: 0 } },
    { name: 'lLoLeg',  halfHeight: 0.30, radius: 0.11, localOffset: { x: -0.15, y: 0.15, z: 0 } },
    { name: 'rUpLeg',  halfHeight: 0.30, radius: 0.14, localOffset: { x:  0.15, y: 0.65, z: 0 } },
    { name: 'rLoLeg',  halfHeight: 0.30, radius: 0.11, localOffset: { x:  0.15, y: 0.15, z: 0 } },
  ],
  joints: [
    { parent: 'neck',   child: 'head',   anchorParent: { x: 0,    y:  0.06, z: 0 }, anchorChild: { x: 0, y: -0.10, z: 0 } },
    { parent: 'torso',  child: 'neck',   anchorParent: { x: 0,    y:  0.40, z: 0 }, anchorChild: { x: 0, y: -0.06, z: 0 } },
    { parent: 'torso',  child: 'lUpArm', anchorParent: { x: -0.28, y:  0.35, z: 0 }, anchorChild: { x: 0, y:  0.20, z: 0 } },
    { parent: 'lUpArm', child: 'lLoArm', anchorParent: { x: 0,     y: -0.20, z: 0 }, anchorChild: { x: 0, y:  0.20, z: 0 } },
    { parent: 'torso',  child: 'rUpArm', anchorParent: { x:  0.28, y:  0.35, z: 0 }, anchorChild: { x: 0, y:  0.20, z: 0 } },
    { parent: 'rUpArm', child: 'rLoArm', anchorParent: { x: 0,     y: -0.20, z: 0 }, anchorChild: { x: 0, y:  0.20, z: 0 } },
    { parent: 'torso',  child: 'lUpLeg', anchorParent: { x: -0.15, y: -0.42, z: 0 }, anchorChild: { x: 0, y:  0.30, z: 0 } },
    { parent: 'lUpLeg', child: 'lLoLeg', anchorParent: { x: 0,     y: -0.30, z: 0 }, anchorChild: { x: 0, y:  0.30, z: 0 } },
    { parent: 'torso',  child: 'rUpLeg', anchorParent: { x:  0.15, y: -0.42, z: 0 }, anchorChild: { x: 0, y:  0.30, z: 0 } },
    { parent: 'rUpLeg', child: 'rLoLeg', anchorParent: { x: 0,     y: -0.30, z: 0 }, anchorChild: { x: 0, y:  0.30, z: 0 } },
  ],
}

export const RAGDOLLS_BY_KIND: Record<string, RagdollConfig> = {
  'pipe-zombie': PIPE_ZOMBIE_RAGDOLL,
  'mini-zombie': MINI_ZOMBIE_RAGDOLL,
  'sweeper-zombie': SWEEPER_RAGDOLL,
}

export function spawnRagdoll(
  engine: GameEngine,
  cfg: RagdollConfig,
  rootPosition: Vec3,
  initialImpulse: Vec3,
  ttlSeconds: number,
): void { /* … instantiate bones + joints, attach skinned mesh, schedule despawn … */ }
```

Ragdolls are tagged with a `ttl` and despawned after `T.RAGDOLL_TTL_S` (default 8 s) to bound on-screen body counts. On despawn we play a "sink" tween that fades them through the floor.

The `Enemy.die()` method picks the config from `RAGDOLLS_BY_KIND[this.cfg.kind]` so enemy classes don't need to know about ragdoll shapes — they just declare their `kind` and the right ragdoll wakes up.

### 10.5 Player damage feedback

```typescript
// actors/PlayerHurt.ts (helper, owned by Player)
takeDamage(amount: number, sourceKind: string, fromDirection: Vector3) {
  if (this.iframesUntil > performance.now() / 1000) return     // 0.5s i-frames
  this.iframesUntil = performance.now() / 1000 + T.PLAYER_IFRAMES_S
  this.hp = Math.max(0, this.hp - amount)
  this.engine!.events.emit({ type: 'PlayerHit', damage: amount, remainingHp: this.hp, sourceKind })
  this.engine!.camera.applyShake(T.PLAYER_HIT_SHAKE_AMPL, T.PLAYER_HIT_SHAKE_S)
  // HUD listens for PlayerHit and shows red vignette overlay
  if (this.hp <= 0) {
    this.engine!.events.emit({ type: 'PlayerDied', killer: sourceKind })
  } else if (this.hp / this.maxHp < 0.25) {
    this.engine!.events.emit({ type: 'LowHealth' })
  }
}
```

---

## 11. Enemy AI — State Machine

### 11.1 Base `Enemy` and the state machine

```typescript
// actors/enemies/Enemy.ts
import { Vector3 } from 'three'
import { Entity } from '../Entity'
import { StateMachine } from '../../ai/StateMachine'
import type { GameEngine } from '../../engine/GameEngine'

export type EnemyState = 'Idle' | 'Detect' | 'Pursue' | 'Attack' | 'Stagger' | 'Die'

export interface EnemyConfig {
  kind: string
  maxHp: number
  hitDamage: number
  moveSpeed: number          // m/s
  attackReach: number        // m
  attackCycleS: number       // seconds between attacks
  detectionRangeM: number    // m — passive detection
  aggroRangeM: number        // m — once seen, will pursue this far
  scalesWithPlayerLevel: boolean
}

export abstract class Enemy extends Entity {
  id: number
  hp: number
  protected target?: Entity              // usually the player
  protected state: StateMachine<EnemyState>
  protected attackedAt = -Infinity
  protected currentPath?: Vector3[]
  protected pathIndex = 0

  constructor(readonly cfg: EnemyConfig, public playerLevel: number) {
    super()
    this.hp = cfg.maxHp + (cfg.scalesWithPlayerLevel ? Math.max(0, playerLevel - 1) : 0)
    this.state = new StateMachine<EnemyState>('Idle')
  }

  update(dt: number): void {
    switch (this.state.current) {
      case 'Idle':    this.tickIdle(dt);    break
      case 'Detect':  this.tickDetect(dt);  break
      case 'Pursue':  this.tickPursue(dt);  break
      case 'Attack':  this.tickAttack(dt);  break
      case 'Stagger': this.tickStagger(dt); break
      case 'Die':     /* no-op; ragdoll handles physics */ break
    }
  }

  protected tickIdle(_dt: number) {
    const player = this.engine!.player
    if (this.canSee(player) && this.distanceTo(player) < this.cfg.detectionRangeM) {
      this.state.transition('Detect')
    }
  }

  protected tickDetect(_dt: number) {
    // Brief 'noticed you' delay so the AI feels reactive, not omniscient
    this.engine!.audio.playOneShotAt('zombieGroan', this.position, 0.6)
    setTimeout(() => this.state.transition('Pursue'), T.ENEMY_DETECT_DELAY_MS)
  }

  protected tickPursue(dt: number) {
    const player = this.engine!.player
    const dist = this.distanceTo(player)
    if (dist <= this.cfg.attackReach) {
      this.state.transition('Attack')
      return
    }
    if (dist > this.cfg.aggroRangeM) {
      this.state.transition('Idle')
      this.currentPath = undefined
      return
    }
    this.followPath(dt, player.position)
  }

  protected tickAttack(_dt: number) {
    const now = performance.now() / 1000
    if (now - this.attackedAt < this.cfg.attackCycleS) return
    this.attackedAt = now
    const player = this.engine!.player
    const dir = new Vector3().subVectors(player.position, this.position).normalize()
    player.takeDamage(this.cfg.hitDamage, this.cfg.kind, dir)
    if (this.distanceTo(player) > this.cfg.attackReach) {
      this.state.transition('Pursue')
    }
  }

  protected tickStagger(dt: number) {
    if (performance.now() / 1000 > this.staggerUntil) {
      this.state.transition('Pursue')
    }
  }

  protected followPath(dt: number, goal: Vector3) {
    if (!this.currentPath || this.pathStale(goal)) {
      this.currentPath = this.engine!.navMesh.findPath(this.position, goal) ?? []
      this.pathIndex = 0
    }
    if (!this.currentPath.length) return
    const next = this.currentPath[this.pathIndex]
    const toNext = new Vector3().subVectors(next, this.position)
    if (toNext.lengthSq() < 0.04) {
      this.pathIndex++
      if (this.pathIndex >= this.currentPath.length) { this.currentPath = undefined; return }
    }
    toNext.normalize().multiplyScalar(this.cfg.moveSpeed * dt)
    this.position.add(toNext)
  }

  protected canSee(other: Entity): boolean {
    return this.engine!.physics.lineOfSight(this.position, other.position, T.PHYSICS_GROUP_WALL)
  }

  protected distanceTo(other: Entity): number {
    return this.position.distanceTo(other.position)
  }

  protected die(fromDirection: Vector3) {
    this.state.transition('Die')
    spawnRagdoll(this.engine!, HUMANOID_RAGDOLL, this.position, fromDirection, T.RAGDOLL_TTL_S)
    this.engine!.events.emit({
      type: 'EnemyKilled',
      enemyKind: this.cfg.kind as EnemyKind,
      position: { ...this.position },
      pointsDelta: this.cfg.maxHp,                          // simplified per PDF: HP ≈ point yield
      goldDelta: tuning.GOLD_PER_KIND[this.cfg.kind] ?? 0,
      multiplier: this.engine!.scoring.currentMultiplier(),
    })
    this.engine!.despawnActor(this)
  }
}
```

### 11.2 PipeZombie + MiniZombie configs

```typescript
// actors/enemies/PipeZombie.ts
export class PipeZombie extends Enemy {
  constructor(playerLevel: number) {
    super({
      kind: 'pipe-zombie',
      maxHp: T.PIPE_ZOMBIE_HP,
      hitDamage: T.PIPE_ZOMBIE_DMG,
      moveSpeed: T.PIPE_ZOMBIE_SPEED,
      attackReach: T.PIPE_ZOMBIE_REACH,
      attackCycleS: T.PIPE_ZOMBIE_ATTACK_CYCLE,
      detectionRangeM: T.PIPE_ZOMBIE_DETECT,
      aggroRangeM: T.PIPE_ZOMBIE_AGGRO,
      scalesWithPlayerLevel: false,
    }, playerLevel)
  }
}

// actors/enemies/MiniZombie.ts
export class MiniZombie extends Enemy {
  constructor(playerLevel: number) {
    super({
      kind: 'mini-zombie',
      maxHp: T.MINI_ZOMBIE_HP,
      hitDamage: T.MINI_ZOMBIE_DMG,
      moveSpeed: T.MINI_ZOMBIE_SPEED,
      attackReach: T.MINI_ZOMBIE_REACH,
      attackCycleS: T.MINI_ZOMBIE_ATTACK_CYCLE,
      detectionRangeM: T.MINI_ZOMBIE_DETECT,
      aggroRangeM: T.MINI_ZOMBIE_AGGRO,
      scalesWithPlayerLevel: false,
    }, playerLevel)
  }
}
```

### 11.3 Generic state machine

```typescript
// ai/StateMachine.ts
export class StateMachine<S extends string> {
  current: S
  onEnter = new Map<S, () => void>()
  onExit  = new Map<S, () => void>()
  constructor(initial: S) { this.current = initial }

  transition(to: S): void {
    if (this.current === to) return
    this.onExit.get(this.current)?.()
    this.current = to
    this.onEnter.get(to)?.()
  }
}
```

---

## 12. Sweeper Zombie

The Sweeper is a special enemy: **invincible**, released by the dungeon timer, always knows the player's room, and pathfinds through the room graph at a steady pace until it catches up.

### 12.1 Behavior

```typescript
// actors/enemies/SweeperZombie.ts
import { Enemy } from './Enemy'
import { tuning as T } from '../../tuning'

export class SweeperZombie extends Enemy {
  constructor(playerLevel: number) {
    super({
      kind: 'sweeper-zombie',
      maxHp: Number.POSITIVE_INFINITY,
      hitDamage: T.SWEEPER_DMG,
      moveSpeed: T.SWEEPER_SPEED,
      attackReach: T.SWEEPER_REACH,
      attackCycleS: T.SWEEPER_ATTACK_CYCLE,
      detectionRangeM: Number.POSITIVE_INFINITY,    // always knows the player
      aggroRangeM:    Number.POSITIVE_INFINITY,
      scalesWithPlayerLevel: false,
    }, playerLevel)
    this.state.transition('Pursue')                  // never idles
  }

  takeDamage(_amount: number, _from: Vector3): void {
    // Invincible. Optional: play a "clang" sfx so the player feels feedback.
    this.engine?.audio.playOneShotAt('sweeperClang', this.position, 0.4)
  }

  tickPursue(dt: number) {
    // Always target the player's current room center first; once in that room, target the player directly.
    const player = this.engine!.player
    const playerRoom = this.engine!.dungeon.roomContainingPoint(player.position)
    const myRoom = this.engine!.dungeon.roomContainingPoint(this.position)
    const goal = myRoom === playerRoom ? player.position : playerRoom.center
    this.followPath(dt, goal)

    // Proximity audio: increasingly loud footstep loop within 12 m
    const dist = this.distanceTo(player)
    const intensity = Math.max(0, 1 - dist / T.SWEEPER_PROX_LOUD_RANGE)
    this.engine!.audio.setLoopGain('sweeperSteps', intensity)
  }
}
```

### 12.2 Release

The Sweeper is **not present at the start of the run**. It's spawned by the timer service (§17) when the hidden countdown hits zero. On spawn:

1. The dungeon picks the room **farthest from the player**, measured by BFS over the room edge graph **and** filtered to require at least `T.SWEEPER_MIN_WORLD_DIST_M` (8 m) of straight-line distance from the player. If no room satisfies both criteria, fall back to the pure-BFS winner. This avoids the failure mode where a branchy procgen layout's "graph-farthest" room is physically right next to the player.
2. A `SweeperReleased` event is emitted to the HUD.
3. An ominous one-shot (`sweeperRoar`) plays from the spawn position (the player hears it directional).
4. The looping footsteps sfx is mounted and its gain ramps up with proximity.

```typescript
// engine/sweeper-release.ts
export function releaseSweeper(engine: GameEngine) {
  const player = engine.player
  const farRoom = engine.dungeon.farthestRoomFrom(player.position)
  const sweeper = new SweeperZombie(player.level)
  sweeper.position.copy(farRoom.center)
  engine.spawnActor(sweeper)
  engine.audio.playOneShotAt('sweeperRoar', sweeper.position)
  engine.audio.startLoop('sweeperSteps', sweeper)
  engine.events.emit({ type: 'SweeperReleased' })
}
```

---

## 13. Navmesh & Pathfinding

### 13.1 Navmesh service

We bake the navmesh from the dungeon's collision geometry **after** procgen finishes. The bake takes ~100–300 ms for an MVP-sized dungeon and runs once per run.

```typescript
// ai/NavMeshService.ts
import { init as initRecast, NavMesh, NavMeshQuery, generateSoloNavMesh } from '@recast-navigation/core'
import { threeToSoloNavMeshArgs } from '@recast-navigation/three'
import { Vector3, Mesh, Object3D } from 'three'

export class NavMeshService {
  private mesh!: NavMesh
  private query!: NavMeshQuery

  async init() { await initRecast() }

  async build(meshes: Mesh[]) {
    const args = threeToSoloNavMeshArgs(meshes, {
      cs: 0.2,                     // cell size — 0.2 m grid
      ch: 0.2,                     // cell height
      walkableSlopeAngle: 50,
      walkableHeight: 18,          // in voxels (= 18 * ch = 3.6 m)
      walkableRadius: 2,           // = 0.4 m
      walkableClimb: 1,
      minRegionArea: 8,
      mergeRegionArea: 20,
    })
    const result = generateSoloNavMesh(...args)
    if (!result.success) throw new Error('navmesh bake failed: ' + result.error)
    this.mesh = result.navMesh
    this.query = new NavMeshQuery({ navMesh: this.mesh })
  }

  findPath(from: Vector3, to: Vector3): Vector3[] | null {
    const result = this.query.computePath(
      { x: from.x, y: from.y, z: from.z },
      { x: to.x,   y: to.y,   z: to.z   },
    )
    if (!result.success) return null
    return result.path.map(p => new Vector3(p.x, p.y, p.z))
  }

  destroy() { this.mesh?.destroy(); this.query?.destroy() }
}
```

### 13.2 Path caching strategy

Enemies cache their last path and re-query when **either**:

- The player has moved more than `T.PATH_REPLAN_DIST_M` (default 1.5 m) since the last query, OR
- The enemy has reached the end of the cached path.

This caps navmesh queries at roughly `enemyCount × 1 Hz` in steady state.

```typescript
// ai/path-cache.ts (used inside Enemy.followPath)
protected pathStale(currentGoal: Vector3): boolean {
  if (!this.currentPath || !this.lastGoal) return true
  return currentGoal.distanceTo(this.lastGoal) > T.PATH_REPLAN_DIST_M
}
```

---

## 14. Procedural Generation — Branching Tree

The PDF specifies up to 4 doors per room. We build a **branching tree** of rooms: starting from a root, we randomly attach children until we hit a target room count.

### 14.1 Algorithm

```
1. Pick MVP room count from [T.MIN_ROOMS, T.MAX_ROOMS] using the run PRNG (default 5–8).
2. Place the root room at origin, with the entrance on its south wall.
3. While roomCount < target:
     a. From the pool of rooms with free door slots, pick one (weighted by depth — biased
        toward shallower rooms so the tree stays "bushy").
     b. Pick a free door slot.
     c. Generate a new room template (in MVP this is always LockerRoom).
     d. Place the new room on the other side of the door, oriented so its own door
        aligns with the parent door. Check for collision with already-placed rooms;
        on collision, retry up to T.PLACEMENT_RETRIES, otherwise mark this slot
        "blocked" and continue.
     e. Insert the new room into the graph; consume the door slot on both sides.
4. After all rooms are placed, ensure exactly one room is the "exit" — the one
    farthest from the root by graph distance. The exit room contains the "return
    to surface" interactable that ends the run successfully.
```

### 14.2 Types

```typescript
// procgen/types.ts
export type Direction = 'N' | 'E' | 'S' | 'W'

export interface DoorSlot {
  direction: Direction         // wall this door is on (in the room's local frame)
  localPosition: Vec3          // door center, in room-local space
  width: number                // m
  height: number               // m
  connected?: number           // index of the room on the other side, if any
}

export interface RoomInstance {
  index: number                // 0 is the root
  template: RoomTemplate
  worldPosition: Vec3          // origin of the room in world space
  worldRotation: number        // yaw in radians (multiples of pi/2)
  doors: DoorSlot[]
  enemies: EnemyInstance[]
  loot: LootInstance[]
  obstacles: ObstacleInstance[]
  isExit: boolean
}

export interface DungeonGraph {
  rooms: RoomInstance[]
  edges: Array<[number, number]>     // [roomA, roomB]
  rootIndex: number
  exitIndex: number
}
```

### 14.3 The generator

```typescript
// procgen/DungeonGenerator.ts
import { RunPrng } from './prng'
import { LockerRoom } from './LockerRoom'
import { tuning as T } from '../tuning'

export class DungeonGenerator {
  constructor(private prng: RunPrng) {}

  generate(): DungeonGraph {
    const targetRoomCount = this.prng.int(T.MIN_ROOMS, T.MAX_ROOMS)
    const rooms: RoomInstance[] = []
    const edges: Array<[number, number]> = []

    // 1. Root
    const root = this.instantiateRoom(LockerRoom, { x: 0, y: 0, z: 0 }, 0)
    root.index = 0
    rooms.push(root)

    // 2. Grow
    while (rooms.length < targetRoomCount) {
      const candidate = this.pickRoomWithFreeDoor(rooms)
      if (!candidate) break
      const slot = this.pickFreeDoor(candidate, rooms)
      if (!slot) break

      let placed: RoomInstance | null = null
      for (let attempt = 0; attempt < T.PLACEMENT_RETRIES && !placed; attempt++) {
        const childTemplate = LockerRoom    // MVP: only one template
        const candidateRoom = this.instantiateRoom(childTemplate, ZERO_VEC, 0)
        const childSlot = this.pickInverseDoor(candidateRoom, slot.direction)
        if (!childSlot) continue
        const transform = this.computeAlignmentTransform(candidate, slot, candidateRoom, childSlot)
        candidateRoom.worldPosition = transform.position
        candidateRoom.worldRotation = transform.rotation
        if (!this.collides(candidateRoom, rooms)) {
          candidateRoom.index = rooms.length
          slot.connected = candidateRoom.index
          childSlot.connected = candidate.index
          rooms.push(candidateRoom)
          edges.push([candidate.index, candidateRoom.index])
          placed = candidateRoom
        }
      }
      if (!placed) slot.connected = -1     // mark blocked
    }

    // 3. Pick exit
    const exitIndex = this.farthestFromRoot(rooms, edges, 0)
    rooms[exitIndex].isExit = true

    return { rooms, edges, rootIndex: 0, exitIndex }
  }

  // … implementation of helpers: pickRoomWithFreeDoor, pickFreeDoor, instantiateRoom,
  // computeAlignmentTransform (matrix that aligns child door to parent door),
  // collides (AABB overlap check across rooms), farthestFromRoot (BFS).
}
```

### 14.4 Spatial collision

Each `RoomTemplate` declares an axis-aligned bounding box (`bboxLocal`). After rotating into world space, we check overlap against every already-placed room. The check is cheap because room counts are small and we don't rotate by arbitrary angles (only 90° increments).

### 14.5 Room graph access at runtime

```typescript
// procgen/DungeonGraph runtime helpers
export class Dungeon {
  constructor(readonly graph: DungeonGraph) {}

  roomContainingPoint(p: Vector3): RoomInstance {
    // O(rooms). For MVP we don't bother with a spatial index.
    for (const room of this.graph.rooms) {
      if (this.pointInRoom(p, room)) return room
    }
    return this.graph.rooms[this.graph.rootIndex]
  }

  farthestRoomFrom(p: Vector3, minWorldDist = T.SWEEPER_MIN_WORLD_DIST_M): RoomInstance {
    const here = this.roomContainingPoint(p)
    const dists = bfsRoomDistances(this.graph, here.index)
    let best = here.index,         bestBfs = 0       // best room that ALSO clears the world-dist floor
    let fallback = here.index,     fallbackBfs = 0   // best room by BFS only, ignoring world distance
    for (let i = 0; i < dists.length; i++) {
      if (dists[i] > fallbackBfs) { fallback = i; fallbackBfs = dists[i] }
      const worldDist = p.distanceTo(this.graph.rooms[i].center as any)
      if (worldDist < minWorldDist) continue
      if (dists[i] > bestBfs) { best = i; bestBfs = dists[i] }
    }
    // If no room cleared the world-distance floor, fall back to pure BFS so we still spawn somewhere.
    return this.graph.rooms[bestBfs > 0 ? best : fallback]
  }
}
```

---

## 15. Room Kit — Locker Room

The MVP ships **one** room template. It's the gym-locker-room described in PDF Room 1. The kit is a small set of modular meshes the `RoomBuilder` instantiates per room.

### 15.1 Template definition

```typescript
// procgen/LockerRoom.ts
import { tuning as T } from '../tuning'

export const LockerRoom: RoomTemplate = {
  slug: 'locker-room',
  bboxLocal: {
    min: { x: -4, y: 0, z: -3 },
    max: { x:  4, y: 3, z:  3 },
  },
  doors: [
    { direction: 'N', localPosition: { x: 0,   y: 0, z: -3 }, width: 1.4, height: 2.2 },
    { direction: 'S', localPosition: { x: 0,   y: 0, z:  3 }, width: 1.4, height: 2.2 },
    { direction: 'E', localPosition: { x: 4,   y: 0, z:  0 }, width: 1.4, height: 2.2 },
    { direction: 'W', localPosition: { x: -4,  y: 0, z:  0 }, width: 1.4, height: 2.2 },
  ],
  enemySpawns: [
    { weight: 0.6, kind: 'pipe-zombie', localPosition: { x:  1.5, y: 0, z:  0 } },
    { weight: 0.4, kind: 'pipe-zombie', localPosition: { x: -1.5, y: 0, z:  0 } },
    { weight: 0.5, kind: 'mini-zombie', localPosition: { x:  0,   y: 0, z:  1.5 } },
    { weight: 0.3, kind: 'mini-zombie', localPosition: { x:  0,   y: 0, z: -1.5 } },
  ],
  lootSpawns: [
    { weight: 0.7, slug: 'gold-pile-small',  localPosition: { x:  3.5, y: 0, z:  2.5 } },
    { weight: 0.3, slug: 'health-potion',    localPosition: { x: -3.5, y: 0, z: -2.5 } },
  ],
  obstacleSpawns: [
    { weight: 0.5, slug: 'locked-box',       localPosition: { x:  3.5, y: 0, z: -2.5 } },
  ],
  ambientLightColor: 0xa0a0c0,
  pointLights: [
    { localPosition: { x: -2.5, y: 2.5, z:  0 }, color: 0xffe0a0, intensity: 0.8 },
    { localPosition: { x:  2.5, y: 2.5, z:  0 }, color: 0xffe0a0, intensity: 0.8 },
  ],
}
```

### 15.2 RoomBuilder

```typescript
// procgen/RoomBuilder.ts
import { Group, Mesh, Vector3, Quaternion } from 'three'
import type { AssetLoader } from '../assets/AssetLoader'

export class RoomBuilder {
  constructor(private assets: AssetLoader) {}

  async build(room: RoomInstance): Promise<Group> {
    const g = new Group()
    g.position.copy(room.worldPosition as any)
    g.rotation.y = room.worldRotation

    // Lay floor tiles in a 1m grid across the bbox.
    const tile = await this.assets.loadModel('models/dungeon-floor.glb')
    for (let x = room.template.bboxLocal.min.x; x < room.template.bboxLocal.max.x; x++) {
      for (let z = room.template.bboxLocal.min.z; z < room.template.bboxLocal.max.z; z++) {
        const t = tile.clone()
        t.position.set(x + 0.5, 0, z + 0.5)
        g.add(t)
      }
    }

    // Walls along the bbox perimeter, gapped at each door slot.
    this.addWallsWithDoors(g, room)

    // Lockers along the long walls (procedural placement, seeded).
    const locker = await this.assets.loadModel('models/locker.glb')
    this.placeLockersAlongWall(g, room, locker, 'N')
    this.placeLockersAlongWall(g, room, locker, 'S')

    // Lighting
    for (const light of room.template.pointLights) {
      addRoomLight(g, light.localPosition as Vec3, light.color, light.intensity)
    }

    return g
  }

  // … addWallsWithDoors, placeLockersAlongWall (procedural spacing 0.6 m between lockers,
  // skip locker positions that would overlap a door slot or an enemy spawn point)
}
```

### 15.3 Locker interaction

Each locker is a small interactable. The interaction is owned by the engine but uses the API's `Item` data via the `ContentBundle` passed in `loadRun`:

```typescript
// actors/Locker.ts
import { Entity } from './Entity'

export class Locker extends Entity {
  readonly kind = 'locker'
  locked: boolean
  contents: { itemSlug: string; quantity: number }[]

  // Player presses E within interact range
  onInteract(player: PlayerController) {
    if (this.locked) {
      // Try lockpick; if no lockpick, no-op (UI shows a tooltip)
      const lp = player.consumeOne('lock-pick')
      if (!lp) return
      this.locked = false
      this.engine!.audio.playOneShotAt('lockerOpen', this.position)
      return
    }
    for (const { itemSlug, quantity } of this.contents) {
      player.addToInventory(itemSlug, quantity)
      this.engine!.events.emit({ type: 'ItemPickedUp', itemSlug, quantity })
    }
    this.contents = []
  }
}
```

---

## 16. Spatial Audio

### 16.1 Architecture

Music is non-spatial and goes through Howler. Sfx that needs positional cues (enemy groans, footsteps, locker creaks, the Sweeper) uses Web Audio `PannerNode` directly. A central `AudioService` owns the `AudioContext` and the listener position (synced to the player camera each frame).

```typescript
// audio/AudioService.ts
import { Howl } from 'howler'
import { Vector3, Camera } from 'three'

export class AudioService {
  private ctx: AudioContext
  private master: GainNode
  private musicGain: GainNode
  private sfxGain: GainNode
  private sfxBuffers = new Map<string, AudioBuffer>()
  private musicHowls = new Map<string, Howl>()
  private listenerPos = new Vector3()

  constructor() {
    this.ctx = new (window.AudioContext || (window as any).webkitAudioContext)()
    this.master = this.ctx.createGain()
    this.musicGain = this.ctx.createGain()
    this.sfxGain = this.ctx.createGain()
    this.musicGain.connect(this.master)
    this.sfxGain.connect(this.master)
    this.master.connect(this.ctx.destination)
  }

  async loadSfx(slug: string, url: string): Promise<void> {
    const buf = await fetch(url).then(r => r.arrayBuffer())
    this.sfxBuffers.set(slug, await this.ctx.decodeAudioData(buf))
  }

  loadMusic(slug: string, url: string): void {
    this.musicHowls.set(slug, new Howl({ src: [url], loop: true, volume: 0.7 }))
  }

  playMusic(slug: string, fadeS = 1) { /* … crossfade if another track is playing … */ }

  playOneShot(slug: string, volume = 1) {
    const buf = this.sfxBuffers.get(slug); if (!buf) return
    const src = this.ctx.createBufferSource()
    const gain = this.ctx.createGain()
    gain.gain.value = volume
    src.buffer = buf
    src.connect(gain).connect(this.sfxGain)
    src.start()
  }

  playOneShotAt(slug: string, position: Vec3, volume = 1) {
    const buf = this.sfxBuffers.get(slug); if (!buf) return
    const src = this.ctx.createBufferSource()
    const panner = this.ctx.createPanner()
    panner.panningModel = 'HRTF'
    panner.distanceModel = 'inverse'
    panner.refDistance = 1
    panner.maxDistance = T.AUDIO_MAX_DISTANCE_M
    panner.rolloffFactor = 1
    panner.positionX.value = position.x
    panner.positionY.value = position.y
    panner.positionZ.value = position.z
    const gain = this.ctx.createGain()
    gain.gain.value = volume
    src.buffer = buf
    src.connect(panner).connect(gain).connect(this.sfxGain)
    src.start()
  }

  syncListener(camera: Camera) {
    camera.getWorldPosition(this.listenerPos)
    if (this.ctx.listener.positionX) {
      this.ctx.listener.positionX.value = this.listenerPos.x
      this.ctx.listener.positionY.value = this.listenerPos.y
      this.ctx.listener.positionZ.value = this.listenerPos.z
      // forward / up vectors derived from camera quaternion
      const forward = new Vector3(0, 0, -1).applyQuaternion(camera.quaternion)
      const up = new Vector3(0, 1, 0).applyQuaternion(camera.quaternion)
      this.ctx.listener.forwardX.value = forward.x
      this.ctx.listener.forwardY.value = forward.y
      this.ctx.listener.forwardZ.value = forward.z
      this.ctx.listener.upX.value = up.x
      this.ctx.listener.upY.value = up.y
      this.ctx.listener.upZ.value = up.z
    }
  }
}
```

### 16.2 Music transitions

- **Crossfade** between tracks over 1 s.
- **Duck** the music to 40% of its current volume whenever a loud sfx (Sweeper roar, weapon-broke chord) plays, then ramp back over 1.5 s.
- One-shot stingers (zone discovered, sweeper released) overlay the current track and don't duck it themselves.

### 16.3 Spatial mixing rules

| Sound class            | Spatial?    | Distance model        | Loops?   |
| ---------------------- | ----------- | --------------------- | -------- |
| Music tracks           | No          | n/a                   | Yes      |
| Pipe swing / hit       | Yes         | inverse, refDist 1    | No       |
| Enemy groan / scream   | Yes         | inverse, refDist 1    | No       |
| Sweeper footsteps loop | Yes         | inverse, refDist 2    | Yes      |
| Locker open / clock    | Yes         | inverse, refDist 1    | No       |
| HUD beeps / pickup     | No          | n/a                   | No       |
| Sweeper roar           | Yes (one-shot) | inverse, refDist 1 | No       |

---

## 17. Timer & Sweeper Release

The dungeon countdown is **hidden** by design (per PDF). The HUD never shows a timer. Players learn the time-of-day from in-room clocks (and at run start, from Jay's daily briefing — owned by the Angular shell).

The countdown value is **per-day, API-driven**: the C# API returns `sweeperReleaseSeconds` on the `RunDescriptor` based on Jay's morning briefing. The engine respects whatever the API hands it; it only falls back to `T.SWEEPER_RELEASE_DEFAULT_S` (90 s) if the API omits the field (e.g. local dev without the API).

```typescript
// engine/TimerService.ts
import { releaseSweeper } from './sweeper-release'
import { tuning as T } from '../tuning'

export class TimerService {
  private remaining: number
  private released = false

  constructor(initialRemainingSeconds: number = T.SWEEPER_RELEASE_DEFAULT_S) {
    this.remaining = initialRemainingSeconds
  }

  tick(dt: number, engine: GameEngine): void {
    if (this.released) return
    this.remaining -= dt
    if (this.remaining <= 0) {
      this.released = true
      releaseSweeper(engine)
    }
  }

  // Player finds a clock prop and presses E
  reveal(engine: GameEngine): void {
    engine.events.emit({ type: 'ClockFound', remainingSeconds: this.remaining })
  }
}
```

`GameEngine.loadRun(...)` constructs the `TimerService` with `new TimerService(run.sweeperReleaseSeconds ?? T.SWEEPER_RELEASE_DEFAULT_S)`. This keeps the PDF's "each day is different" flavor while letting MVP dev run without a live API.

---

## 18. Engine ↔ Angular Integration

The Angular side wraps the engine's typed event bus with a thin signal-based facade. The engine never imports Angular; the client does all the bridging.

### 18.1 Inputs the engine consumes

```typescript
// engine API: every frame the host calls engine.setInputs(...)
export interface InputState {
  forward: number          // -1..1
  right: number            // -1..1
  jump: boolean            // pressed this frame
  sprint: boolean          // held
  crouch: boolean          // toggle target (true = crouched)
  attack: boolean          // pressed this frame
  interact: boolean        // pressed this frame
  pause: boolean           // pressed this frame
}

engine.setInputs(snapshot)
engine.applyMouseDelta(dx, dy)    // separate because pointer-lock events fire async
```

The Angular dungeon page maps keyboard/mouse events to `InputState` via a tiny `InputBridge` service.

### 18.2 Events the engine emits

(Full union in §6.4.) Angular's `DungeonPage` subscribes to a curated subset and updates signal-backed services:

```typescript
// (in the Angular client — not the engine)
ngAfterViewInit() {
  this.engine = new GameEngine({ canvas: this.canvasRef.nativeElement })
  this.engine.events.on('EnemyKilled',  e => this.hud.recordKill(e))
  this.engine.events.on('PlayerHit',    e => this.hud.recordHit(e))
  this.engine.events.on('LowHealth',     () => this.hud.flashLowHealth())
  this.engine.events.on('SweeperReleased', () => this.hud.sweeperWarning())
  this.engine.events.on('PlayerDied',    e => this.runFlow.handleDeath(e))
  this.engine.events.on('RunCompleted',  e => this.runFlow.handleCompletion(e))
}
```

### 18.3 Pause semantics

`engine.pause()` freezes the tick (no physics step, no actor updates). The Angular shell shows the pause modal on top of the canvas. The pointer-lock is released on pause and re-acquired on resume. `engine.resume()` reverses everything.

**Audio policy on pause** (tuned for horror tension):

| Audio class                | On pause                                                  |
| -------------------------- | --------------------------------------------------------- |
| Music                      | Duck to -6 dB; keep playing                               |
| Sweeper proximity loop     | Duck to **30 %** of current gain; keep playing            |
| All other spatial sfx loops | Mute (fade out over 100 ms)                              |
| One-shots in flight        | Allowed to finish; new one-shots blocked                  |

The Sweeper loop staying audible is intentional: the pause modal is not a safe zone. The player still feels the thing closing in, which prevents pause-camping when it's nearby.

### 18.4 Cleanup contract

When the Angular `DungeonPage` is destroyed (navigation), it MUST call `engine.dispose()`. Failing to do so leaks the WebGL context, the AudioContext, and the Rapier WASM world. Disposal is idempotent.

---

## 19. Tuning Numbers + Formulas

All gameplay constants live in `src/tuning.ts`. Every value is paired with the formula or rationale that produced it so designers can adjust the curve, not just the value.

```typescript
// tuning.ts (excerpt — full file is ~120 lines)

/* ─────────────  Player base  ───────────── */
export const PLAYER_BASE_HP = 20                 // per PDF
export const PLAYER_HP_PER_LEVEL = 5             // per PDF: "increases by 5 for each level"
//  → maxHp(level) = PLAYER_BASE_HP + PLAYER_HP_PER_LEVEL * (level - 1)
export const PLAYER_HEIGHT_STANDING = 1.7        // m (average adult)
export const PLAYER_HEIGHT_CROUCHED = 0.85       // = standing / 2
export const PLAYER_RADIUS = 0.35                // capsule radius — generous for forgiving collisions
export const PLAYER_EYE_STANDING = 1.6           // standing eye height
export const PLAYER_EYE_CROUCH = 0.75            // crouched eye height
export const PLAYER_FOV_DEG = 75                 // vertical FOV, standard FPS

/* ─────────────  Player movement  ───────────── */
export const PLAYER_WALK_SPEED = 3.0             // m/s — brisk walk
export const PLAYER_RUN_SPEED  = 5.0             // m/s — runs 1.67× walk
export const PLAYER_CROUCH_SPEED = 1.5           // m/s — crouch is 0.5× walk
export const PLAYER_GROUND_ACCEL = 12            // 1/s exponent — reach 95% target velocity in ~0.25 s
export const PLAYER_AIR_ACCEL = 2                // sluggish in air — no strafe-jump abuse
export const GRAVITY = 18                        // m/s² — heavier than Earth for game feel
export const PLAYER_JUMP_VELOCITY = 5.4          // m/s
//  → jump height = v²/(2g) = 5.4²/(2·18) ≈ 0.81 m
//  → time to peak = v/g ≈ 0.3 s
export const PLAYER_IFRAMES_S = 0.5              // invulnerability window after a hit

/* ─────────────  Stamina  ───────────── */
export const STAMINA_MAX = 5.0                   // seconds of continuous sprint
export const STAMINA_REGEN_PER_S = 1.0           // regen rate when not sprinting
//  → full regen from empty takes STAMINA_MAX / STAMINA_REGEN_PER_S = 5 s

/* ─────────────  Pipe weapon  ───────────── */
export const PIPE_SWING_DAMAGE = 3               // per PDF (Pipe deals 3 hit damage)
export const PIPE_SWING_CYCLE_S = 0.5            // 2 swings/sec
export const PIPE_SWING_HIT_FRAME_S = 0.18       // hit detection fires 0.18 s into the swing anim
export const PIPE_REACH_M = 1.6                  // length of the swing arc + arm
export const PIPE_HIT_RADIUS_M = 0.25            // capsule cast radius — forgiving on near-misses
export const PIPE_DURABILITY = 20                // per PDF (Pipe has 20 health)
//  → durability_loss_per_hit is DATA-DRIVEN: the value comes from the Item record returned
//    by the API. MVP default below is used only if the Item record doesn't specify.
//    PDF text reads "two health off"; we default to 1 in MVP for less-punishing playtesting,
//    and let designers move it to 2 via the API without code changes.
export const PIPE_DURABILITY_LOSS_PER_HIT_DEFAULT = 1
//  → swings until break (at default) = PIPE_DURABILITY / 1 = 20

/* ─────────────  Hit feedback  ───────────── */
export const HIT_FLASH_S = 0.08                  // 80 ms red tint on enemies
export const HIT_KNOCKBACK_IMPULSE = 4           // impulse magnitude on hit
export const PLAYER_HIT_SHAKE_AMPL = 0.06        // camera shake amplitude on player hit
export const PLAYER_HIT_SHAKE_S = 0.15           // duration

/* ─────────────  Ragdolls  ───────────── */
export const RAGDOLL_TTL_S = 8                   // bodies despawn 8 s after death
export const RAGDOLL_INITIAL_IMPULSE = 6         // impulse along hit direction at death

/* ─────────────  Pipe Zombie  ───────────── */
export const PIPE_ZOMBIE_HP = 6                  // per PDF
export const PIPE_ZOMBIE_DMG = 3                 // per PDF
export const PIPE_ZOMBIE_SPEED = 2.5             // m/s — slower than player walk so kiting works
export const PIPE_ZOMBIE_REACH = 1.5             // m
export const PIPE_ZOMBIE_ATTACK_CYCLE = 1.0      // s — one hit per second when in reach
export const PIPE_ZOMBIE_DETECT = 8              // m — passive detection range
export const PIPE_ZOMBIE_AGGRO = 12              // m — once seen, stays aggro within this range

/* ─────────────  Mini Zombie  ───────────── */
export const MINI_ZOMBIE_HP = 3                  // per PDF
export const MINI_ZOMBIE_DMG = 2                 // per PDF
export const MINI_ZOMBIE_SPEED = 4.5             // m/s — almost as fast as player run, scary
export const MINI_ZOMBIE_REACH = 1.0
export const MINI_ZOMBIE_ATTACK_CYCLE = 0.8
export const MINI_ZOMBIE_DETECT = 10
export const MINI_ZOMBIE_AGGRO = 14

/* ─────────────  Sweeper Zombie  ───────────── */
export const SWEEPER_DMG = 5                     // ~25% of base HP per hit — terrifying
export const SWEEPER_SPEED = 4.0                 // slower than mini, faster than pipe
export const SWEEPER_REACH = 1.5
export const SWEEPER_ATTACK_CYCLE = 1.0
export const SWEEPER_RELEASE_DEFAULT_S = 90      // FALLBACK only. Per-day value is provided by the
                                                 // API via RunDescriptor.sweeperReleaseSeconds, per Jay's
                                                 // morning briefing (PDF). MVP default if API omits it.
export const SWEEPER_PROX_LOUD_RANGE = 12        // m — proximity loop hits max gain at distance 0
export const SWEEPER_MIN_WORLD_DIST_M = 8        // Sweeper spawn must be at least this far from the
                                                 // player in straight-line world distance, in addition
                                                 // to being the BFS-farthest room. See §12.2 / §14.5.

/* ─────────────  Procgen  ───────────── */
export const MIN_ROOMS = 5
export const MAX_ROOMS = 8
export const PLACEMENT_RETRIES = 8

/* ─────────────  Pathfinding  ───────────── */
export const PATH_REPLAN_DIST_M = 1.5
export const ENEMY_DETECT_DELAY_MS = 250

/* ─────────────  Audio  ───────────── */
export const AUDIO_MAX_DISTANCE_M = 30

/* ─────────────  Physics timestep (see §6.5)  ───────────── */
export const PHYSICS_FIXED_DT = 1 / 60               // s — Rapier world.timestep. Fixed for stable ragdolls.
export const PHYSICS_MAX_STEPS_PER_FRAME = 5         // cap to prevent spiral of death
//  → max simulated time per render frame = PHYSICS_FIXED_DT * PHYSICS_MAX_STEPS_PER_FRAME ≈ 83 ms

/* ─────────────  Physics groups (bit flags)  ───────────── */
export const PHYSICS_GROUP_WALL    = 0b0001
export const PHYSICS_GROUP_PLAYER  = 0b0010
export const PHYSICS_GROUP_ENEMY   = 0b0100
export const PHYSICS_GROUP_PICKUP  = 0b1000

/* ─────────────  Gold yields  ───────────── */
export const GOLD_PER_KIND: Record<string, number> = {
  'pipe-zombie': 5,   // per PDF
  'mini-zombie': 5,   // per PDF
}

export const tuning = { /* … re-exports the above as a single object for ergonomic access … */ }
```

### 19.1 Scaling formula (per the frontend doc proposal)

```
HP scaling enemies:   hp(level)     = baseHp + (level - 1)            // +1 HP per player level
Damage scaling:       dmg(level)    = baseDmg + floor((level - 1) / 3) // +1 dmg every 3 levels
Player max HP:        maxHp(level)  = 20 + 5*(level - 1)              // per PDF
```

The `scalesWithPlayerLevel` field on each enemy controls whether scaling applies (Pipe + Mini do not scale per PDF; Sword + Knight + Sweeper do, but those are out of MVP).

---

## 20. Testing Strategy & Sample Tests

Vitest is the test runner. The engine is pure TypeScript so tests run in plain Node + jsdom — no Angular harness.

### 20.1 What we test

| Module                | What we test                                                      |
| --------------------- | ----------------------------------------------------------------- |
| `procgen/prng`        | Determinism (same seed → same sequence)                           |
| `procgen/DungeonGenerator` | Determinism, room-count bounds, all rooms reachable from root |
| `actors/weapons/Pipe` | Damage application, durability decrement, cycle gating            |
| `actors/enemies/Enemy` (state machine) | Transitions Idle → Detect → Pursue → Attack, Stagger interrupts Attack |
| `ai/NavMeshService`   | Path between two reachable points has length > 0; no path between disconnected points returns null |
| `engine/EventBus`     | Listeners receive only their type; unsubscribe works              |
| `tuning`              | Sanity: every constant referenced by code is exported             |

### 20.2 Sample tests

```typescript
// tests/prng.test.ts
import { describe, it, expect } from 'vitest'
import { RunPrng } from '../src/procgen/prng'

describe('RunPrng', () => {
  it('is deterministic for the same seed', () => {
    const a = new RunPrng('seed-1')
    const b = new RunPrng('seed-1')
    for (let i = 0; i < 100; i++) expect(a.next()).toBeCloseTo(b.next(), 12)
  })
  it('produces different sequences for different seeds', () => {
    const a = new RunPrng('seed-1')
    const b = new RunPrng('seed-2')
    const sa = Array.from({ length: 20 }, () => a.next())
    const sb = Array.from({ length: 20 }, () => b.next())
    expect(sa).not.toEqual(sb)
  })
  it('range(min, max) is bounded', () => {
    const p = new RunPrng('x')
    for (let i = 0; i < 1000; i++) {
      const v = p.range(5, 10)
      expect(v).toBeGreaterThanOrEqual(5)
      expect(v).toBeLessThan(10)
    }
  })
})
```

```typescript
// tests/dungeon-generator.test.ts
import { describe, it, expect } from 'vitest'
import { DungeonGenerator } from '../src/procgen/DungeonGenerator'
import { RunPrng } from '../src/procgen/prng'
import { tuning as T } from '../src/tuning'

describe('DungeonGenerator', () => {
  it('produces the same graph for the same seed', () => {
    const a = new DungeonGenerator(new RunPrng('seed-1')).generate()
    const b = new DungeonGenerator(new RunPrng('seed-1')).generate()
    expect(a.rooms.map(r => r.template.slug)).toEqual(b.rooms.map(r => r.template.slug))
    expect(a.edges).toEqual(b.edges)
    expect(a.exitIndex).toBe(b.exitIndex)
  })

  it('respects min/max room counts', () => {
    for (const seed of ['a', 'b', 'c', 'd', 'e']) {
      const g = new DungeonGenerator(new RunPrng(seed)).generate()
      expect(g.rooms.length).toBeGreaterThanOrEqual(T.MIN_ROOMS)
      expect(g.rooms.length).toBeLessThanOrEqual(T.MAX_ROOMS)
    }
  })

  it('every room is reachable from the root', () => {
    const g = new DungeonGenerator(new RunPrng('connectivity')).generate()
    const visited = new Set<number>([g.rootIndex])
    const queue = [g.rootIndex]
    while (queue.length) {
      const node = queue.shift()!
      for (const [a, b] of g.edges) {
        if (a === node && !visited.has(b)) { visited.add(b); queue.push(b) }
        if (b === node && !visited.has(a)) { visited.add(a); queue.push(a) }
      }
    }
    expect(visited.size).toBe(g.rooms.length)
  })
})
```

```typescript
// tests/state-machine.test.ts
import { describe, it, expect, vi } from 'vitest'
import { StateMachine } from '../src/ai/StateMachine'

describe('StateMachine', () => {
  it('fires onExit and onEnter on transition', () => {
    const fsm = new StateMachine<'A' | 'B'>('A')
    const exitA = vi.fn()
    const enterB = vi.fn()
    fsm.onExit.set('A', exitA)
    fsm.onEnter.set('B', enterB)
    fsm.transition('B')
    expect(exitA).toHaveBeenCalled()
    expect(enterB).toHaveBeenCalled()
    expect(fsm.current).toBe('B')
  })
  it('is a no-op if transitioning to the current state', () => {
    const fsm = new StateMachine<'A' | 'B'>('A')
    const exitA = vi.fn()
    fsm.onExit.set('A', exitA)
    fsm.transition('A')
    expect(exitA).not.toHaveBeenCalled()
  })
})
```

```typescript
// tests/pipe.test.ts
import { describe, it, expect, vi } from 'vitest'
import { Pipe } from '../src/actors/weapons/Pipe'
import { tuning as T } from '../src/tuning'

describe('Pipe weapon', () => {
  it('does not fire faster than its cycle', () => {
    const pipe = new Pipe()
    const engine = makeFakeEngine({ now: 0 })
    pipe.use(engine, engine.player, 0)
    engine.now = T.PIPE_SWING_CYCLE_S * 0.5
    const before = pipe.currentDurability
    pipe.use(engine, engine.player, 0)
    expect(pipe.currentDurability).toBe(before)            // no second swing yet
    engine.now = T.PIPE_SWING_CYCLE_S * 1.01
    pipe.use(engine, engine.player, 0)
    expect(pipe.currentDurability).toBe(before - T.PIPE_DURABILITY_LOSS_PER_HIT)
  })
  it('emits WeaponBroke when durability reaches zero', () => {
    const pipe = new Pipe()
    const engine = makeFakeEngine({ now: 0 })
    const spy = vi.spyOn(engine.events, 'emit')
    for (let i = 0; i < T.PIPE_DURABILITY + 1; i++) {
      engine.now = i * (T.PIPE_SWING_CYCLE_S + 0.01)
      pipe.use(engine, engine.player, 0)
    }
    expect(spy).toHaveBeenCalledWith(expect.objectContaining({ type: 'WeaponBroke' }))
  })
})

// makeFakeEngine is a small test helper that returns an engine-shaped object
// with stubbed physics.castShape, events bus, player, etc.
```

### 20.3 What we don't test

- Three.js render output — we don't snapshot rendered pixels. Visual correctness is a manual QA bar.
- Rapier physics step output — we trust the library.
- Frame-rate / performance — that's a manual perf check in §22.

---

## 21. Build, Release, and Client Integration

### 21.1 Engine build (Vite library mode)

```typescript
// vite.config.ts in the engine repo
import { defineConfig } from 'vite'
import { resolve } from 'node:path'
import dts from 'vite-plugin-dts'

export default defineConfig({
  plugins: [dts({ insertTypesEntry: true })],
  build: {
    lib: {
      entry: resolve(__dirname, 'src/index.ts'),
      name: 'LivingLandEngine',
      formats: ['es'],
      fileName: 'index',
    },
    rollupOptions: {
      external: ['three', '@dimforge/rapier3d-compat', '@recast-navigation/three',
                 '@recast-navigation/core', 'miniplex', 'howler', 'alea'],
    },
    target: 'es2022',
    sourcemap: true,
  },
})
```

### 21.2 package.json (engine)

```jsonc
{
  "name": "@rearles/livingland-engine",
  "version": "0.1.0",
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": { "import": "./dist/index.js", "types": "./dist/index.d.ts" }
  },
  "files": ["dist"],
  "scripts": {
    "build": "vite build",
    "test":  "vitest run",
    "lint":  "eslint src --ext .ts"
  },
  "peerDependencies": {
    "three":                       "^0.160.0",
    "@dimforge/rapier3d-compat":   "^0.13.0",
    "@recast-navigation/three":    "^0.30.0",
    "@recast-navigation/core":     "^0.30.0",
    "miniplex":                    "^2.0.0",
    "howler":                      "^2.2.0",
    "alea":                        "^1.0.0"
  }
}
```

### 21.3 Release

- Manual or CI: `npm version <patch|minor|major>` → `git push --tags`.
- A GitHub Action triggered by `v*` tags runs build + tests + `npm publish --access public`.
- Pre-MVP we can `npm publish --access restricted` if we don't want the package public yet, or use a GitHub Packages registry scoped to `@rearles`.

### 21.4 Client integration

In the Angular client:

```jsonc
// package.json
{
  "dependencies": {
    "@rearles/livingland-engine": "^0.1.0",
    "three": "^0.160.0",
    "@dimforge/rapier3d-compat": "^0.13.0",
    "@recast-navigation/three": "^0.30.0",
    "@recast-navigation/core": "^0.30.0",
    "miniplex": "^2.0.0",
    "howler": "^2.2.0",
    "alea": "^1.0.0"
  }
}
```

For **local development** (engine + client checked out side by side):

```bash
# in the engine repo
cd ~/dev/livingland-engine
npm run build && npm link

# in the client repo
cd ~/dev/TheLivingLand
npm link @rearles/livingland-engine
```

Changes to engine source need a `npm run build` (or `vite build --watch`) before the client picks them up.

---

## 22. Performance Budget

Targets for the MVP slice (Locker Room, 5–8 rooms, ≤6 enemies on screen):

| Metric                          | Budget                | Notes                                                  |
| ------------------------------- | --------------------- | ------------------------------------------------------ |
| Frame rate                      | 60 fps target         | Acceptable down to 30 fps on a 2018 MacBook Air        |
| Frame time                      | ≤ 16.7 ms             |                                                        |
| Render: draw calls              | ≤ 200                 | Use instanced meshes for repeating room geometry       |
| Render: triangles on screen     | ≤ 30k                 |                                                        |
| Physics: bodies                 | ≤ 80                  | Includes ragdolls (9 bodies each)                      |
| Audio: simultaneous voices      | ≤ 32                  | Hard cap; oldest one-shots get culled                  |
| Memory: heap                    | ≤ 250 MB              | Including Three.js + Rapier + decoded audio buffers    |
| Memory: GPU                     | ≤ 200 MB              | Textures + render targets                              |
| Bundle (engine ESM, gzip)       | ≤ 80 KB               | Excluding peer-deps (Three, Rapier are loaded by host) |

### 22.1 Where time goes (estimated)

| Subsystem                       | Per-frame budget        |
| ------------------------------- | ----------------------- |
| Input + AI updates              | ≤ 2 ms                  |
| Physics step (Rapier)           | ≤ 3 ms                  |
| ECS systems (particles, audio)  | ≤ 1 ms                  |
| Rendering (3D pass + upscale)   | ≤ 8 ms                  |
| Browser overhead + slack        | ≤ 2.7 ms                |

### 22.2 What we measure and how

- A debug overlay (`F3`) shows live frame time, draw calls, triangles, physics step ms, active actors, navmesh path queries / sec.
- Vitest doesn't measure perf; manual profiling uses Chrome DevTools Performance panel.
- A scripted "stress run" loads a synthetic 8-room dungeon with 12 enemies and asserts average frame time across 60 s.

---

## 23. MVP Acceptance Checklist

A run is "MVP-acceptable" when:

- [ ] The player can log in, equip the Pipe at the shop, and enter the dungeon from the level-select page.
- [ ] The dungeon contains 5–8 Locker Rooms connected by doors per the branching-tree generator.
- [ ] Pipe Zombies and Mini Zombies spawn in rooms per the template's spawn weights.
- [ ] WASD + mouse-look movement feels Half-Life-ish; jump + crouch + sprint all work.
- [ ] Hitting an enemy with the pipe deals damage, plays the hit sfx + flash, knocks them back.
- [ ] Enemies die into a ragdoll that physically interacts with the environment and despawns after 8 s.
- [ ] Enemies pursue the player using the navmesh; they don't get stuck on walls or doors.
- [ ] 90 s into the run, the Sweeper Zombie is released with a roar and audible footstep loop that intensifies as it gets close.
- [ ] The Sweeper kills the player if it reaches them; the player cannot kill the Sweeper.
- [ ] The player can hear positional audio cues (enemy groans, Sweeper proximity) through `PannerNode`.
- [ ] The HUD shows current HP, gold, score with multiplier, equipped weapon + durability, and a stamina bar.
- [ ] On player death or successful escape (returning to the root room exit), the engine emits `RunCompleted` and the Angular shell navigates to the summary page.
- [ ] The full run runs at ≥ 30 fps on a mid-2018 MacBook Air.
- [ ] No console errors during a normal run.
- [ ] `npm test` in the engine repo passes.
- [ ] The engine package builds and publishes; the client consumes it via `npm install`.

---

## 24. Resolved Decisions

The questions that surfaced during drafting were resolved in review with Ryan on 2026-05-22. Each entry notes the decision and where it is implemented in the doc.

1. **Pipe durability per hit** — *Data-driven, default 1, API can override.* The `Item` record returned by the API carries the per-weapon `durability_loss_per_hit` value. The Pipe class reads it at construction time; `T.PIPE_DURABILITY_LOSS_PER_HIT_DEFAULT` (1) is only used if the API omits it. Implemented in §10.2 (Pipe class) and §19 (tuning).
2. **Sweeper release timing** — *Per-day, API-driven.* `RunDescriptor.sweeperReleaseSeconds` from the API is the source of truth; `T.SWEEPER_RELEASE_DEFAULT_S` (90 s) is the fallback if the API omits the field. Implemented in §17 (TimerService constructor) and §19 (tuning).
3. **Ragdoll constraint configuration** — *Per-enemy configs.* `PIPE_ZOMBIE_RAGDOLL` (9 bones, segmented limbs), `MINI_ZOMBIE_RAGDOLL` (5 bones, single-segment limbs), `SWEEPER_RAGDOLL` (11 bones, head + neck + bulkier radii). Looked up at death time via `RAGDOLLS_BY_KIND[kind]`. Implemented in §10.4.
4. **Pause behavior under Sweeper proximity** — *Duck the Sweeper loop to 30 %, mute all other spatial sfx loops, keep music at -6 dB.* The Sweeper loop staying audible is intentional — pause is not a safe zone. Implemented in §18.3.
5. **Mid-run save** — *Confirmed off.* Per PDF spirit, a run is a single push. Closing the tab forfeits the run after the API's 15-minute heartbeat timeout. Implemented in §1.3 (non-goals).
6. **Stamina UX** — *Always visible on the HUD.* Matches the PDF's HUD sketch (page 6). No fade or context-aware hiding for MVP. Implemented in the frontend doc's HUD components.
7. **First-person hands asset fallback** — *MVP ships with weapon-only if the asset pack doesn't provide the four per-weapon poses (`idle`, `swing-windup`, `swing-hit`, `hide`).* Settings toggle becomes a no-op for MVP in that case; authoring the poses in Blender is a v0.2 task. Implemented in §9.4.
8. **Sweeper spawn metric** — *BFS-farthest AND at least 8 m world-space distance from the player.* If no room clears both, fall back to pure BFS so the Sweeper still spawns somewhere. Implemented in §12.2 and §14.5; `T.SWEEPER_MIN_WORLD_DIST_M` in §19.
