# The Living Land — REST API Design

**Status:** Draft v1
**Owner:** Bolder Games, LLC
**Stack:** ASP.NET Core 8, Entity Framework Core 8, SQL Server, JWT auth

---

## 1. Overview

The Living Land API is the authoritative backend for a browser-based, first-person zombie dungeon crawler. It serves two broad responsibilities:

1. **Content service** — exposes the static game design data (zones, rooms, enemies, bosses, obstacles, items) sourced from the game design document. This data is read-heavy and changes only when designers edit it.
2. **Player service** — manages player accounts, persistent player state (level, gold, XP, inventory), and the history of completed dungeon runs (kills, time, depth, score).

Combat itself is **resolved on the client** (the Angular game client). The API is not authoritative over moment-to-moment combat; instead, the client periodically syncs state and posts completed-run summaries. This is a deliberate scope choice — see §13 *Non-Goals*.

### 1.1 Goals

- Provide a clean, predictable, versioned REST surface for both the game client and a future admin/CMS UI.
- Be the single source of truth for game content; designers edit content via admin endpoints, never by re-deploying the client.
- Persist enough player state for cross-device play and a meaningful "career" arc (levels, gold, inventory, run history).
- Support both anonymous browsing of public content (wiki-style) and authenticated player/admin actions.

### 1.2 Non-Goals (v1)

- Real-time combat simulation, multiplayer, or PvP.
- Authoritative anti-cheat. Clients submit run results; server applies sanity checks (see §10) but does not re-simulate combat.
- Procedural dungeon generation server-side. The client generates the layout from a seed; the server stores the seed and the room-type list it expanded to.
- Matchmaking, chat, friends, or leaderboards beyond a basic top-N high-score endpoint (deferred to v2).

---

## 2. Architecture

```
┌─────────────────────┐        HTTPS/JSON         ┌──────────────────────┐
│  Angular Client     │ ─────────────────────────►│  ASP.NET Core 8 API  │
│  (browser game)     │ ◄─────────────────────────│  (this document)     │
└─────────────────────┘        JWT bearer         └──────────┬───────────┘
                                                             │ EF Core
                                                             ▼
                                                  ┌──────────────────────┐
                                                  │  SQL Server          │
                                                  │  (content + players) │
                                                  └──────────────────────┘
```

Single-process monolith. The API project is layered as:

- **Controllers** — thin, attribute-routed, return `IActionResult`/`ActionResult<T>`. No business logic.
- **Services** — business logic (`IShopService`, `IDungeonRunService`, `ICombatRulesService`, etc.). Constructor-injected.
- **Repositories** *(thin wrapper over EF Core `DbContext`)* — only used where a query is reused in 3+ places. Otherwise services use `DbContext` directly. We're not over-abstracting.
- **Domain entities** — POCOs that map to EF Core tables.
- **DTOs** — request/response shapes; never expose entities directly across the wire.
- **Validators** — FluentValidation per request DTO.

### 2.1 Project Layout

```
src/
  LivingLand.Api/              # ASP.NET Core host, controllers, middleware
  LivingLand.Application/      # Services, DTOs, validators
  LivingLand.Domain/           # Entities, enums, domain exceptions
  LivingLand.Infrastructure/   # EF Core DbContext, migrations, identity
tests/
  LivingLand.Api.Tests/        # WebApplicationFactory integration tests
  LivingLand.Application.Tests/# Unit tests for services
```

---

## 3. Tech Stack & Conventions

| Concern              | Choice                                             |
| -------------------- | -------------------------------------------------- |
| Framework            | ASP.NET Core 8 (minimal hosting + MVC controllers) |
| ORM                  | EF Core 8 (code-first migrations)                  |
| Database             | SQL Server 2022 (LocalDB for dev)                  |
| Auth                 | ASP.NET Core Identity + JWT bearer                 |
| Validation           | FluentValidation 11                                |
| Mapping              | Mapster (lighter than AutoMapper)                  |
| Logging              | Serilog → console + rolling file                   |
| API docs             | Swashbuckle / OpenAPI                              |
| Testing              | xUnit + FluentAssertions + Testcontainers (SQL)    |
| Containerization     | Docker (multi-stage build), `docker-compose` dev   |

### 3.1 Conventions

- All routes prefixed `/api/v1/`. Version segment is mandatory.
- `kebab-case` for URL segments, `camelCase` for JSON properties.
- Resource collections are pluralized (`/enemies`, not `/enemy`).
- IDs in URLs are integer surrogate keys; entities also expose a stable `slug` (e.g. `"sword-zombie"`) for client-side asset lookup.
- All timestamps are UTC ISO-8601 strings.
- All error responses follow RFC 7807 `application/problem+json`.

---

## 4. Domain Model

The entities below are derived directly from the design document. Field names match PDF terminology where possible.

### 4.1 Content entities (designer-managed, read-only to players)

#### `Zone`
The three macro-areas described in the PDF. The PDF text mentions "Three Zones of five levels each" but also references a Zone 4 in the contents — we treat Zone 4 as reserved/future and ship Zones 1–3 in v1.

| Field        | Type      | Notes                                                  |
| ------------ | --------- | ------------------------------------------------------ |
| `id`         | int PK    |                                                        |
| `slug`       | string    | `"shopkeep"`, `"dungeon"`, `"mothership"`              |
| `name`       | string    | "The Shopkeep"                                         |
| `ordinal`    | int       | 1, 2, 3                                                |
| `description`| string    | Long-form lore text                                    |
| `levelCount` | int       | 5 per the PDF                                          |
| `musicTrackIds` | int[]  | FK list → `MusicTrack`                                 |

#### `Level`
A logical progression unit inside a zone.

| Field        | Type      | Notes                                       |
| ------------ | --------- | ------------------------------------------- |
| `id`         | int PK    |                                             |
| `zoneId`     | int FK    | → `Zone`                                    |
| `ordinal`    | int       | 1–5 within zone                             |
| `name`       | string    |                                             |
| `description`| string    | Goal / challenge text shown on level select |
| `minPlayerLevel` | int   | Gates access                                |
| `recommendedPlayerLevel` | int |                                       |
| `bossId`     | int? FK   | Optional → `Boss` (for boss-end levels)     |

#### `Room`
The 10 room templates described in the PDF (Locker, Basketball, Bed, Office, Conference, Kitchen, Barracks, Armory, Park, Living). Procedural generation picks from these.

| Field            | Type    | Notes                                            |
| ---------------- | ------- | ------------------------------------------------ |
| `id`             | int PK  |                                                  |
| `slug`           | string  | `"locker-room"`, etc.                            |
| `name`           | string  | "Locker Room"                                    |
| `description`    | string  | Lore / visual description from the PDF          |
| `rarity`         | enum    | `Common`, `Uncommon`, `Rare` (Armory is Rare)    |
| `possibleEnemyIds` | int[] | FK list of allowed `Enemy` spawns                |
| `possibleLootIds`  | int[] | FK list of allowed `Item` drops                  |
| `possibleObstacleIds` | int[] | FK list of allowed `Obstacle` placements      |

#### `Enemy`
Sword, Knight, Pipe, Mini, plus shared schema for special enemies.

| Field         | Type    | Notes                                                |
| ------------- | ------- | ---------------------------------------------------- |
| `id`          | int PK  |                                                      |
| `slug`        | string  | `"sword-zombie"`                                     |
| `name`        | string  |                                                      |
| `kind`        | enum    | `Standard`, `Boss`, `Sweeper`                        |
| `hitPoints`   | int     | From PDF table                                       |
| `pointsValue` | int     |                                                      |
| `goldValue`   | int     |                                                      |
| `hitDamage`   | int     | Damage dealt to player on hit                        |
| `moveSpeed`   | enum    | `Slow`, `MediumFast`, `VeryFast`                     |
| `description` | string  |                                                      |
| `scalesWithPlayerLevel` | bool | True for Sword/Knight/Sweeper; false for Mini/Pipe |

#### `Boss`
Sweeper Zombie and Mega Zombie.

| Field          | Type   | Notes                                                  |
| -------------- | ------ | ------------------------------------------------------ |
| `id`           | int PK |                                                        |
| `slug`         | string | `"sweeper-zombie"`, `"mega-zombie"`                    |
| `name`         | string |                                                        |
| `enemyId`      | int FK | → `Enemy` (bosses are a specialization)                |
| `isKillable`   | bool   | Sweeper = false, Mega = true                           |
| `behavior`     | string | "Released at timer zero, hunts player", etc.          |
| `introCueSlug` | string | Audio cue (e.g. `"sweeper-released"`)                  |

#### `Obstacle`
Trip Wire, Locked Box, Gun Turret.

| Field        | Type   | Notes                                                  |
| ------------ | ------ | ------------------------------------------------------ |
| `id`         | int PK |                                                        |
| `slug`       | string |                                                        |
| `name`       | string |                                                        |
| `damage`     | int    | Damage on trigger (0 for Locked Box)                   |
| `disarmable` | bool   |                                                        |
| `disarmTools`| enum[] | `TrapDisarmer`, `Melee`, `Firearm`, `LockPick`         |
| `description`| string |                                                        |

#### `Item`
Single table with discriminator. Covers weapons, ammo, potions, misc. Field shape varies; uses TPH (table-per-hierarchy) in EF Core.

| Field        | Type    | Notes                                                  |
| ------------ | ------- | ------------------------------------------------------ |
| `id`         | int PK  |                                                        |
| `slug`       | string  |                                                        |
| `name`       | string  |                                                        |
| `category`   | enum    | `Gun`, `Melee`, `Ammo`, `Potion`, `Misc`               |
| `cost`       | int     | In gold                                                |
| `health`     | int?    | For weapons (durability)                               |
| `hitDamage`  | int?    | For weapons                                            |
| `amount`     | int?    | For ammo (rounds per box)                              |
| `effectText` | string? | For potions/misc                                       |
| `iconSlug`   | string  | Frontend uses this to find the asset                   |

#### `MusicTrack`
| Field        | Type    | Notes                                                  |
| ------------ | ------- | ------------------------------------------------------ |
| `id`         | int PK  |                                                        |
| `slug`       | string  |                                                        |
| `title`      | string  | "Darksides", etc.                                      |
| `usageTag`   | enum    | `Dungeon`, `Shopkeep`, `Boss`                          |

### 4.2 Player entities (per-account, mutable)

#### `Player`
The user's persistent profile. One per account.

| Field          | Type      | Notes                                                |
| -------------- | --------- | ---------------------------------------------------- |
| `id`           | guid PK   |                                                      |
| `userId`       | guid FK   | → AspNetUsers (Identity)                             |
| `characterName`| string    | Defaults to "Gathar" but user can rename             |
| `level`        | int       | Starts at 1                                          |
| `currentXp`    | int       |                                                      |
| `maxHealth`    | int       | Starts at 20, +5 per level (per PDF)                 |
| `currentHealth`| int       | Persisted so session can resume                      |
| `gold`         | int       |                                                      |
| `createdAt`    | datetime  |                                                      |
| `lastPlayedAt` | datetime  |                                                      |

#### `InventoryItem`
A line item in a player's inventory.

| Field         | Type    | Notes                                                  |
| ------------- | ------- | ------------------------------------------------------ |
| `id`          | guid PK |                                                        |
| `playerId`    | guid FK |                                                        |
| `itemId`      | int FK  | → `Item`                                               |
| `quantity`    | int     | For stackables (ammo, potions, lock picks)             |
| `currentHealth` | int?  | For weapons; tracks remaining durability               |
| `equipped`    | bool    | True for the currently-wielded weapon                  |

Unique constraint: `(playerId, itemId)` for stackables; weapons with durability are stored one row per instance.

#### `DungeonRun`
One playthrough into the dungeon, from entering the vault to exiting (or dying).

| Field           | Type      | Notes                                              |
| --------------- | --------- | -------------------------------------------------- |
| `id`            | guid PK   |                                                    |
| `playerId`      | guid FK   |                                                    |
| `seed`          | long      | The PRNG seed the client used for room generation  |
| `startedAt`     | datetime  |                                                    |
| `endedAt`       | datetime? |                                                    |
| `outcome`       | enum      | `InProgress`, `Escaped`, `Died`, `BeatMegaZombie`  |
| `zombiesKilled` | int       |                                                    |
| `goldEarned`    | int       |                                                    |
| `xpEarned`      | int       |                                                    |
| `deepestRoom`   | int       | Index of the deepest room reached                  |
| `topMultiplier` | int       | Highest kill-streak multiplier achieved            |
| `finalScore`    | int       |                                                    |

#### `RunEvent` *(optional, deferred to v2 if scope tight)*
Append-only event log per run: `roomEntered`, `enemyKilled`, `itemPickedUp`, `hitTaken`. Useful for replays and anti-cheat sanity checks but not required for v1's summary screen.

### 4.3 ER diagram (textual)

```
AspNetUsers (Identity)
   │ 1:1
Player ─┬──< InventoryItem >──── Item
        └──< DungeonRun

Zone ──< Level ──> Boss ──> Enemy
Room ──< RoomEnemy >── Enemy
Room ──< RoomLoot  >── Item
Room ──< RoomObstacle >── Obstacle
```

---

## 5. API Surface

All routes are prefixed `/api/v1`. JSON only. Bearer-token auth except where noted.

### 5.1 Auth & Identity

| Method | Route                       | Auth         | Purpose                                  |
| ------ | --------------------------- | ------------ | ---------------------------------------- |
| POST   | `/auth/register`            | anonymous    | Create account + Player profile         |
| POST   | `/auth/login`               | anonymous    | Returns JWT + refresh token              |
| POST   | `/auth/refresh`             | refresh tok. | Rotate access token                      |
| POST   | `/auth/logout`              | bearer       | Revoke refresh token                     |
| GET    | `/auth/me`                  | bearer       | Returns current user + roles             |

JWT carries `sub` (user id), `playerId`, and `role` (`Player` or `Admin`). 15-minute access token, 14-day refresh token (httpOnly cookie).

### 5.2 Content (public read, admin write)

Every content collection follows the same pattern:

| Method | Route                       | Auth     | Purpose                          |
| ------ | --------------------------- | -------- | -------------------------------- |
| GET    | `/{resource}`               | anon     | List + filter + paginate         |
| GET    | `/{resource}/{idOrSlug}`    | anon     | Fetch one                        |
| POST   | `/{resource}`               | admin    | Create                           |
| PUT    | `/{resource}/{id}`          | admin    | Replace                          |
| PATCH  | `/{resource}/{id}`          | admin    | Partial update (JSON Merge Patch)|
| DELETE | `/{resource}/{id}`          | admin    | Soft-delete (sets `deletedAt`)   |

Resources: `zones`, `levels`, `rooms`, `enemies`, `bosses`, `obstacles`, `items`, `music-tracks`.

Useful nested reads (avoid round-trips):

- `GET /zones/{id}/levels`
- `GET /levels/{id}/rooms` (the room templates eligible to spawn)
- `GET /rooms/{id}/enemies` / `/loot` / `/obstacles`
- `GET /items?category=Gun&maxCost=200`

#### Query conventions

```
?page=1&pageSize=25&sort=name,asc&q=zombie&category=Gun
```

Pagination response envelope:

```json
{
  "data": [ /* items */ ],
  "page": 1,
  "pageSize": 25,
  "totalItems": 137,
  "totalPages": 6
}
```

### 5.3 Player

| Method | Route                                    | Auth   | Purpose                                       |
| ------ | ---------------------------------------- | ------ | --------------------------------------------- |
| GET    | `/players/me`                            | player | Full profile (level, xp, gold, health)        |
| PATCH  | `/players/me`                            | player | Rename character, etc.                        |
| GET    | `/players/me/inventory`                  | player | All inventory items                           |
| POST   | `/players/me/inventory/{itemId}/equip`   | player | Equip a weapon (un-equips others of category) |
| GET    | `/players/me/stats`                      | player | Aggregate stats (lifetime kills, runs, etc.)  |

Admins can read any player at `/players/{id}` (for support).

### 5.4 Shop

The shop is the canonical "Zone 1" interaction. Pricing comes from `Item.cost`.

| Method | Route                       | Auth   | Purpose                                       |
| ------ | --------------------------- | ------ | --------------------------------------------- |
| GET    | `/shop/inventory`           | player | Items available to the current player. Filtered by `Item.cost` ≤ player's unlocked tier (which is gated by player level). |
| POST   | `/shop/purchase`            | player | Buy items. Body: `[{itemId, quantity}]`. Server validates gold, debits, adds to inventory. |
| POST   | `/shop/sell`                | player | Future — sell durability-remaining weapons    |

`POST /shop/purchase` is transactional. Either the full basket succeeds or none of it does. Returns updated `gold` balance and new inventory snapshot.

### 5.5 Dungeon Runs

The lifecycle: client calls `start`, plays the run client-side, periodically `heartbeat`s, calls `complete` (or `abandon`) at the end.

| Method | Route                              | Auth   | Purpose                                          |
| ------ | ---------------------------------- | ------ | ------------------------------------------------ |
| POST   | `/runs`                            | player | Start a new run. Server returns `runId` + `seed`. Player must have ≥1 hp and not have an `InProgress` run already. |
| GET    | `/runs/{id}`                       | player | Read current state                               |
| POST   | `/runs/{id}/heartbeat`             | player | Client snapshots progress (room reached, current hp, kills so far). Server persists. |
| POST   | `/runs/{id}/complete`              | player | Finalize: outcome, totals. Server validates sanity, applies XP/gold to Player, returns level-up info. |
| POST   | `/runs/{id}/abandon`               | player | Mark `Abandoned`; no rewards                     |
| GET    | `/players/me/runs?limit=20`        | player | Run history for the summary/career screen        |

**Sanity checks on `complete`** (lightweight anti-cheat; not authoritative):

- `goldEarned` ≤ `zombiesKilled` × (max single-kill gold across all enemy types) × 2.
- Run duration ≥ `zombiesKilled` × min-time-per-kill (e.g. 0.4s).
- `xpEarned` derived server-side from `zombiesKilled` + `deepestRoom`; client-submitted xp is ignored.
- `finalScore` recomputed server-side from `zombiesKilled` × base + multiplier. Submitted value is logged but not trusted.

Anything failing checks → run is accepted as `Outcome.Died` with no rewards, and a `RunFlagged` event is logged for admin review.

### 5.6 Leaderboards (lightweight, v1)

| Method | Route                       | Auth   | Purpose                                     |
| ------ | --------------------------- | ------ | ------------------------------------------- |
| GET    | `/leaderboard/high-scores?limit=100` | anon | Top runs by `finalScore`                |
| GET    | `/leaderboard/deepest?limit=100`     | anon | Top runs by `deepestRoom`               |
| GET    | `/leaderboard/me`                    | player | Player's own ranks                      |

### 5.7 Admin

| Method | Route                       | Auth  | Purpose                                                       |
| ------ | --------------------------- | ----- | ------------------------------------------------------------- |
| GET    | `/admin/players`            | admin | Paginated player list                                         |
| POST   | `/admin/players/{id}/grant` | admin | Manually grant gold/items (support tool)                      |
| GET    | `/admin/flagged-runs`       | admin | Runs that failed sanity checks                                |
| POST   | `/admin/content/seed`       | admin | Re-seed content from canonical fixtures (idempotent)         |

---

## 6. Authentication & Authorization

- **ASP.NET Core Identity** stores users + password hashes (PBKDF2).
- **JWT bearer** for the API; tokens signed with HS256 from a secret in app config (in production: rotated, stored in a secret manager).
- **Refresh tokens** are opaque random strings stored in `RefreshTokens` table with `userId`, `expiresAt`, `revokedAt`. Delivered as `HttpOnly; Secure; SameSite=Lax` cookies.
- **Roles**: `Player` (default), `Admin` (manually granted). Use `[Authorize(Roles="Admin")]` on admin controllers.
- **Anonymous endpoints**: marked `[AllowAnonymous]`. Content GETs are anonymous; content writes require `Admin`.

### 6.1 CORS

Allow the Angular client's origin (configurable per environment). Allow credentials so the refresh-token cookie works.

### 6.2 Rate limiting

ASP.NET Core's built-in rate limiter:
- 10 requests/sec sliding window per IP for anonymous routes.
- 30 requests/sec per user for authenticated routes.
- 5 requests/min per IP for `/auth/login` and `/auth/register`.

---

## 7. Error Handling

All errors return `application/problem+json` per RFC 7807:

```json
{
  "type": "https://livingland.example/errors/insufficient-gold",
  "title": "Insufficient gold",
  "status": 400,
  "detail": "Player has 30 gold but purchase requires 100.",
  "instance": "/api/v1/shop/purchase",
  "errors": { /* optional FluentValidation field errors */ }
}
```

A global `ExceptionMiddleware` maps domain exceptions to status codes:

| Exception                          | Status | When                                 |
| ---------------------------------- | ------ | ------------------------------------ |
| `EntityNotFoundException`          | 404    | GET/PATCH/DELETE on unknown id       |
| `ValidationException` (FluentVal.) | 400    | Bad request body                     |
| `DomainRuleException`              | 422    | E.g. "not enough gold", "level locked" |
| `ConflictException`                | 409    | E.g. starting a run while one is in progress |
| `UnauthorizedAccessException`      | 401    | Bad/missing token                    |
| `ForbiddenException`               | 403    | Authenticated but lacks role         |
| anything else                      | 500    | Logged with correlation id           |

Every response carries a `X-Correlation-Id` header so client logs can be tied to server logs.

---

## 8. Validation

FluentValidation per DTO. Examples:

- `RegisterRequest`: email is email-format, password ≥ 10 chars w/ digit + symbol, characterName 3–20 chars `[A-Za-z0-9_-]`.
- `PurchaseRequest`: at least one line item; each `quantity` ≥ 1 and ≤ 99.
- `RunCompleteRequest`: `zombiesKilled` ≥ 0 and ≤ 10,000; `endedAt` > `startedAt`; outcome is a valid enum value.

Validation failures return a 400 with `errors` keyed by JSON path.

---

## 9. Database

### 9.1 Migrations

EF Core code-first migrations under `LivingLand.Infrastructure/Migrations`. CI runs `dotnet ef database update` against an ephemeral SQL Server instance to verify migrations are clean before merge.

### 9.2 Seeding

Content data ships as JSON fixtures in `LivingLand.Infrastructure/Seed/`:
- `enemies.json`, `bosses.json`, `rooms.json`, `obstacles.json`, `items.json`, `zones.json`, `levels.json`, `music-tracks.json`.

`POST /admin/content/seed` (or the `dotnet run -- seed` CLI subcommand) reads these and upserts by `slug`. Idempotent.

The initial fixtures are derived verbatim from the design document tables. Designers edit JSON in version control; admins can also edit via the API for hot-fixes (and the change is exported back to JSON via a `content:export` CLI task).

### 9.3 Soft delete

Entities support soft delete via a `DeletedAt` column and an EF Core global query filter. Deleted content is hidden from GETs but preserved for referential integrity (an old run referencing a now-deleted enemy still resolves).

---

## 10. Cross-Cutting

### 10.1 Caching

- `Cache-Control: public, max-age=300` on anonymous content GETs.
- In-memory cache (`IMemoryCache`) for content collections, invalidated on writes. Most content fits comfortably in memory.
- ETags via `IETagGenerator` for individual entity GETs.

### 10.2 Logging & telemetry

- Serilog with structured properties: `userId`, `playerId`, `correlationId`, `runId`.
- OpenTelemetry traces over OTLP (optional, env-gated).
- Health checks: `/health/live` (process up), `/health/ready` (DB reachable).

### 10.3 Versioning

URL-based (`/v1`). When breaking changes are needed, introduce `/v2` controllers in parallel; deprecate `/v1` with `Deprecation` headers per RFC 8594.

### 10.4 OpenAPI

Swashbuckle generates the spec at `/swagger/v1/swagger.json`. The Angular client uses this to generate its TypeScript API client (`ng-openapi-gen` or `nswag`).

---

## 11. Testing Strategy

| Layer            | Tooling                                              | Coverage target |
| ---------------- | ---------------------------------------------------- | --------------- |
| Service unit     | xUnit + FluentAssertions + EF Core InMemory          | ≥ 80%           |
| Validator unit   | FluentValidation.TestHelper                          | 100% of rules   |
| Controller integration | `WebApplicationFactory` + Testcontainers (SQL Server) | All endpoints, happy + error paths |
| Contract         | Swagger schema diff in CI to detect breaking changes | n/a             |

Integration tests reset the database between cases with `Respawn`.

---

## 12. Deployment

- Docker image built via multi-stage Dockerfile (SDK → runtime, ~110MB final).
- `docker-compose.yml` for local dev: API + SQL Server + Seq (log viewer).
- Configuration via env vars (`ASPNETCORE_ENVIRONMENT`, `ConnectionStrings__Default`, `Jwt__Secret`, etc.). User Secrets in dev, secrets manager in prod.
- Migrations apply on container start in non-prod; in prod they're a separate gated step.

---

## 13. Open Questions

1. **Save slots** — does a player have one running game state, or multiple save slots? v1 assumes a single persistent state (the Player row).
2. **Hard-mode / new game+** — out of scope for v1.
3. **Achievements** — modeled as a future `Achievement` entity + `PlayerAchievement` join. Not in v1.
4. **Internationalization** — content strings are English-only in v1. If localization is needed later, add a `LocalizedString` table (entityId, field, locale, value).
5. **Audio / asset hosting** — the API exposes asset *slugs*; binary assets (audio, sprites) are served from static hosting (CDN). Out of scope for the API itself.
