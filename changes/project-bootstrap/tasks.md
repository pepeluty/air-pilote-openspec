# Tasks: project-bootstrap

## Review Workload Forecast

| Field | Value |
|-------|-------|
| Estimated changed lines | ~2500-4000 |
| 400-line budget risk | High |
| Chained PRs recommended | Yes |
| Suggested split | 8 PRs (see Work Units) |
| Delivery strategy | ask-on-risk |
| Chain strategy | stacked-to-main |

Decision needed before apply: No
Chained PRs recommended: Yes
Chain strategy: stacked-to-main
400-line budget risk: High

### Suggested Work Units

| Unit | Goal | PR | Base |
|------|------|----|------|
| 1 | git init 3 repos + openspec | PR 1 | main |
| 2 | BE scaffolding + shared kernel | PR 2 | main |
| 3 | identity context full | PR 3 | PR 2 |
| 4 | game-records context full | PR 4 | PR 3 |
| 5 | FE toolchain + engine + store | PR 5 | main |
| 6 | FE entities + systems | PR 6 | PR 5 |
| 7 | FE UI + API client | PR 7 | PR 6 |
| 8 | FE↔BE integration + E2E | PR 8 | PR 7 |

## Phase 1: Foundation

- [ ] 1.1 [shared] `git init` `frontend/`, `backend/`, `openspec/` as 3 independent repos; per-repo `.gitignore` + README
- [ ] 1.2 [backend] Create `backend/{package.json,tsconfig.json,.eslintrc.cjs}` NestJS+MikroORM + `no-restricted-imports` layer guard
- [ ] 1.3 [backend] Create `backend/src/{main.ts,app.module.ts}` + `shared/{domain,application}/*` base Entity/VO/UseCase
- [ ] 1.4 [backend] Create `backend/src/shared/{UserExists.port.ts,AuthGuard.ts,errors/*}` typed errors + global AuthGuard (CRITICAL)
- [ ] 1.5 [frontend] Create `frontend/{package.json,vite.config.ts,tsconfig.json,.eslintrc.cjs}` Vite+React+TS+PixiJS+Zustand+Vitest+Playwright
- [ ] 1.6 [frontend] Create `frontend/src/{main.tsx,App.tsx}` screen router skeleton (phase-based)

## Phase 2: Core Implementation — Backend

- [ ] 2.1 [backend] Create `contexts/identity/domain/{User,RefreshToken,PasswordPolicy}.ts` + `vo/{Email,Credentials,AuthToken}.ts` (policy: min 8, max 72, >=1 letter AND >=1 number)
- [ ] 2.2 [backend] Create `contexts/identity/application/ports/*` (PasswordHasher, TokenSigner, RefreshTokenStore, UserRepository) + `RefreshTokenRow`/`LockHandle` types
- [ ] 2.3 [backend] Create `contexts/identity/application/usecases/{RegisterUser,LoginUser}.ts`
- [ ] 2.4 [backend] Create `contexts/identity/application/usecases/RefreshToken.ts` family detection state machine + try/finally lock
- [ ] 2.5 [backend] Create `contexts/identity/application/usecases/Logout.ts` (revoke family + clear refresh cookie)
- [ ] 2.6 [backend] Create `contexts/identity/infrastructure/*` (Nest module, AuthController, MikroORM repos impl `UserExists`, Argon2+JWT adapters, ORM entities) + `migrations/*`
- [ ] 2.7 [backend] Create `contexts/game-records/domain/{GameRecord,vo/Score}.ts` (non-negative Score VO)
- [ ] 2.8 [backend] Create `contexts/game-records/application/{ports/GameRecordRepository,usecases/{PersistGameRecord,GetHighScore,ListGameRecords}}.ts`
- [ ] 2.9 [backend] Create `contexts/game-records/infrastructure/*` (Nest module, controller, MikroORM repo, ORM entity)

## Phase 3: Core Implementation — Frontend

- [ ] 3.1 [frontend] Create `src/game/state/gameStore.ts` Zustand store (phase/score/health + set; bidirectional bridge)
- [ ] 3.2 [frontend] Create `src/game/engine/{Engine,TickerLoop,WorldContainer}.ts` (PIXI.Application, ticker, camera follow+clamp, store.phase subscription)
- [ ] 3.3 [frontend] Create `src/game/entities/{Jet,Projectile,Enemy}.ts`
- [ ] 3.4 [frontend] Create `src/game/systems/{Movement,Shooting}.ts` (diagonal normalize; cooldown; `playing`-gated)
- [ ] 3.5 [frontend] Create `src/game/systems/{EnemyAI,Spawn,Collision}.ts` (enemy destroyed+score; player -health; health=0→gameOver)
- [ ] 3.6 [frontend] Create `src/ui/api/client.ts` fetch wrapper (token attach, refresh-on-401, credentials: include)
- [ ] 3.7 [frontend] Create `src/ui/screens/{Auth,Menu,GameOver,Paused}Screen.tsx` + `hud/HUD.tsx` DOM overlay

## Phase 4: Testing & Integration

- [ ] 4.1 [backend] Jest unit (in-memory fakes): registration success/duplicate-email/weak-pw; login success/invalid-creds; plaintext-never-stored
- [ ] 4.2 [backend] Jest unit: rotation success/reuse→family-revoke/expired; logout revoke-family; lock release on error path (try/finally torture)
- [ ] 4.3 [backend] Jest unit: AuthGuard ok; valid+DELETED→NonExistentUserError 401 (CRITICAL); foreign/expired→Unauthenticated; Score non-negative; GameRecord aggregate
- [ ] 4.4 [backend] testcontainers Postgres: MikroORM repos; paginated history page-2/size-10→empty + past-end; high score 3000-over-1200/950; no-records→null; cannot-access-others; family revoke persists all `revoked`; UserExists read-model; Argon2/JWT adapters
- [ ] 4.5 [frontend] Vitest unit: diagonal move + normalize; no-input hold; fire cooldown; fire-blocked-outside-playing
- [ ] 4.6 [frontend] Vitest unit: enemy destroyed+score; player damage -health; camera follow; clamp at edge; health→0→gameOver; menu→playing; pause trigger
- [ ] 4.7 [frontend] Vitest unit: slow-state publish (1 write/change); per-frame data isolation (store NOT touched)
- [ ] 4.8 [frontend] Playwright E2E: register→menu; login failure; menu→play→gameOver→score saved; pause/resume; HUD score+health update; no canvas pixel asserts v1
