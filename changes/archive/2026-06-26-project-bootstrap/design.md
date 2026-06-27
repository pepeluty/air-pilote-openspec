# Design: project-bootstrap

> Greenfield MVP. All paths CREATE. Resolves spec risks R1 (refresh family), R2 (FE modules), R3 (password policy) + review fixes (1 CRITICAL, 8 WARN).

## Technical Approach

Two repos from zero. FE: manual `PIXI.Application` + React overlay — Pixi owns canvas/ticker + per-frame transforms, React owns HUD/menus/auth, Zustand bridges slow state (score/health/phase) **both directions**. BE: NestJS + Postgres + MikroORM, hexagonal/DDD per bounded context (`identity`, `game-records`); ports = pure TS, adapters = NestJS/ORM/JWT/crypto; ESLint `no-restricted-imports` guards layers. **Cross-context user existence** verified via a shared-kernel `UserExists` query port invoked by the global `AuthGuard` (CRITICAL fix).

## Architecture Decisions

| # | Decision | Choice | Alternatives | Rationale |
|---|----------|--------|--------------|-----------|
| 1 | PixiJS integration | Manual `PIXI.Application` + React overlay | `@pixi/react`, Phaser | Pixi owns ticker, React owns DOM; no reconciler/double state |
| 2 | ORM | MikroORM (Postgres, Data Mapper + UoW) | TypeORM, Prisma | UoW fits DDD aggregates; Prisma schema-first clashes with rich domain |
| 3 | Password hashing | Argon2id behind `PasswordHasher` port | BCrypt, scrypt | OWASP-recommended, memory-hard; port swappable |
| 4 | Refresh rotation | Rotating + family detection; persisted status `issued\|rotated\|revoked` only (`revoked` = terminal) + per-user lock | Sliding/non-rotating | Bounds theft window; reuse is a **trigger** (transition `rotated→revoked`), not a status — no spurious enum (W1) |
| 5 | Cross-context user existence | `UserExists` query port in shared kernel, called by global `AuthGuard` (CRITICAL) | PersistGameRecord port call; FK+catch→domain error | One chokepoint for all protected endpoints; game-records stays decoupled from identity write model; clean 401, no FK 500 |
| 6 | Access token delivery | Authorization header **AND** httpOnly cookie | Header only | Header for API clients/tests; cookie for browser fetch w/ credentials |
| 7 | Repo layout | 3 independent repos (`frontend/`,`backend/`,`openspec/`) | Monorepo | Independent deploy/rollback; no shared runtime dep |
| 8 | Layer enforcement | ESLint `no-restricted-imports` (domain→none, app→domain, infra→all) | tsconfig paths | Fails build on ORM/Nest leakage into `application`/`domain` |
| 9 | FE state ownership | 3-layer: Pixi per-frame / Zustand slow / React UI (bidirectional) | Redux | Pixi emits change events → store; React UI writes phase commands → store → Pixi subscribes (W5) |
| 10 | Error model | Typed domain exception classes + NestJS exception filters → HTTP | `Result<T,E>` return types | Nest filter pipeline maps domain errors to HTTP cleanly; no unwrap tax at every call site (W7) |

## Data Flow

**(a) Game loop** (Pixi→Store→React):
```
Input ──> InputSystem ──> TickerLoop.update(dt)
  └> MovementSystem ─> Jet.update ─> WorldContainer.follow+clamp
  └> ShootingSystem ─> Projectile.spawn (cooldown; blocked unless phase==='playing')
  └> EnemyAISystem/SpawnSystem ─> Enemy.update
  └> CollisionSystem ─> overlap? enemy destroyed (+score) | player damaged (-health)
        └> on change event ─> Zustand set(score|health|phase)  // NOT per-frame ─> HUD re-render
        └> health===0 ─> set(phase:'gameOver'); stop ticker
```

**(a') UI command path** (React→Store→Pixi, W5):
```
Start button ─> store.set({phase:'playing'})    Pause button/Esc ─> store.set({phase:'paused'})
Engine subscribes to store.phase:
  'playing' ─> ticker.start(); spawn Jet; clear enemies
  'paused'  ─> ticker.stop() (state retained on stage)
  'gameOver' ─> ticker.stop(); POST /game-records
```

**(b) Auth refresh rotation** (R1 — persisted status `issued | rotated | revoked`; reuse is a TRIGGER not a status). State machine lives in the `RefreshToken` use case (NOT the port). Lock acquire/release MUST be wrapped in **try/finally** (W4):
```
POST /auth/refresh (cookie R)
  └> acquire lock userId                      // try {
  └> row = store.findByHash(R)
        ├─ null/expired ─> reject 401 invalid-credentials   ... finally release lock
        ├─ status='issued'  ─> store.markRotated(R.id, newHash); issue pair (same familyId, parent=R); set cookie; 200
        └─ status='rotated'  ─> REUSE: store.revokeFamily(familyId) (R,R2,.. → status='revoked'); 401; no tokens
  └> finally release lock                     // }
```
State machine: `issued --rotate--> rotated --(reuse of any descendant)--> revoked`; `issued --(reuse)--> revoked`.

**(c) Score persistence** (CRITICAL — user-existence check; satisfies spec "Non-existent user rejected"):
```
gameOver ─> POST /game-records {score,durationMs} + access token
  └> AuthGuard: verifyAccess(t) ─> userId      (else 401 UnauthenticatedError)
  └> AuthGuard: UserExists(userId) ─> false ─> 401 NonExistentUserError   // CRITICAL: no FK 500
  └> PersistGameRecord ─> Score VO (rejects negative → ValidationError) ─> GameRecord aggregate
  └> GameRecordRepository.save ─> MikroORM UoW ─> Postgres
  └> 201 {id, score, durationMs, timestamp}
```

## File Changes

| File | Action | Description |
|------|--------|-------------|
| `frontend/{package.json,vite.config.ts,tsconfig.json,.eslintrc.cjs}`, `src/main.tsx`, `App.tsx` | Create | Vite+React+TS+PixiJS+Zustand toolchain, entry, screen router |
| `frontend/src/game/{engine/{Engine,TickerLoop,WorldContainer},entities/{Jet,Projectile,Enemy},systems/*,state/gameStore}.ts` | Create | Engine (PIXI.Application, ticker, camera, **store.phase subscription**), entities, systems, Zustand bridge (R2) |
| `frontend/src/ui/{screens/{Auth,Menu,GameOver,Paused}Screen.tsx,hud/HUD.tsx}`, `api/client.ts` | Create | DOM overlay UI (**+Pause screen/trigger**) + fetch client (token attach/refresh) |
| `backend/{package.json,tsconfig.json,.eslintrc.cjs}`, `src/{main.ts,app.module.ts}`, `shared/{domain,application}/*` | Create | NestJS+MikroORM, layer guard, base Entity/VO/UseCase, shared kernel |
| `backend/src/shared/{UserExists.port.ts,AuthGuard.ts,errors/*}` | Create | **Shared kernel**: `UserExists` query port, global AuthGuard (decode + UserExists), typed domain errors (CRITICAL, W7) |
| `backend/src/contexts/identity/domain/{User,RefreshToken,PasswordPolicy}.ts`, `vo/{Email,Credentials,AuthToken}.ts` | Create | identity aggregate + VOs + R3 policy; state machine in RefreshToken use case |
| `backend/src/contexts/identity/application/{ports/*,usecases/{RegisterUser,LoginUser,RefreshToken,Logout}}.ts` | Create | identity **behavior ports** + use cases (R1 family logic + **try/finally lock**) |
| `backend/src/contexts/identity/infrastructure/*`, `migrations/*` | Create | Nest module, AuthController, MikroORM repos (impl `UserExists`), Argon2+JWT adapters, ORM entities, migrations |
| `backend/src/contexts/game-records/domain/{GameRecord,vo/Score}.ts` | Create | game-records aggregate + non-negative Score VO |
| `backend/src/contexts/game-records/application/{ports/GameRecordRepository,usecases/{PersistGameRecord,GetHighScore,ListGameRecords}}.ts` | Create | game-records port + use cases |
| `backend/src/contexts/game-records/infrastructure/*` | Create | Nest module, controller, MikroORM repo, ORM entity |

## Interfaces / Contracts

```ts
// --- Shared kernel (cross-context) ---
interface UserExists { exists(userId: string): Promise<boolean>; } // CRITICAL: identity adapter implements it (read-model)

// --- Error model (typed exceptions; NestJS exception filter maps to HTTP) — W7 ---
class DomainError extends Error { constructor(readonly code:string){super(code);} }
class DuplicateEmailError      extends DomainError {} // 409
class PasswordStrengthError    extends DomainError {} // 422
class ValidationError          extends DomainError {} // 422 (negative score, malformed body)
class InvalidCredentialsError  extends DomainError {} // 401 login / expired refresh
class UnauthenticatedError     extends DomainError {} // 401 no/expired/foreign access token
class NonExistentUserError     extends DomainError {} // 401 jwt ok, user gone (CRITICAL)

// --- identity ports (behavior-oriented; state machine lives in use case, NOT the port) — W3 ---
interface PasswordHasher { hash(p:string):Promise<string>; verify(p:string,h:string):Promise<boolean>; }
interface TokenSigner { signAccess(p:{userId:string}):string; signRefresh():{token:string;hash:string}; verifyAccess(t:string):{userId:string}; }
interface RefreshTokenStore {
  issue(userId:string, familyId:string, parentHash:string|null):Promise<{token:string;hash:string;row:RefreshTokenRow}>;
  findByHash(hash:string):Promise<RefreshTokenRow|null>;
  markRotated(id:string, newHash:string):Promise<void>;
  revokeFamily(familyId:string):Promise<void>;
  acquireLock(userId:string):Promise<LockHandle>; releaseLock(h:LockHandle):Promise<void>;
}
interface UserRepository { findByEmail(e:string):Promise<User|null>; save(u:User):Promise<void>; findById(id:string):Promise<User|null>; }

interface RefreshTokenRow { // W2 — defined
  id:string; userId:string; familyId:string; parentTokenHash:string|null;
  hash:string; status:'issued'|'rotated'|'revoked'; expiresAt:Date; createdAt:Date;
}
interface LockHandle { userId:string; acquiredAt:number; } // W2 — defined

// --- game-records port ---
interface GameRecordRepository {
  save(r:GameRecord):Promise<GameRecord>;
  highScoreOf(userId:string):Promise<number|null>;
  listByUser(userId:string, opts:PageOpts):Promise<Page<GameRecord>>;
}

// R3 password policy (PasswordPolicy.ts): min 8, max 72, >=1 letter AND >=1 number; satisfies(p):boolean
// FE slow-state (R2) + command path (W5):
interface GameStore { phase:'menu'|'playing'|'paused'|'gameOver'; score:number; health:number; set(p:Partial<GameStore>):void; }
// Pixi system iface: update(dt:number, ctx:GameContext):void;  Engine subscribes to store.phase
```

## Testing Strategy

| Layer | Spec scenarios covered | Approach |
|-------|------------------------|----------|
| Unit BE | Registration (success / duplicate-email / weak-pw), Login (success / invalid-creds), Rotation (success / reuse→family revoke / expired), Logout (revoke family + clear cookie), plaintext-never-stored, Score non-negative, GameRecord aggregate | Jest + in-memory fakes |
| Unit BE | **AuthGuard: valid token+existing user→ok; valid token + DELETED user → NonExistentUserError 401 (CRITICAL); foreign/expired token → UnauthenticatedError** (spec: Token-from-other-context, Expired-access) | Jest + UserExists fake |
| Unit FE | Diagonal move + normalized magnitude, no-input hold, fire cooldown, **fire-blocked-outside-playing**, enemy destroyed +score, **enemy damages player -health**, **camera follow**, **clamp at edge**, **health→0→phase gameOver**, **start game menu→playing**, **pause trigger**, **slow-state publish (1 write/change)**, **per-frame data isolation (store NOT touched per frame)** | Vitest |
| Integration BE | MikroORM repos vs Postgres: persistence, paginated history (page-2/size-10→empty, past-end), high score (3000 over 1200/950), no-records→null, cannot-access-others, family revoke persists all rows `revoked`, UserExists read-model, Argon2/JWT adapters, **lock release on error path (try/finally torture test)** | testcontainers Postgres |
| E2E FE | Register success→menu, login failure error, menu→play→gameOver→score saved, pause/resume, HUD score+health update | Playwright (no canvas pixel asserts v1) |

## Migration / Rollout

No migration required (greenfield). Bootstrap order: (1) `git init` 3 repos; (2) backend scaffolding + identity context + **shared kernel (`UserExists`, `AuthGuard`, typed errors)**; (3) game-records context; (4) frontend toolchain + engine (with `phase` subscription); (5) systems + HUD + auth/pause UI; (6) FE↔BE integration. Each milestone = git checkpoint; `git reset --hard` rolls back per repo.

## Open Questions

- [ ] In-flight lock for v1: in-memory `Map<userId,Promise>` (single-instance) vs Postgres advisory lock (multi-instance). Recommend in-memory for MVP; document advisory lock as scale path. (Access-token delivery OQ resolved → Decision 6.)