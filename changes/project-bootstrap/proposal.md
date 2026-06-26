# Proposal: project-bootstrap

> Change: `project-bootstrap` · Scope: **shared (frontend + backend + openspec)** · Project: Air-Pilote · Mode: hybrid

## Intent

Bootstrap a greenfield single-player MVP of Air-Pilote — a 2D top-down (cenital) jet game — by scaffolding two independent git repos, a PixiJS game client, and a NestJS+Postgres backend with JWT auth and score persistence. This is the first productive slice of a multi-year vision (multiplayer, missions, open world deferred).

## Scope

### In Scope

- **Shared**: initialize `frontend/`, `backend/`, and `openspec/` as three independent git repos; no workspace-root `package.json`.
- **Frontend**: Vite + React + TS; manual `PIXI.Application` + DOM overlay; Pixi `Ticker` game loop; `worldContainer` camera follow + clamp; jet movement in all 2D directions; shooting; basic enemy AI; HUD (score, health) via Zustand non-moving layer; auth UI (login/register).
- **Backend**: NestJS + Postgres + hexagonal/DDD per bounded context (`domain`/`application`/`infrastructure`); ESLint `no-restricted-imports` layer guard.
  - `identity` context: register, login, JWT access+refresh, Argon2id, rotating refresh tokens with family detection (httpOnly Secure SameSite=Strict refresh cookie).
  - `game-records` context: persist game record (score, durationMs) + list-by-user / high score per player.
- **Testing**: scaffold FE (Vitest + RTL + Playwright, no canvas pixel assertions v1) and BE (domain unit + app in-memory fakes + infra testcontainers Postgres).

### Out of Scope (NON-GOALS — later SDD changes)

- Real-time multiplayer / authoritative server / WebSocket / netcode / client prediction.
- Missions, objectives, open-world exploration, progress/levels/unlockables, global async leaderboard.

## Capabilities

### New Capabilities

(all greenfield — `openspec/specs/` is empty)

- `frontend-game-client`: PixiJS render + ticker loop, jet movement, shooting, basic enemy AI, top-down scrollable camera, HUD, auth UI, Zustand state bridge.
- `backend-identity`: registration, login, JWT access+refresh, Argon2id, rotating refresh tokens with family detection.
- `backend-game-records`: score persistence per session, high score history per player.

### Modified Capabilities

None — greenfield.

## Approach

Per-bounded-context hexagonal layout (`domain` pure TS → `application` ports/use cases → `infrastructure` NestJS/ORM/JWT adapters). Frontend: manual `PIXI.Application` + React overlay; Pixi owns canvas/ticker, React owns HUD/menus; Zustand bridges slow game-facing state; per-frame data stays in Pixi. MikroORM (Postgres driver, Data Mapper + UoW) behind repository ports; TypeORM fallback documented. Argon2id behind `PasswordHasher` port. Two independent repos + `openspec/` as a third repo.

## Affected Areas

| Area | Impact | Description |
|------|--------|-------------|
| `frontend/` | New | React+TS+PixiJS+Zustand client git repo |
| `backend/` | New | NestJS+Postgres+hexagonal/DDD git repo |
| `openspec/` | New | Workspace SDD context as its own git repo |
| `backend/src/contexts/identity` | New | auth/JWT bounded context |
| `backend/src/contexts/game-records` | New | score persistence bounded context |

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| MikroORM adoption cost / UoW mental model | Med | Seed first context via focused session; TypeORM fallback documented |
| Layer leakage (ORM/Nest types into `application`) | Med | ESLint `no-restricted-imports` enforced early |
| Zustand floods React with per-frame writes | Low | Throttle store writes; per-frame data stays in Pixi |
| Refresh-token rotation race | Low | Family detection + in-flight lock per user |
| Two-repo tool drift | Low | Accept initial drift; extract shared config only if painful |
| No test runner pre-scaffold | Med | Scaffold runners as first work unit |

## Rollback Plan

Each repo is independently deletable: `rm -rf frontend/`, `rm -rf backend/`, or `rm -rf openspec/` (survives independently). Repo `git init` checkpoints allow `git reset --hard` to any bootstrap milestone. No shared runtime dependency exists between repos.

## Dependencies

- Node.js + npm; PostgreSQL instance (local/docker); PixiJS, Zustand, Vite, NestJS, MikroORM, jsonwebtoken, argon2, testcontainers.

## Success Criteria

- [ ] Three independent git repos exist and build (`frontend` dev server runs, `backend` Nest compiles).
- [ ] Player can register + login (JWT); refresh rotates with family detection.
- [ ] Single-player jet moves in all 2D directions, shoots, enemies react, score appears in HUD.
- [ ] Completed session score persists; per-player high score retrievable.
- [ ] Layer guard (`no-restricted-imports`) passes; FE+BE test runners green.