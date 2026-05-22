# Turn 001 — Initial design-doc request

**Date:** 2026-05-21
**User intent:** Ryan handed Claude Code `The Living Land.pdf` (a Bolder Games concept doc) and asked for two design documents derived from it: (1) a REST API in C# that tracks the game's Levels, World, Bosses, Enemies, Obstacles, and Player mechanics; (2) an Angular frontend that exposes the Interface, Player Abilities, World/Levels/Challenges, Bosses, Enemies, Obstacles, and Player Mechanics to the user. Ryan invited clarifying questions before writing.

## Clarifying questions asked
Four questions covering: system purpose (CMS vs. live backend vs. hybrid); user roles & auth; API scope ambition; tech-stack constraints.

## Decisions (from Ryan's answers)
- **System purpose:** Live game backend + web client. Angular is the actual playable client; C# API manages player accounts, save state, inventory, scoring, gameplay events. Combat resolution stays client-side.
- **Users & auth:** Player accounts + admin. Three roles (anonymous, player, admin). JWT.
- **API scope:** CRUD-focused content API (no server-side combat simulation). Standard REST with pagination, filtering, sorting.
- **Tech stack:** ASP.NET Core 8 + EF Core + SQL Server. Angular version left to Claude Code's judgment (defaulted to Angular 17+ standalone).

## Actions
- Created `docs/api-design.md` — full REST API design doc:
  - Architecture, project layout, conventions
  - Domain model (Zone, Level, Room, Enemy, Boss, Obstacle, Item, MusicTrack + Player, InventoryItem, DungeonRun)
  - Endpoint surface: auth, content CRUD, player, shop, runs, leaderboards, admin
  - Auth model (Identity + JWT + refresh cookies), CORS, rate limiting
  - Error model (RFC 7807 problem+json), validation (FluentValidation), caching, versioning, OpenAPI
  - Testing strategy, deployment, open questions
- Created `docs/frontend-design.md` — full Angular frontend design doc:
  - Tech stack (Angular 17+, Three.js for the 3D scene, Howler audio, Tailwind UI, signals-based state)
  - Project structure with engine layer isolated from Angular
  - Routing table, feature breakdowns (auth, world wiki, shop, level select, dungeon, mothership, summary, profile, settings)
  - HUD components mapped to the PDF's interface sketch
  - Run lifecycle, asset organization, perf, accessibility, testing, deployment

## Open follow-ups
Surfaced six open questions at the bottom of `frontend-design.md`; the two flagged to Ryan first were art direction (2D vs 3D) and difficulty scaling specifics.
