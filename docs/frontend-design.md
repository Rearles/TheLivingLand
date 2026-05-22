# The Living Land — Angular Frontend Design

**Status:** Draft v1
**Owner:** Bolder Games, LLC
**Stack:** Angular 17+ (standalone components, signals), TypeScript 5, Three.js, Howler.js, Tailwind CSS
**Companion doc:** [api-design.md](./api-design.md)

---

## 1. Overview

The Angular client is the playable game. A player visits the site, signs in, and runs the dungeon crawler directly in the browser: a first-person, pixel-textured 3D view of Gathar walking through procedurally arranged rooms, swinging melee weapons, shooting firearms, picking up loot, and dying to (or escaping from) the Sweeper Zombie.

Angular owns the **shell** — auth, routing, the shop, the level select, the HUD, the upgrades screen, the gameplay-summary screen, settings. The in-dungeon gameplay scene is rendered with **Three.js** running inside a dedicated Angular component. The two cooperate cleanly: Angular owns navigation, forms, and persistent state; Three.js owns the per-frame render loop and the WebGL canvas. The deep technical specification for the 3D engine, asset pipeline, procgen, AI, and combat lives in a companion document: [3d-engine-design.md](./3d-engine-design.md).

### 1.1 MVP scope (vertical slice)

The first build is a thin end-to-end slice that proves the whole stack. Everything outside this list is deferred. The full game described in the PDF is the v1.0 target; this MVP is v0.1.

- Register / login.
- One zone unlocked: the Dungeon. One level inside it.
- Shop with a single weapon (Pipe) and Health Potions.
- One room template (Locker Room), 3–5 instances stitched per run.
- Two enemy types: Pipe Zombie and Mini Zombie.
- One obstacle: Locked Box.
- Player melee combat with the Pipe.
- Hidden Sweeper-release timer; Sweeper spawns and pursues when it expires.
- Run lifecycle: start → play → die or escape → summary.
- Server-persisted player state (gold, level, inventory, run history).

Subsequent slices add: more room templates → more enemies/weapons → firearms → all obstacles → bosses → mothership → world/wiki pages → admin CMS.

### 1.2 Goals

- Deliver every piece of player-facing functionality the design doc calls out: HUD, controls, dungeon, shop, level select, settings, gameplay summary, lore/wiki browsing.
- Cleanly separate "game scene" (Three.js, fast inner loop) from "UI shell" (Angular, slower outer loop) so each can evolve independently.
- Be testable: components are pure where possible, services are mockable, the game scene is decoupled from the API behind a `GameSessionService`.
- Be navigable to a non-player: the world/lore pages (zones, rooms, enemies, bosses, obstacles) double as a public wiki for the game.

### 1.3 Non-Goals (v1)

- Mobile/touch input. Desktop keyboard + mouse only. Mobile layout for non-gameplay pages is fine; the dungeon itself is desktop-only in v1.
- Console/gamepad input.
- Offline play. The client is a thin layer over the API; everything requires connectivity.
- Multiplayer rendering. Single-player only.

---

## 2. Tech Stack & Conventions

| Concern              | Choice                                             | Why                                                                 |
| -------------------- | -------------------------------------------------- | ------------------------------------------------------------------- |
| Framework            | Angular 17+ (standalone components, signals)       | Modern Angular; no NgModules; reactive primitives                   |
| Language             | TypeScript 5, `strict: true`                       |                                                                     |
| Rendering            | Three.js (r160+) for the dungeon scene             | Largest ecosystem and example corpus, lightweight, explicit control over the render loop. See [3d-engine-design.md](./3d-engine-design.md) for why Three.js over Babylon.js. |
| Physics              | Rapier (`@dimforge/rapier3d-compat`)               | Modern Rust/WASM physics; fast, deterministic, actively maintained. |
| Audio                | Howler.js                                          | Sprite-friendly, handles browser autoplay quirks                    |
| Styling              | Tailwind CSS                                       | Faster than Material for the non-game UI; lets us match game aesthetic |
| State                | Angular signals + a small `ComponentStore` per feature where needed | Avoid full NgRx — overkill for this scope                  |
| HTTP                 | `HttpClient` + interceptors (auth, error)          |                                                                     |
| API client           | Generated from server OpenAPI via `ng-openapi-gen` | Types stay in sync with the C# API                                  |
| Routing              | Angular Router with route-level lazy-loading       |                                                                     |
| Testing              | Jest (unit) + Playwright (e2e)                     | Karma is legacy; Jest is the modern default                         |
| Build                | Angular CLI w/ esbuild (default in v17)            |                                                                     |

### 2.1 Conventions

- Standalone components only. No `NgModule`s except the root bootstrap.
- File naming: `feature-name.component.ts`, `feature-name.service.ts`, `feature-name.model.ts`.
- Signals for component state. RxJS only where genuinely async-streaming (HTTP, websockets-if-we-add-them).
- One smart container per route, dumb presentational components beneath it.
- All API access goes through generated clients in `src/api/` — never raw `HttpClient` calls from components.

---

## 3. Architecture

```
src/
  app/
    app.config.ts                 # providers, router, interceptors
    app.routes.ts                 # top-level routes (lazy)
    core/                         # auth, http interceptors, error handler
    shared/                       # ui primitives (Button, Modal, HealthBar, etc.)
    features/
      auth/                       # login, register, account
      home/                       # landing page
      shop/                       # Zone 1 — Jay Longfury
      level-select/               # pick a zone/level
      dungeon/                    # Zone 2 — the playable game scene
        dungeon.page.ts           # Angular wrapper
        engine/                   # Three.js game engine code (TS, not Angular)
          scene.ts
          player-controller.ts
          enemy-ai.ts
          weapons.ts
          obstacles.ts
          procedural-generator.ts
          combat.ts
          input.ts
        hud/                      # HUD overlay components
      mothership/                 # Zone 3 — same engine, special arena + boss
      summary/                    # post-run summary screen
      profile/                    # player career, stats, inventory
      world/                      # public lore/wiki — zones, rooms, enemies, bosses, obstacles
      settings/                   # audio/video/controls
  api/                            # generated client (do not hand-edit)
  assets/
    models/                       # .glb 3D models
    textures/
    audio/                        # tracks + sfx
    sprites/                      # 2D UI art
```

### 3.1 Layered separation

The dungeon **engine** under `features/dungeon/engine/` is plain TypeScript with no Angular dependencies. It exposes a `GameEngine` class with methods like `init(canvas)`, `loadRun(seed, content)`, `tick(dt)`, `pause()`, `dispose()`, plus an `Observable<GameEvent>` stream for things Angular needs to react to (`enemyKilled`, `playerDied`, `roomEntered`, `lowHealth`, `runComplete`).

The Angular `DungeonPage` component:
1. Owns the `<canvas>` element.
2. Instantiates `GameEngine` in `ngAfterViewInit`.
3. Feeds it content (loaded from the API).
4. Subscribes to `GameEvent`s and updates the HUD, triggers heartbeats, etc.
5. Tears down on navigation.

This split keeps the inner loop free of change detection overhead.

---

## 4. Routing

All feature routes are lazy-loaded. Auth-guarded routes use a `playerGuard`; admin tools use `adminGuard`.

| Path                          | Component                  | Auth     | Notes                                  |
| ----------------------------- | -------------------------- | -------- | -------------------------------------- |
| `/`                           | `HomePage`                 | anon     | Landing, lore intro, "play" CTA        |
| `/login`                      | `LoginPage`                | anon     |                                        |
| `/register`                   | `RegisterPage`             | anon     |                                        |
| `/world`                      | `WorldOverviewPage`        | anon     | The three zones, links to detail pages |
| `/world/zones/:slug`          | `ZoneDetailPage`           | anon     |                                        |
| `/world/rooms`                | `RoomListPage`             | anon     | All 10 room types                      |
| `/world/rooms/:slug`          | `RoomDetailPage`           | anon     |                                        |
| `/world/enemies`              | `EnemyListPage`            | anon     |                                        |
| `/world/enemies/:slug`        | `EnemyDetailPage`          | anon     | Stats, behavior, image                 |
| `/world/bosses/:slug`         | `BossDetailPage`           | anon     |                                        |
| `/world/obstacles/:slug`      | `ObstacleDetailPage`       | anon     |                                        |
| `/play`                       | `LevelSelectPage`          | player   | Pick zone/level                        |
| `/play/shop`                  | `ShopPage`                 | player   | Jay Longfury                           |
| `/play/dungeon`               | `DungeonPage`              | player   | The actual game                        |
| `/play/mothership`            | `MothershipPage`           | player   | Unlocks at the right XP level          |
| `/play/summary/:runId`        | `RunSummaryPage`           | player   | Post-run results                       |
| `/profile`                    | `ProfilePage`              | player   | Stats, inventory, upgrades             |
| `/settings`                   | `SettingsPage`             | player   |                                        |
| `/admin/**`                   | (admin feature)            | admin    | CMS for content (out of scope v1 docs) |

---

## 5. Feature Breakdowns

### 5.1 Auth (`features/auth/`)

- `LoginPage` — email + password form. On success stores JWT in memory and triggers a refresh-token cookie set by the API.
- `RegisterPage` — email + password + `characterName`. Same flow as login.
- `AuthService` — `login()`, `logout()`, `register()`, `currentUser` signal, `isAuthenticated` computed signal.
- `AuthInterceptor` — attaches `Authorization: Bearer <token>` and handles 401 by triggering token refresh once before bailing to `/login`.

### 5.2 World / lore browser (`features/world/`)

Pure read-only pages backed by the public content endpoints. Each page is a smart container that fetches from the API and renders a presentational component.

- `WorldOverviewPage` — three cards, one per zone, each with description from the API.
- `ZoneDetailPage` — zone description + list of its levels.
- `EnemyListPage` — grid of all enemies with key stats (HP, points, gold, damage, speed).
- `EnemyDetailPage` — full card: HP, point value, gold value, hit damage, speed, scaling behavior, kind (Standard/Boss/Sweeper). For bosses, also intro audio cue and behavior text.
- `RoomListPage` / `RoomDetailPage` — visual gallery + lore text per room template.
- `ObstacleDetailPage` — name, damage, whether disarmable, tools that can disarm it.

These pages give us a wiki that doubles as marketing for the game, indexed by search engines.

### 5.3 Shop (`features/shop/`) — Zone 1

This implements Jay Longfury's stall.

- `ShopPage` — three tabs: **Weapons**, **Potions**, **Misc**. Within each tab, items are cards showing name, icon, stats, and price. A card is disabled if the player can't afford it.
- Player gold balance visible top-right at all times.
- Add-to-cart pattern: items go into a basket on the right side; "Checkout" calls `POST /shop/purchase` atomically.
- `ShopService` — wraps the generated API client; exposes `inventory$`, `purchase(items)`.

### 5.4 Level select (`features/level-select/`)

- Three zone tiles. Inside each, five level tiles. Locked levels show a padlock and "requires level N".
- Selecting a level navigates to `/play/dungeon?level=…` (or `/play/mothership` for the final level).
- Shows player's current weapon, current health, gold, and inventory summary so the player knows what they're going in with.

### 5.5 Dungeon (`features/dungeon/`) — Zone 2, the playable game

This is the biggest piece. Split into:

#### 5.5.1 `DungeonPage` (Angular)

Responsibilities:
- Mount `<canvas>` element, instantiate `GameEngine`.
- Call `POST /runs` to obtain a `runId` + `seed`; pass them to the engine.
- Fetch content (rooms, enemies, items) from the API and hand it to the engine.
- Render HUD components as an overlay (`<app-hud>`).
- Subscribe to `engine.events$`:
  - `enemyKilled` → update HUD score & kill count signals.
  - `lowHealth` → flash red overlay.
  - `playerDied` → call `POST /runs/{id}/complete` with outcome=`Died`, navigate to summary.
  - `runComplete` → same but outcome reflects what happened.
  - `clockFound` → show in-game time briefly.
  - `sweeperReleased` → play sting + ominous overlay.
- Heartbeat every 10s: `POST /runs/{id}/heartbeat`.
- Handle pause (Esc): show modal with Resume / Settings / Quit (quit = abandon).

#### 5.5.2 Game engine modules (plain TypeScript under `engine/`)

| Module                  | Responsibility                                                       |
| ----------------------- | -------------------------------------------------------------------- |
| `scene.ts`              | Three.js scene, camera, renderer, lighting, render loop              |
| `procedural-generator.ts` | From the run seed, picks room templates and stitches them with tunnels. Each room has up to 4 doors per the PDF. |
| `player-controller.ts`  | WASD movement, mouse-look, run-with-limited-stamina, jump (if needed) |
| `weapons.ts`            | Pipe, Sword, Wrench, Pistol, Shotgun, Rifle — swing/fire timing, durability, damage |
| `enemy-ai.ts`           | State machines for Sword, Knight, Pipe, Mini zombies — pathfinding to player, swing animation, contact damage |
| `boss-ai.ts`            | Sweeper (untouchable, hunts player) and Mega (room arena, multi-phase) |
| `obstacles.ts`          | Trip wire, locked box, gun turret — trigger logic & disarm interactions |
| `combat.ts`             | Damage resolution, hit invulnerability frames, multiplier streak     |
| `input.ts`              | Keyboard + mouse event normalization                                 |
| `audio.ts`              | Wraps Howler.js. Zone-aware music, sound-effect bank                 |
| `timer.ts`              | The hidden Sweeper-release countdown + the in-room clocks            |

The render loop runs in `requestAnimationFrame`. Engine state is intentionally **not** Angular signals — change detection on every frame would be a disaster. The engine emits coarse-grained events to Angular at meaningful moments only.

#### 5.5.3 HUD components (`features/dungeon/hud/`)

Overlay rendered on top of the canvas. Each is a small standalone component reading from a `HudStateService` (signals) that the `DungeonPage` updates from engine events. Matches the sketch from page 6 of the design doc.

- `<app-health-bar>` — bottom-right, segmented bar.
- `<app-score-display>` — bottom-left: `200 × 2` (score with multiplier), `20 Gold` underneath.
- `<app-weapon-indicator>` — currently equipped weapon name + durability remaining.
- `<app-ammo-counter>` — only shown when a firearm is equipped (`5/20`).
- `<app-xp-bar>` — slim bar showing progress to next level.
- `<app-low-health-overlay>` — red vignette when hp ≤ 25% of max.
- `<app-sweeper-warning>` — pulsing border + audio cue when Sweeper is released.
- `<app-damage-indicator>` — directional arrow when struck.

Crucially the timer is **not** in the HUD (per the PDF — the player has to find clocks in-room). A `<app-clock-popup>` appears briefly when the engine emits `clockFound`.

#### 5.5.4 Player abilities (mapping PDF → engine)

| PDF ability               | Implementation                                                       |
| ------------------------- | -------------------------------------------------------------------- |
| Walk                      | WASD, default speed                                                  |
| Run (limited time)        | Hold Shift; drains a stamina meter; regenerates when not running. Stamina is not on the HUD by default but is shown if Setting "Show stamina" is on. |
| Shoot                     | Left-click while a firearm is equipped; consumes a round of matching ammo; weapon loses 2 health per shot (per PDF) |
| Melee swing               | Left-click while a melee weapon is equipped; weapon loses 1–2 health per swing depending on weapon (per PDF) |
| Pick up items             | E to interact when reticle is over a pickup                          |
| Disarm trap / pick lock   | E when reticle is over the obstacle; consumes a `TrapDisarmer` or `LockPick` if equipped |
| Throw potion / grenade    | G to throw the active throwable (cycle with mouse wheel)             |

### 5.6 Mothership (`features/mothership/`) — Zone 3

A specialization of the dungeon engine. Different generator (single arena, no procedural rooms), different music (boss tracks), the Sweeper timer is suspended, and the Mega Zombie is spawned as the primary threat with periodic adds emerging from the ship. Same HUD.

### 5.7 Gameplay Summary (`features/summary/`)

After a run ends. Calls `GET /runs/{id}` for the canonical numbers (server is the truth source — the client doesn't trust its own counters here in case of clock skew).

Shows, per the PDF:
- Total points scored
- Number of zombies killed
- Time spent in the dungeon
- Plus: gold earned, XP earned, deepest room reached, top multiplier, outcome (escaped / died / beat the boss)
- Plus: level-up callout if the run pushed the player to a new level

CTA buttons: "Back to Shop", "Play again", "View career".

### 5.8 Profile (`features/profile/`)

- **Stats tab** — lifetime kills, runs completed, best score, deepest room ever reached.
- **Inventory tab** — every item the player owns, grouped by category. Click a weapon to equip it (calls `POST /players/me/inventory/{itemId}/equip`).
- **Upgrades tab** — visualizes the player's level, XP progress, max-health milestones (per PDF, +5 max HP per level).
- **Run history tab** — table of past runs (date, outcome, score, kills, depth).

### 5.9 Settings (`features/settings/`)

- Audio: master / music / sfx sliders.
- Video: resolution scale, FOV, draw distance.
- Controls: rebind keys (stored client-side in localStorage).
- Account: change password, log out.

---

## 6. Shared UI (`shared/`)

A tiny set of primitives — we keep it small and rely on Tailwind utility classes for the rest.

- `<app-button>` — variants (primary, secondary, danger).
- `<app-modal>` — backdrop + focus trap.
- `<app-loading>` — skeleton + spinner.
- `<app-stat-bar>` — generic segmented bar (used by HUD health, XP, durability).
- `<app-toast>` — non-blocking notifications (purchase succeeded, level up).
- `<app-icon>` — looks up SVG by slug, used by item icons.

---

## 7. State Management

Three tiers:

1. **Signals inside components** — the default. Local UI state (modal open, tab index, form values).
2. **Singleton services with signals** — cross-cutting state that survives navigation:
   - `AuthService` — `currentUser`, `isAuthenticated`.
   - `PlayerService` — `player`, `inventory`, `gold`, `level`. Loaded on login, kept fresh after shop purchases and run completions.
   - `HudStateService` — bridges engine events to HUD components during a run.
   - `AudioService` — music + sfx playback.
3. **Engine state** — owned by the Three.js layer, not reactive. Mutates 60+ times per second.

No NgRx. If scope grows, we can introduce `@ngrx/component-store` per feature; we don't need a global store.

---

## 8. API Integration

- Generated TS client lives in `src/api/`. Regenerated by `npm run api:generate` against `swagger.json`.
- Each feature has a thin `*.facade.ts` (or service) that wraps the generated client with the Angular conveniences we want (signals, error toasts, retries).
- `ErrorInterceptor` — converts `problem+json` responses to `AppError` and pushes them through a `NotificationService` for toast display. Specific 4xx codes (`401`, `403`, `422`) get bespoke handling.

### 8.1 Run lifecycle in the client

```
LevelSelectPage → "Enter dungeon"
   │
   ▼
POST /runs          ─── server returns { runId, seed }
   │
   ▼
DungeonPage mounts
   ├─ load content (zones/rooms/enemies/items) — cached
   ├─ instantiate GameEngine(seed, content)
   ├─ start render loop
   └─ subscribe to engine.events$
        ├─ every 10s: POST /runs/{id}/heartbeat
        ├─ on roomEntered: update HUD, log event
        └─ on playerDied | runComplete:
              POST /runs/{id}/complete { outcome, totals }
              navigate → /play/summary/:runId
```

If the user closes the tab mid-run, the server-side run stays `InProgress` until either a `heartbeat` times out (15 min) or the player starts a new run (the server auto-abandons the stale one with a 409 fallback). The next time the player loads the level-select page, we show a "Resume run?" banner if their run is still `InProgress`.

---

## 9. Assets

Full asset pipeline details belong in the companion [3d-engine-design.md](./3d-engine-design.md). Summary:

- **Visual target**: pixel-textured 3D — low-poly meshes with small (16–64 px) nearest-filtered textures. The scene is optionally rendered to a low-resolution offscreen target (e.g. 640×360) and upscaled with nearest-neighbor for a chunky retro feel.
- **Sourcing**: free CC0 asset packs (Kenney, Quaternius, Synty's free tier, itch.io pixel-3D packs) for base meshes. Procedural primitive geometry where no pack asset fits. Textures are authored as pixel art (Aseprite) or downsampled from pack PBR maps to low-color palettes.
- **3D models**: `.glb` (glTF binary, Draco-compressed) in `assets/models/` — Gathar's first-person hands, each weapon, each zombie variant, room kit pieces.
- **Textures**: `.png` (indexed or low-color) under `assets/textures/`, organized by material. Loaded with `THREE.NearestFilter` on both min and mag.
- **Audio**:
  - Music tracks named per the PDF (`darksides.mp3`, `relapse.mp3`, ...) under `assets/audio/music/`. Played per `MusicTrack.usageTag` (dungeon, shopkeep, boss).
  - Sfx under `assets/audio/sfx/` — combined into a sprite via Howler for low-latency triggers.
- **Sprites & icons** under `assets/sprites/` — pixel-art UI items, indexed by the `iconSlug` from the API.

Assets are served from a CDN in production, not bundled. The client lazy-loads them in chunks (`GLTFLoader` for models, Howler for audio).

---

## 10. Performance

- Three.js renderer at 60 fps target. Object pooling for projectiles and damage numbers.
- Instanced meshes for repeated geometry (lockers, basketballs, conference chairs).
- Angular ChangeDetectionStrategy.OnPush everywhere outside the engine.
- HUD subscribes to a `HudStateService` that throttles "noisy" engine events (e.g. health changes during a damage-over-time) to 10 Hz max.
- Use `requestIdleCallback` for non-critical work (analytics flushes, prefetching the next room's assets).

---

## 11. Accessibility

Realistically, a first-person 3D shooter has accessibility limits, but the non-game pages should be fully accessible:

- All non-game pages: WCAG 2.1 AA — semantic HTML, proper headings, keyboard navigation, ARIA labels on icon buttons.
- Color: no information conveyed by color alone (health bar also has a numeric reading).
- Settings page exposes: subtitles for spoken/diegetic sounds, "high-contrast HUD" toggle, motion-reduction (camera bob, screen shake).
- The dungeon itself has a "low-flash" mode that mutes the red damage vignette and Sweeper-warning pulse.

---

## 12. Testing

| Layer                  | Tool                     | What we test                                                |
| ---------------------- | ------------------------ | ----------------------------------------------------------- |
| Components             | Jest + Angular TestBed   | Renders, inputs/outputs, signal-driven behavior             |
| Services               | Jest                     | Pure logic, HTTP mocked via interceptor                     |
| Generated API client   | n/a                      | Trust the generator + the C# contract tests                 |
| Engine modules         | Jest                     | Pure-TS modules (combat math, generator determinism)        |
| End-to-end             | Playwright               | Sign up → buy item → enter dungeon → die → summary visible  |
| Visual regression      | Playwright screenshots   | HUD layout, shop layout, summary screen                     |

E2e tests run against a containerized API + Angular dev server in CI.

---

## 13. Build & Deployment

- `ng build --configuration production` → static bundle.
- Hosted as a static SPA (Cloudflare Pages / Netlify / S3 + CloudFront — TBD).
- Environment file picks API base URL per env: `dev.api.livingland.example`, `api.livingland.example`.
- CSP header restricts script/img/audio sources to our CDN and API origin.
- Sourcemaps uploaded to Sentry on release.

---

## 14. Open Questions

1. **Difficulty scaling specifics** — the PDF says Sword/Knight scale with player level but Mini/Pipe don't. Proposed formula: scaling enemies gain `+1 HP` per player level above 1 and `+1 hitDamage` every 3 player levels. Confirm or override in the next pass.
2. **Procedural generation determinism** — proposed default: deterministic from the run seed so any run can be replayed for debugging or sharing. Confirmed unless objected.
3. **Saving mid-run** — explicitly *not* supported per the PDF spirit (the dungeon is a single push). Closing the tab forfeits the run after a 15-minute heartbeat gap.
4. **Admin CMS UI** — the API supports admin writes; the in-Angular `/admin/**` section is deferred to post-MVP. Designers edit content via JSON fixtures + the seed endpoint until then.
5. **First-time tutorial** — the PDF describes a tutorial. MVP plan: a scripted first run after registration that surfaces tooltip overlays for movement, melee, pickup, and shop. Detailed flow lives in a future tutorial-spec doc.
6. **Run replays** — once procgen is deterministic and we have a `RunEvent` log, replays become viable. Deferred but the data model already supports it.
