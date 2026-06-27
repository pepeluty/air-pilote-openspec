# Verification Report

**Change**: project-bootstrap
**Mode**: Standard (strict_tdd: false)
**Verdict**: PASS WITH WARNINGS

---

## Task Completion

| Phase | Tasks | Complete | Incomplete |
|-------|-------|----------|------------|
| Phase 1: Foundation | 6 | 6 | 0 |
| Phase 2: Core Implementation — Backend | 9 | 9 | 0 |
| Phase 3: Core Implementation — Frontend | 7 | 7 | 0 |
| Phase 4: Testing & Integration | 8 | 8 | 0 |
**Total**: 30/30

Evidence: `openspec/changes/project-bootstrap/tasks.md` shows all 30 tasks marked `[x]`. Source files for every task confirmed present (backend: 53 files across `src/{shared,contexts/{identity,game-records},migrations}`; frontend: 33 source files across `src/{game,ui}` + `e2e/game-flow.spec.ts`). All 3 git repos exist with clean history matching the apply-progress commits (backend `main` tip `c4e69bb`; frontend `main` tip `4caf194`; openspec `main` tip `33c680b`).

---

## Build/Type/Lint Evidence

| Repo | tsc | eslint | build | Notes |
|-------|-----|--------|-------|-------|
| backend | exit 0 | exit 0 | N/A | `npx tsc --noEmit` clean; `npx eslint .` clean (layer-guard `no-restricted-imports` active). |
| frontend | exit 0 | exit 0 | exit 0 | `npx tsc --noEmit` clean; `npx eslint . --ext .ts,.tsx` clean; `npm run build` (`tsc && vite build`) produced `dist/` (783 modules, 1.69s). |

---

## Test Execution Evidence

| Repo | Test runner | Tests passed | Tests failed | Total | Exit code | Notes |
|-------|------------|-------------|-------------|-------|-----------|-------|
| backend | Jest | 46 | 0 | 46 | 0 | 7 suites: 5 unit + 2 testcontainers integration (Docker AVAILABLE — both Postgres testcontainers ran green in ~8.5s each). |
| frontend | Vitest | 15 | 0 | 15 | 0 | 5 suites (world-container, movement, shooting, collision, gamestore-bridge). |
| frontend | Playwright | 5 | 0 | 5 | 0 | chromium headless, 4 workers; all DOM-overlay assertions (no canvas pixel reads). |

Backend console.warn (`JWT_ACCESS_SECRET not set…dev fallback`) is an intentional dev-fallback warning from `JwtTokenSigner` for local/test runs — NOT a failure. Production ops warning, not a verify blocker.

---

## Spec Compliance Matrix

### frontend-game-client

| Requirement | Scenario | Status | Evidence |
|-------------|----------|--------|----------|
| Jet Movement | Diagonal movement | PASS | `movement.spec.ts` > "diagonal hold normalizes the direction so magnitude equals cardinal speed" |
| Jet Movement | No input | PASS | `movement.spec.ts` > "no input holds the jet in place: velocity zeroed, position maintained" |
| Shooting | Fire projectile | PASS | `shooting.spec.ts` > "fires one projectile, blocks the next shot by cooldown, then fires again once the cooldown elapses" (cooldown verified) |
| Shooting | Fire blocked outside playing phase | PASS | `shooting.spec.ts` > "blocks fire outside the playing phase (paused)" + "menu" + "gameOver" (3 tests) |
| Basic Enemy AI | Enemy destroyed by projectile | PASS | `collision.spec.ts` > "enemy destroyed by projectile: enemy removed and score increments" |
| Basic Enemy AI | Enemy damages player | PASS | `collision.spec.ts` > "enemy damages the player: health decreases and the enemy is removed" |
| Top-down Scrollable Camera | Follow jet within world | PASS | `movement.spec.ts` > "after movement, WorldContainer.follow is called…" + `world-container.spec.ts` > "follow centers the camera on the target" |
| Top-down Scrollable Camera | Clamp at world edge | PASS | `world-container.spec.ts` > "clamp at edge: does not reveal area outside the world map" (east/south/west/north) |
| HUD Overlay | Score updates on enemy destroyed | PASS | `collision.spec.ts` (score increments via bridge) + `game-flow.spec.ts` > "HUD: score + health elements exist and reflect store values" |
| HUD Overlay | Health updates on damage | PASS | `collision.spec.ts` (health published) + `game-flow.spec.ts` > HUD test (health 100→60 reactive) |
| Game Phases | Start game | PASS | `game-flow.spec.ts` > "menu → play → gameOver…" (Start Game click → HUD visible) |
| Game Phases | Player death | PASS | `collision.spec.ts` > "health reaching zero transitions phase to gameOver" + E2E gameOver path |
| Auth UI | Register success | PASS | `game-flow.spec.ts` > "register: on success transitions to the menu and shows Start Game" |
| Auth UI | Login failure | PASS | `game-flow.spec.ts` > "login failure: invalid credentials display an error without leaving the auth screen" |
| Zustand State Bridge | Slow state publish | PASS | `gamestore-bridge.spec.ts` > "publishes score EXACTLY ONCE per change event (not per frame)" |
| Zustand State Bridge | Per-frame data isolation | PASS | `gamestore-bridge.spec.ts` > "N ticks of movement never touch the store with per-frame positions" (100 ticks, no x/y/vx/vy/facing leak) |

**frontend-game-client compliance**: 16/16 scenarios PASS.

### backend-identity

| Requirement | Scenario | Status | Evidence |
|-------------|----------|--------|----------|
| Registration | Successful registration | PASS | `registration-login.spec.ts` > "creates a user, hashes the password, and issues access + refresh tokens" |
| Registration | Duplicate email rejected | PASS | `registration-login.spec.ts` > "rejects a duplicate email with DuplicateEmailError…" |
| Registration | Weak password rejected | PASS | `registration-login.spec.ts` > "rejects a weak password with PasswordStrengthError before any I/O" |
| Login | Successful login | PASS | `registration-login.spec.ts` > "issues tokens when credentials match" |
| Login | Invalid credentials | PASS | `registration-login.spec.ts` > "rejects a wrong password with InvalidCredentialsError…" (+ unknown user, no enumeration) |
| Refresh Token Rotation | Successful rotation | PASS | `refresh-rotation.spec.ts` > "rotates a valid issued token: issues a new pair + marks the old rotated" + integration "issues, rotates, and revokes a family" |
| Refresh Token Rotation | Reuse detected revokes family | PASS | `refresh-rotation.spec.ts` > "detects reuse of an already-rotated token and revokes the whole family" + integration "family revoke persists all `revoked`" |
| Refresh Token Rotation | Expired or invalid refresh rejected | PASS | `refresh-rotation.spec.ts` > "rejects an expired refresh token" + "rejects an unknown refresh token" + "rejects a revoked token" |
| Access Token Delivery | Access token usable for authenticated calls | PASS | `auth-guard.spec.ts` > "allows a valid token whose user still exists" (request.user populated) + integration JwtTokenSigner roundtrip |
| Access Token Delivery | Expired access token | PASS | `auth-guard.spec.ts` > "rejects a foreign/expired token (verifyAccess throws) with UnauthenticatedError" |
| Logout / Revocation | Logout revokes family | PASS | `refresh-rotation.spec.ts` > "revokes the family of a presented refresh token…" + `auth-controller.spec.ts` > "clears BOTH the refresh and access cookies after calling the logout use case" (cookie cleared) |
| Password Hashing Port | Plaintext never persisted | PASS | `registration-login.spec.ts` > "never persists plaintext (password stored hashed, not as the raw input)" + integration `Argon2PasswordHasher` roundtrip |

**backend-identity compliance**: 12/12 scenarios PASS.

### backend-game-records

| Requirement | Scenario | Status | Evidence |
|-------------|----------|--------|----------|
| Persist Game Record | Successful persistence | PASS | `game-records.integration.spec.ts` > "saves a record and reads it back by the same userId" (uses domain `GameRecord`/`Score` VOs, server-assigned timestamp/id verified in `score-game-record.spec.ts`) |
| Persist Game Record | Unauthenticated request rejected | PASS | `auth-guard.spec.ts` > "rejects a request with no token at all with UnauthenticatedError" (AuthGuard is the single chokepoint for the persist endpoint) |
| Persist Game Record | Negative score rejected | PASS | `score-game-record.spec.ts` > "rejects a negative score with ValidationError" (Score VO) |
| Persist Game Record | Non-existent user rejected | PASS | `auth-guard.spec.ts` > "rejects a valid token for a DELETED user with NonExistentUserError (CRITICAL)" (CRITICAL design Decision #5) |
| High Score per Player | Retrieve high score | PASS | `game-records.integration.spec.ts` > "highScoreOf returns the max (3000 over 1200/950)" |
| High Score per Player | No records exist | PASS | `game-records.integration.spec.ts` > "highScoreOf returns null when the player has no records" |
| List Game Records by User | Paginated history | PASS | `game-records.integration.spec.ts` > "saves a record and reads it back…" + pagination (`page`/`total`/`hasMore` shape) verified in past-end tests |
| List Game Records by User | Cannot access other players' records | PASS | `game-records.integration.spec.ts` > "cannot access other players' records — listByUser is scoped to the caller only" (userA's records invisible to userB) |
| List Game Records by User | Pagination past the end | PASS | `game-records.integration.spec.ts` > "listByUser returns an empty page when paginated past the end" (5 records, offset 10 → empty, total 5) + "page-2/size-10 is empty also for a user with zero records" |
| Authentication Enforcement | Token from another context rejected | PASS | `auth-guard.spec.ts` > "rejects a foreign/expired token (verifyAccess throws) with UnauthenticatedError" (AuthGuard verifies via TokenVerifier port; foreign tokens throw) |

**backend-game-records compliance**: 10/10 scenarios PASS.

---

## Design Coherence

| # | Decision | Status | Evidence |
|---|----------|--------|----------|
| 1 | Manual PIXI.Application + React overlay (not @pixi/react) | PASS | `frontend/src/game/engine/{Engine,TickerLoop,WorldContainer}.ts` import `pixi.js` directly (`Application`/`Container`/`Ticker`); no `@pixi/react` in `package.json` deps. React overlay in `src/ui/screens/*.tsx` + `hud/HUD.tsx`. |
| 2 | MikroORM (Postgres, Data Mapper + UoW) | PASS | `backend/package.json` deps: `@mikro-orm/core` ^6.3.13, `@mikro-orm/postgresql`, `@mikro-orm/nestjs`, `@mikro-orm/migrations`. Adapters in `infrastructure/persistence/*Adapter.ts`; 2 migrations. testcontainers ran against Postgres. |
| 3 | Argon2id behind PasswordHasher port | PASS | `backend/src/contexts/identity/application/ports/PasswordHasher.port.ts` (interface) + `infrastructure/crypto/Argon2PasswordHasher.ts` (argon2 ^0.41.1). Integration test verifies hash/verify roundtrip. |
| 4 | Refresh rotation + family detection + per-user lock, statuses `issued\|rotated\|revoked` | PASS | `RefreshToken.ts` use case: status branch `issued→rotate`, `rotated→revokeFamily`, `revoked→reject`. Lock acquired + `try/finally` release (per W4). `RefreshTokenRow.status` typed `'issued'\|'rotated'\|'revoked'`. Lock torture test passes. |
| 5 | UserExists + AuthGuard CRITICAL | PASS | `shared/UserExists.port.ts` + `shared/AuthGuard.ts`: verify token → `userExists.exists(userId)` → `false` throws `NonExistentUserError`. Dedicated CRITICAL unit test for deleted-user path. No FK 500 surface. |
| 6 | Access token both header + httpOnly cookie | PASS | `AuthGuard.extractToken` reads Bearer header first, falls back to `ACCESS_TOKEN_COOKIE` (verified by cookie unit test). `AuthController` sets/clears both cookies. FE `client.ts` attaches Bearer header + `credentials: 'include'`. |
| 7 | 3 independent repos | PASS | `backend/.git`, `frontend/.git`, `openspec/.git` all exist with independent commit histories (no shared remote). |
| 8 | ESLint no-restricted-imports layer guard | PASS | `.eslintrc.cjs` overrides: domain→none (`@nestjs/*`, `@mikro-orm/*`, infra, app), application→domain (no framework/infra), shared-kernel ports/framework-agnostic except AuthGuard/Filter exempted. `npx eslint .` exit 0 — no layer leakage anywhere. |
| 9 | 3-layer FE state (Pixi per-frame / Zustand slow / React UI bidirectional) | PASS | `gameStore.ts` holds ONLY slow state (phase/score/health/isAuthenticated/playStartedAt). Per-frame isolation test proves no x/y/vx/vy/facing leak after 100 ticks. Pixi→store (CollisionSystem sets), React→store (Start/Pause buttons), Engine subscribes to store.phase. Bridge contract test passes exactly one `set` per change event. |
| 10 | Typed domain exceptions + NestJS filter | PASS | `shared/errors/*` defines `DomainError` + 6 subclasses. `shared/DomainExceptionFilter.ts` maps each to HTTP status (409/422/401/401/401/401) via `@Catch(DomainError)`; unknown → 500. Registered globally in `main.ts`. |

---

## Issues

### CRITICAL
None.

### WARNING
1. **Live FE↔BE integration not exercised (mocked-only E2E)**: the Playwright E2E suite mocks ALL backend endpoints via `page.route()`; no test runs the real NestJS server + Postgres + browser together. This is an accepted v1 scope decision (documented in apply-progress deviations 1-3), and `npm audit` transitive vulnerabilities from PR 5 remain unactioned. Both are post-verify/deploy concerns, not apply gates — but a real FE↔BE smoke run before production is recommended.
2. **`window.__gameStore` dev-only test hook in `main.tsx`**: a `import.meta.env.DEV`-guarded exposure of the Zustand store on `window` (apply-progress deviation 1). Guard body is dead code in production builds (Vite inlines `DEV=false`); harmless but a documented addition not in the original design File Changes list.
3. **E2E login-failure message drift**: the client's refresh-on-401 path surfaces "Session expired" on a login 401 (because mocked `/auth/refresh` also 401s) rather than the backend's "Invalid credentials". The spec only mandates "displays an error without changing phase" (no exact text) — satisfied. Cosmetic: a future client fix should skip refresh-on-401 for `/auth/login` + `/auth/register` (no session yet).

### SUGGESTION
1. JWT_ACCESS_SECRET dev-fallback `console.warn` fires per `JwtTokenSigner` instantiation in tests. Consider gating the warning behind a single process-level flag to keep test output quiet in CI while still warning in dev boots.
2. Coverage threshold in `openspec/config.yaml` is 0 (no coverage tooling enforced). Once the codebase stabilizes, consider adding `jest --coverage` + a realistic threshold and a Vitest coverage report to the verify gates.
3. `npm audit` transitive vulnerabilities (from PR 5 toolchain) — non-blocking for MVP but should be triaged before any public deploy.

---

## Final Verdict

**PASS WITH WARNINGS**

All 30/30 tasks complete; all build/type/lint gates exit 0; all 66 tests pass (46 Jest including 2 testcontainers Postgres suites, 15 Vitest, 5 Playwright) with real captured exit codes. Every spec scenario across all 3 domains (frontend-game-client 16/16, backend-identity 12/12, backend-game-records 10/10) has a passing covering test mapped to a spec scenario. All 10 design decisions are coherent with the actual code. Warnings are accepted-scope deviations (mocked-only E2E, the dev-only test hook, login-failure message drift) explicitly documented in the apply-progress and do not break any spec contract. Ready for `sdd-archive`.