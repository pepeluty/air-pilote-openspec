# Exploration: project-bootstrap

> Change: `project-bootstrap` · Project: Air-Pilote · Mode: hybrid (openspec + Engram)
> Scope: workspace-level (frontend repo + backend repo + shared openspec context)
> Architecture forks are LOCKED upstream — this exploration defines HOW to implement them, not WHETHER.

## Current State

**Greenfield.** The workspace at `/Users/pepeluty/Codigo/games/Air-Pilote` contains only:
- `openspec/` — SDD planning context (config.yaml, specs/, changes/)
- `.atl/` — agent tooling config

No `package.json`, no git history, no source code, no test runner, no lint/format config. Two independent git repositories (frontend, backend) will need to be initialized. Stack is decided upstream and **locked**:

- **Frontend**: React + TypeScript + **PixiJS** (2D WebGL, scene graph, ticker-based loop). Top-down (cenital) jet maneuverable in all 2D directions over a large scrollable panoramic terrain map.
- **Backend**: NestJS + **PostgreSQL** + **Hexagonal Architecture (Ports & Adapters)** + **DDD** (domain → application → infrastructure).
- **Auth**: **JWT** (email + password), access + refresh tokens, stateless, httpOnly cookie or Authorization header.
- **Layout**: TWO INDEPENDENT git repositories in the workspace. `openspec/` stays at workspace root and covers both repos.

## Affected Areas

This change CREATES (nothing is modified — greenfield):
- `frontend/` — new git repo (React + TS + PixiJS client)
- `backend/` — new git repo (NestJS + Postgres + hexagonal/DDD)
- `openspec/changes/project-bootstrap/proposal.md` — proposal produced next phase
- `openspec/changes/project-bootstrap/specs/{domain}/spec.md` — delta specs
- `openspec/changes/project-bootstrap/design.md` — technical design
- `openspec/changes/project-bootstrap/tasks.md` — task breakdown
- Workspace-level tooling stays optional; per-repo tooling is per repo.

The exploration itself does not create repositories or code. It narrows the design space so `proposal`/`spec`/`design` can be written with confidence.

## Approaches

### 1. Frontend: PixiJS ↔ React integration

| Approach | Pros | Cons | Complexity |
|----------|------|------|------------|
| **A. `@pixi/react`** (react-in-jsx, declarative `<Sprite>`/`<Container>`) | React-friendly JSX API; reuses React mental model; declarative scene tree | Renderer lags behind PixiJS releases; per-frame re-render cost; awkward for high-entropy game state (every entity = a component); limited escape hatches for imperative perf tuning; community quieter than core Pixi | Medium |
| **B. Manual `<canvas>` ref + `new PIXI.Application()` in `useEffect`** | Full control; PixiJS ticker drives the game loop natively (no React reconciliation on hot path); React handles menus/HUD/lobby only; clean separation "canvas = game, React = UI shell"; simplest mental model for a game | Must hand-wire lifecycle (init, resize, destroy); no declarative scene graph — scenes built imperatively; bridge between Pixi state and React HUD is hand-rolled (via Zustand or custom event bus) | Medium |
| **C. Hybrid: manual Pixi app + thin React UI overlay (HUD/menus in React, game in Pixi)** | Best of both: performant imperative game render + idiomatic React UI; Pixi owns canvas + ticker, React owns DOM overlay; state shared via a small store (Zustand) that both sides read | Slightly more glue (store subscriptions on both sides); need discipline to keep game logic out of React | Medium-High |

**Recommendation: B/C hybrid (manual Pixi Application + React overlay) + Zustand for bridging state.**
A top-down action game updates dozens of entities per tick; reconciling that through `@pixi/react`'s declarative tree would fight the renderer. React should own the chrome (menu, HUD, pause, login, scores) and PixiJS should own the canvas + ticker. A single Zustand store holds "game-facing" state (score, health, game phase) that both the Pixi scene and the React HUD subscribe to; high-frequency per-entity transforms stay inside Pixi and never touch React.

### 2. Frontend: build tool / scaffolding

| Approach | Pros | Cons | Effort |
|----------|------|------|--------|
| **Vite + React + TS** | Fast HMR, native ESM, first-class TS, tiny config, official React template, pairs perfectly with PixiJS (asset handling via `import`), dev experience unmatched | None worth discussing for a game | Low |
| **Next.js** | SSR, routing, image opt | SSR is meaningless for a canvas game; adds server runtime cost and complexity; Pixi is purely client | High |
| **CRA (create-react-app)** | Familiar | Deprecated/unmaintained; slow | — |

**Recommendation: Vite + React + TS** (`npm create vite@latest frontend -- --template react-ts`). Add PixiJS, Zustand, and (later) React Testing Library + Vitest.

### 3. Frontend: game loop & state ownership

- PixiJS `Ticker` (delta-time, runs on rAF under the hood) MUST drive the simulation update for entities inside the canvas. No custom `requestAnimationFrame` loop — reinventing the ticker buys nothing.
- State lives in **three layers**:
  - **Pixi scene tree** — transforms, sprites, containers (imperative, high frequency, never in React).
  - **Zustand store ("game-facing")** — slow-changing state the HUD cares about: `score`, `health`, `lives`, `phase` (menu/playing/paused/gameOver), `sessionId`. Both Pixi systems and React components subscribe here.
  - **React component state** — purely UI-local (form inputs, settings panel open/closed). Never game-critical.
- **Rule**: high-frequency data (per-frame positions, velocities) stays in Pixi. Zustand is updated only when a value the HUD shows actually changes (e.g. score crosses a tick) — throttled, not every frame.

### 4. Frontend: camera / scrollable top-down terrain

- PixiJS has no built-in "camera". The idiomatic pattern: a `worldContainer` (holds the big terrain + entities), and we move `worldContainer.position` to produce a camera follow.
- `cameraFollow(target, viewport)` each tick: `worldContainer.position.set(viewport.width/2 - target.x, viewport.height/2 - target.y)` (with optional lerp for smoothing and clamp to map bounds so the camera never shows outside the terrain).
- Tiling: a single large `TilingSprite` for the terrain texture + visible-area culling (only render tiles in the camera's viewport) keeps cost flat with map size.
- HUD lives in a **separate non-moving React/DOM layer** above the canvas (camera movement must not move the HUD).

### 5. Backend: project structure for NestJS + Hexagonal + DDD

Recommended folder per bounded context (NestJS-aware, framework stays at the edges):

```
backend/src/
├── shared/                      # shared kernel (value objects, errors, base classes)
└── contexts/
    ├── identity/                # bounded context: auth/users
    │   ├── domain/              # entities, value objects, domain services (NO Nest imports)
    │   │   ├── user.entity.ts
    │   │   ├── credentials.vo.ts
    │   │   └── auth-token.vo.ts
    │   ├── application/         # use cases / command+query handlers
    │   │   ├── register.handler.ts
    │   │   ├── login.handler.ts
    │   │   ├── refresh.handler.ts
    │   │   └── ports/           # PORTS (interfaces), implemented by infra
    │   │       ├── password-hasher.port.ts
    │   │       ├── user-repository.port.ts
    │   │       └── token-signer.port.ts
    │   └── infrastructure/      # ADAPTERS (framework deps live here)
    │       ├── controllers/     # NestJS @Controller — translates HTTP ↔ use cases
    │       ├── nest-identity.module.ts  # NestJS wiring (only place importing @nestjs)
    │       ├── persistence/      # ORM entities + repository adapters
    │       └── jwt/             # token-signer adapter (jsonwebtoken), refresh store
    └── game-records/            # bounded context: scores / progress
        ├── domain/
        ├── application/
        └── infrastructure/
├── app.module.ts               # imports one module per context
└── main.ts                     # bootstrap
```

**Layering rules (enforce via ESLint `no-restricted-imports` in the proposal):**
- `domain/` imports nothing from NestJS or the ORM. Pure TS.
- `application/` imports `domain/` + defines `ports/` (interfaces). No Nest.
- `infrastructure/` imports `application/ports` + `domain` to implement them; NestJS controllers/modules live ONLY here.

### 6. Backend: ORM choice

| ORM | DDD fit | Pros | Cons |
|-----|---------|------|------|
| **Prisma** | Weak-ish | Great DX, type-safe generated client, migrations | Active Record-style API encourages anemic models; mapping between Prisma rows and rich domain entities is manual; value objects need `map`/`serialize` glue; harder to truly hide persistence behind a repository port | 
| **TypeORM** | Medium | Mature, Nest integration, supports custom repositories, can map columns ↔ entities | Data Mapper is optional but defaults still leak; entity decorators put framework metadata in entities (mild leak); value object support manual |
| **MikroORM** | **Best** | Explicit Data Mapper by default; rich entities first-class; full Unit of Work + identity map; serializes value objects natively; repository pattern is the canonical API; great TS support; NestJS package exists | Smaller community than TypeORM; learning curve for UoW/identity map |

**Recommendation: MikroORM (Postgres driver) for DDD-purity.**
Its Data Mapper + Unit-of-Work model matches hexagonal/DDD intent: domain entities stay plain, persistence is an adapter behind a `UserRepository` port. The infra layer owns `EntitySchema`/`@Entity` mappings — domain stays framework-free. **Fallback if team velocity wins**: TypeORM with strict Data Mapper + custom repositories. **Avoid Prisma** for the rich-domain contexts (still fine for read-model queries if we later add a CQRS read side).

### 7. Backend: JWT in hexagonal — port/adapter sketch

- **Ports** (`application/ports/`): `PasswordHasher` (`hash`, `compare`), `TokenSigner` (`signAccess`, `signRefresh`, `verify`), `RefreshTokenStore` (`issue`, `rotate`, `revoke`), `UserRepository` (`findByEmail`, `save`).
- **Adapters** (`infrastructure/`):
  - `BcryptPasswordHasher` (or **Argon2id** — preferred for new projects, memory-hard) implementing `PasswordHasher`.
  - `JwtTokenSigner` (jsonwebtoken) implementing `TokenSigner`.
  - `DbRefreshTokenStore` (or Redis later) implementing `RefreshTokenStore` — stores a hashed refresh token per user, supports rotation + family detection (reuse of an already-rotated refresh = revoke the whole family, classic JWT refresh-rotation defense).
  - ORM-backed adapter implementing `UserRepository`.
- **Password hashing**: **Argon2id** preferred (`@node-rs/argon2` or `argon2` npm). BCrypt acceptable fallback. Hashing is an *infrastructure detail* behind a port — domain never imports the lib.
- **Cookie vs header**: support both — write `Authorization: Bearer <access>` and also accept access token from `httpOnly` cookie when `X-Requested-With: same-origin` is present. Refresh token MUST always go in `httpOnly`, `Secure`, `SameSite=Strict` cookie. Env-driven for local dev.

### 8. Backend: bounded contexts

- **`identity`** — registration, login, refresh, logout, token verify. Aggregates: `User`. Value objects: `Email`, `Credentials`, `AuthToken`.
- **`game-records`** — persistence of "important" game records (scores, progress). Aggregate: `GameRecord` (userId, score, durationMs, endedAt, metadata). Future-proofed for leaderboards but keep v1 minimal: persist + list-by-user.
- Optional later: `sessions` (for refresh-token family tracking) — start inside `identity` and split only if it grows.

### 9. Repo layout

```
Air-Pilote/                       (workspace, NOT a git repo by itself)
├── frontend/                     (independent git repo)
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── eslint.config.* / .prettierrc
│   └── src/...
├── backend/                      (independent git repo)
│   ├── package.json
│   ├── tsconfig.json
│   ├── nest-cli.json
│   └── src/...
├── openspec/                     (workspace-level SDD context, versioned how? see below)
└── .atl/
```

**Question for proposal phase**: should `openspec/` be its own git repo, a submodule of both, or untracked workspace docs?
- *Recommendation*: keep `openspec/` as an **independent third git repo** at the workspace level (`git init openspec/`), so planning artefacts survive each app repo being cloned/deleted and can be shared independently. Each app repo references it via a symlink or simple `../openspec/` read — no submodule coupling.

**Shared tooling**: do NOT hoist eslint/prettier/tsconfig into a workspace root (no root `package.json`). Each repo owns its tooling independently — two truly independent repos. If drift becomes painful later, extract a shared `eslint-config-air-pilote` repo. Default: duplicate configs, accept minor drift.

### 10. Testing strategy (intended — re-eval after scaffolding)

- **Frontend**: Vitest + React Testing Library for HUD/UI/logic; **Playwright** for E2E (covers canvas-mounted game launch + lobby/auth flows). Canvas pixel assertions are explicitly out-of-scope v1 — assert "game phase transitions" and "score appears in HUD", not sprite positions.
- **Backend**: NestJS Jest (already bundled). Three test layers:
  - **Domain** — pure unit tests, no Nest, no DB. Entities/VOs enforce invariants.
  - **Application** — use-case handler tests with **in-memory fake adapters** implementing the ports (fast, no I/O). This is the bulk of behavioural tests.
  - **Infrastructure** — adapter integration tests against a **real Postgres via testcontainers** (preferred) or `pg-mem` (fast but partial). JWT/port adapters tested with real libs.
- `strict_tdd: false` per config — bootstrap scaffolding first, then raise flag once runners exist.

## Recommendation (per fork, consolidated)

| Fork | Recommendation |
|------|----------------|
| PixiJS+React integration | **Manual Pixi Application + React DOM overlay**; `@pixi/react` rejected for per-frame cost |
| Game state home | Pixi scene tree (high-freq) + **Zustand** (game-facing slow state) + React local (UI) |
| Game loop | **Pixi Ticker** (built on rAF), no hand-rolled loop |
| Camera/scroll | Move a `worldContainer`; follow target each tick; clamp to map bounds; HUD in non-moving layer |
| Build tool | **Vite + React + TS** |
| Backend structure | **NestJS with hexagonal layering per bounded context** (`domain`/`application`/`infrastructure`), framework only in `infrastructure` |
| ORM | **MikroORM** (Postgres). Fallback TypeORM. Avoid Prisma for rich-domain contexts |
| Bounded contexts | **`identity`** (auth/JWT) and **`game-records`** (scores). Future `sessions` if needed |
| Password hash | **Argon2id** behind a `PasswordHasher` port |
| Refresh tokens | **Rotating refresh tokens with family detection**, stored hashed, refresh always in httpOnly Secure SameSite=Strict cookie |
| Repo layout | **Two independent git repos** (frontend/, backend/) + **openspec/ as a third repo**; no workspace root package.json; per-repo tooling |
| Testing | FE: Vitest + RTL + Playwright (no canvas pixel assertions v1). BE: domain unit + application with in-memory fakes + infra with testcontainers Postgres |

## Risks

- **`@pixi/react` temptation** — picking it for "React friendliness" risks a re-render ceiling mid-game. Mitigation: decision is research-backed, document the rationale in `design.md`.
- **MikroORM adoption cost** — smaller community, UoW mental model. Mitigation: lock pair-programming session to seed the first context; TypeORM fallback documented.
- **Repository-port abstraction discipline** — easy to leak ORM types into `application`. Mitigation: ESLint `no-restricted-imports` rule planned in `design.md`/`tasks.md`.
- **Two-repo drift** — eslint/tsconfig diverge over time. Mitigation: accept initially; extract shared config repo only if drift becomes painful.
- **openspec/ ownership ambiguity** — could be lost if one repo is deleted. Mitigation: bootstrap `openspec/` as its own git repo in `proposal`/`tasks`.
- **Game loop blocking on React re-renders** — if Zustand updates flood React. Mitigation: throttle store writes; rule: per-frame data stays in Pixi.
- **Auth refresh-token rotation race** — concurrent refresh requests reuse tokens. Mitigation: family detection (reuse revokes family) + short-lived in-flight lock per user.
- **No test runner exists yet** — `strict_tdd: false` is intentional; risk of untested bootstrap grows until scaffolding done. Mitigation: `tasks.md` should scaffold runners early (first work unit).
- **Playwright + WebGL canvas** — flaky on CI. Mitigation: scope E2E to flows around the canvas, not pixel-level canvas assertions.

## Ready for Proposal

**Yes.** Architecture forks are closed; this exploration translated them into concrete structural choices (Pixi manual-mode + Zustand, MikroORM, Argon2id, rotating refresh tokens, per-context hexagonal layout, two independent repos + openspec as third). The next phase (`sdd-propose`) should:
1. State scope per repo (frontend-bootstrap / backend-bootstrap / shared openspec-bootstrap) and tag each delta spec with `frontend-*` / `backend-*` / `shared-*`.
2. Decide `openspec/` as its own git repo (recommended) vs untracked.
3. Lock testing stack once tasks plan the first work unit.
4. Include a rollback plan (each repo independently deletable; openspec/ survives).