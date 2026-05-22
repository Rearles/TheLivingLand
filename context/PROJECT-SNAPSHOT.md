# The Living Land — Project Snapshot

> **Living document.** Update this file at the end of each session to reflect decisions made, work completed, and what's next. Individual turn files in `context/` are the historical archive; this file is what Claude Code reads to orient itself at session start.

Last updated: turn-014 (2026-05-22)

---

## What this project is

A passion + learning project: can Claude Code build a first-person 3D dungeon crawler entirely from a design document? Ryan Earles is the solo developer/designer; Claude Code handles all 3D wiring, API scaffolding, and Angular client code. The source design document is `The Living Land.pdf` (Bolder Games concept doc) — it was provided as a chat attachment in early sessions and is not committed to the repo.

---

## MVP scope (vertical slice)

Register/login → Shop (one weapon: Pipe) → one zone (Dungeon) → one room template (Locker Room) → two enemy types (Pipe Zombie + Mini Zombie) → Sweeper release → die-or-escape → summary screen.

**Done bar:** "Playable for fun" — a 5–10 min run feels like a real game. Tension and satisfaction present; sound tuned; HUD readable.

---

## Visual style

- **Rendering:** 640×360 internal render target, nearest-neighbor upscale, nearest-filtered textures, modern lighting model
- **Art direction:** pixel-textured first-person 3D — 3D geometry with low-res pixel-art textures (chunky retro feel)
- **Assets:** CC0 free packs (Quaternius modular dungeon + zombies, Kenney mini dungeon kit) plus procedural primitives where no pack asset fits. No bespoke Blender modeling by Ryan.

---

## Tech stack

### Backend
- **Runtime:** ASP.NET Core 8 + EF Core + SQL Server
- **Auth:** JWT + refresh cookies; three roles: anonymous, player, admin
- **API style:** CRUD content API (no server-side combat resolution), standard REST with pagination/filtering/sorting
- **Conventions:** RFC 7807 `problem+json` errors, FluentValidation, OpenAPI/Swagger

### Frontend / engine
- **UI framework:** Angular 17+ standalone components, signals-based state, Tailwind CSS
- **Renderer:** Three.js (renderer only, not an engine)
- **Physics:** Rapier (WASM)
- **Audio:** Web Audio API PannerNode
- **Package split:** The engine lives in a **separate npm package** (`@rearles/livingland-engine`, its own GitHub repo) and is published to npm. The Angular client consumes it via `npm install`. This is the agreed architecture — honor it during scaffolding.

---

## Domain model (from api-design.md)

Entities: `Zone`, `Level`, `Room`, `Enemy`, `Boss`, `Obstacle`, `Item`, `MusicTrack`, `Player`, `InventoryItem`, `DungeonRun`.

Item records drive gameplay data (e.g., `Item.durabilityLossPerHit` — MVP default 1 hit; live value data-driven from the API). Do not hard-code tuning constants that belong in the API.

---

## 3D engine architecture decisions

### Actor/ECS/Services hybrid
Three first-class categories — this is not optional, it is the settled design:
- **Actors** — OOP objects for individuals (`Player`, `Enemy`, `Sweeper`). Tick themselves. Created/destroyed explicitly.
- **ECS components** — for swarms: projectiles, particles, spatial audio sources. Managed by an ECS world.
- **Services** — long-lived infrastructure: `PhysicsWorld`, `AudioService`, `NavMeshService`, `Scheduler`, `AssetLoader`. Created in `GameEngine.init()`, disposed in `dispose()`.

### `GameEngine.tick()` order (do not reorder)
1. Document inputs — push-from-host: Angular calls `engine.setInputs()` / `engine.applyMouseDelta()` per frame; the engine never polls DOM.
2. Update ECS world
3. Tick all actors
4. Physics accumulator — accumulate variable render `dt`, step Rapier at `PHYSICS_FIXED_DT` (1/60 s) up to `PHYSICS_MAX_STEPS_PER_FRAME` (5) per render frame. Excess accumulator dropped (no spiral of death).
5. Sync dynamic bodies — walk `dynamicBodies: Map<number, { body, mesh }>`, copy Rapier quaternion/translation → THREE mesh position/quaternion.
6. Render

### Camera ownership
`PlayerController.onAttach()` sets `engine.activeCamera = this.camera`. The renderer reads from `engine.activeCamera`.

### Dynamic-body registry
`registerDynamicBody(body, mesh)` / `unregisterDynamicBody(handle)` are public methods on `GameEngine`. All ragdoll bones and physics-driven meshes go through this registry.

### Movement
Half-Life-ish smooth accel/decel. Walk ~3 m/s, run ~5 m/s, no air control. Jump (Space, ~0.5 m hop), crouch (LCtrl, half-height capsule). Both in MVP.

### Pathfinding
Full navmesh via `recast-navigation-js`. Enemies and Sweeper pathfind through it.

### Sweeper
- Pathfinds through the room graph; always knows player's room
- Release timing: `RunDescriptor.sweeperReleaseSeconds` (API-driven); default 90 s if API omits field
- Spawn placement: BFS-farthest room AND ≥ 8 m world distance (`SWEEPER_MIN_WORLD_DIST_M = 8`); fall back to pure BFS-farthest if no room clears the floor

### Procedural generation
Branching tree per PDF spec — 1–4 doors per room, tree/DAG layout (not a linear chain).

### Audio
- Full 3D spatial audio via Web Audio API `PannerNode` with distance falloff
- Music ducks under SFX
- **Pause audio policy:** music −6 dB, Sweeper proximity loop duck to 30 %, other spatial loops mute, one-shots play to completion

### Combat / ragdolls
- Enemy death triggers full physics ragdoll via Rapier
- Per-enemy ragdoll configs (do not use a single generic config):
  - `PIPE_ZOMBIE_RAGDOLL` — 9 bones
  - `MINI_ZOMBIE_RAGDOLL` — 5 bones (single-segment limbs)
  - `SWEEPER_RAGDOLL` — 11 bones (separate head + neck)
- **Pipe durability:** data-driven from `Item` record; `PIPE_DURABILITY_DEFAULT = 1` is the fallback constant only

### HUD
Angular DOM + Tailwind. Pixel-art icons, `image-rendering: pixelated`, bitmap-feel webfont. Stamina bar always visible. Naturally accessible.

### First-person hands
Visible hands + weapon by default (toggle in settings to hide). MVP ships weapon-only if the chosen asset pack does not include per-weapon FP poses.

### Misc settled decisions
- **Determinism:** loose — procgen layout is seeded and reproducible; per-frame randomness uses `Math.random`. No replays.
- **Mid-run save:** off.
- **Tuning constants live in `tuning.ts`:** `PHYSICS_FIXED_DT = 1/60`, `PHYSICS_MAX_STEPS_PER_FRAME = 5`.

### Deferred (resolve during §10 review)
- Ragdoll-bone ownership: actor (one `Ragdoll` actor per dead enemy) vs ECS (one `RagdollBone` component per bone)
- Audio-loop ownership for Sweeper footsteps: actor vs ECS world

---

## Repository & workflow

### Branching model (GitFlow-style)
| Branch | Role |
|--------|------|
| `master` | Production / released |
| `develop` | Integration default branch |
| `story/*` `feature/*` `fix/*` | Short-lived; branch off `develop`, PR back to `develop` |
| `hotfix/*` | Branch off `master`, merge back to `master` + `develop` |

Releases: merge-and-tag `develop` → `master`.

### Branch protection (applied to both `master` and `develop`)
- PR required before merging
- 0 required approving reviewers (solo project; self-merge after review is fine)
- No force pushes, no deletions
- Required linear history
- Required conversation resolution
- `enforce_admins: false` (Ryan can override in emergencies)

### Tool paths & auth
- `gh` CLI installed at `~/.local/bin/gh` (added to PATH in `~/.bash_profile`)
- GitHub auth: `Rearles` account, keyring, HTTPS credential helper (`gh auth setup-git`)
- System: macOS Monterey — CLT is **outdated**. Avoid `brew install` for formulas that compile from source. Binary downloads work fine.

### Context recording rule
Every conversation turn must produce a `context/archive/turn-NNN-<short-title>.md` file before the session ends. Check the highest existing number inside `context/archive/`. Auto-approved in `.claude/settings.local.json` (git-ignored). **Also update this snapshot file** to reflect the session's decisions and current state.

---

## What's been built

| File | Status |
|------|--------|
| `docs/api-design.md` | Complete — REST API design |
| `docs/frontend-design.md` | Complete — Angular frontend design (updated to reflect 3D + pixel-texture) |
| `docs/3d-engine-design.md` | Complete — 2 511-line 3D engine design; reviewed in turns 010–011 |
| `CONTRIBUTING.md` | Complete — branching and workflow docs |
| `context/archive/turn-001` through `turn-014` | Historical turn log (all 14 turns archived) |
| `CLAUDE.md` | Auto-read instructions for Claude Code at session start |
| `context/PROJECT-SNAPSHOT.md` | This file — authoritative session-start summary |

---

## Current state (turn-014, 2026-05-22)

- **Current branch:** `develop` (clean, up to date with origin)
- **No open PRs**
- **develop history:**
  - `d880ff1` — Consolidate session history: CLAUDE.md, PROJECT-SNAPSHOT, archive turn logs (PR #2)
  - `f7ea3f8` — Add 3D engine design doc (PR #1)
  - `d177d42` — initial pass with design folder and context folder
- All design doc work is merged to develop. Three design docs are complete and on develop.

---

## What's next

1. **Scaffold implementation** — the next major body of work. Three repos to set up:
   - `@rearles/livingland-engine` — standalone npm package (its own GitHub repo)
   - ASP.NET Core 8 API project (in this repo, likely under `api/`)
   - Angular 17+ client project (in this repo, likely under `client/`)
2. **Wire up Tier 1 security scanning** (Dependabot alerts + version updates, CodeQL, secret scanning / push protection) when first code files land
3. **Create engine GitHub repo** (`@rearles/livingland-engine`) before writing engine code
4. **CLT update** (non-blocking): `sudo rm -rf /Library/Developer/CommandLineTools && sudo xcode-select --install`

---

## Key files at a glance

- [docs/api-design.md](../docs/api-design.md) — REST API design (authoritative)
- [docs/frontend-design.md](../docs/frontend-design.md) — Angular frontend design (authoritative)
- [docs/3d-engine-design.md](../docs/3d-engine-design.md) — 3D engine design (authoritative; 2 511 lines)
- [CONTRIBUTING.md](../CONTRIBUTING.md) — git workflow
- [context/](.) — turn-by-turn archive
- [memory/](../memory/) — persistent Claude Code memory (see MEMORY.md)
