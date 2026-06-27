# Design: jet-types-and-movement

> Resolves the 2 flagged items. **(1) Damage tuning**: enemy health = 100; Interceptor dmg 30 (⌈100/30⌉=4 hits), Balanced 45 (3 hits), Heavy 80 (2 hits) — all in required ranges. **(2) Acceleration model**: exponential Euler `currentSpeed += (target - currentSpeed) * k * (dt/1000)`, k per-second. The locked 0.004/0.005/0.006 were the exploration's per-ms draft; with the `/1000` formula k must rescale ×1000 → 4.0/5.0/6.0 (identical feel: 0.004/ms × 16.67ms = 4.0/s × 0.01667s = 0.0667/frame). This unit conversion IS the documented unit trap.

## Technical Approach

Extends the MVP per proposal + 3 delta specs. **Backend**: new `jet-types` bounded context mirroring `game-records` (aggregate + VOs + port + `ListJetTypes` use case + `@Public` controller + module + Data Mapper + seed migration). A `JetTypeExists` query port (mirrors the shared `UserExists` pattern from project-bootstrap Decision #5) lets `PersistGameRecord` reject unknown `jetTypeId` with a clean `ValidationError` (422) — no FK 500. **Frontend**: `MovementSystem` rewritten with the exponential Euler integrator (cruise-in-last-direction replaces freeze); `Enemy` multi-hit (health 100); `CollisionSystem` applies `jet.damage` on projectile hit and `defense` % on contact; `MenuScreen` adds 3-card selection; `gameStore` carries `selectedJetTypeId` + cached stats; `saveGameRecord` carries `jetTypeId`. FE/BE concerns kept separate per `rules.design`.

## Architecture Decisions

| # | Decision | Choice | Alternatives | Rationale |
|---|----------|--------|--------------|-----------|
| 1 | Jet types backend | New `jet-types` bounded context | Extend `game-records`; FE constants only | Repo commits to 1-context-per-domain (identity ≠ game-records); a static read-catalog must not couple to a per-user write model; user requires DB registration. Mirrors `game-records` shape verbatim. |
| 2 | Seed data | Migration with FIXED UUIDs | Admin CRUD; auto-generated IDs | Seed-only (admin CRUD is a non-goal); fixed UUIDs → FE fallback constants + FK default match across envs. |
| 3 | Acceleration model | Exponential Euler `currentSpeed += (target - currentSpeed) * k * (dt/1000)` | Closed-form `v(t)=max·(1-e^(-kt))`; linear ramp | Frame-rate-independent; stable (no overshoot); symmetric ramp/decay via one `k`; matches "exponential" wording in spec. |
| 4 | No-input behavior | Cruise in last held direction at `cruiseSpeed` | Freeze; auto-stop | Spec MODIFIED requirement explicitly replaces the freeze; `lastDirection` tracked on the Jet. |
| 5 | Enemy health | Fixed 100; `jet.damage` per hit | Variable enemy health | One tunable; hit-count math works out (below); enables multi-hit without enemy variety (non-goal). |
| 6 | Defense formula | `actualDamage = ENEMY_CONTACT_DAMAGE * (1 - defense/100)` % reduction | Flat reduction; immunity chance | Spec-mandated; scales cleanly across defense 0-100; no negative-flooring edge. |
| 7 | `jet_type_id` on records | NOT NULL FK, default Balanced UUID | Required no default; separate table | Existing rows backfilled to Balanced; new records validated by use case (422 on unknown) — default is a safety net, not the validation path. |
| 8 | Jet selection UI | Inline 3-card in `MenuScreen` | Separate `JetSelectScreen` | "Fixed for session" → selection once per menu visit; 3 cards don't justify a route. |

## Data Flow

**(a) Jet selection** (React→Store→Pixi):
```
MenuScreen mount ─> GET /jet-types (@Public) ─ok─> 3 cards
                                  └─fail─> FALLBACK_JET_TYPES (constants.ts)
player clicks card ─> store.set({ selectedJetTypeId, jetStats }) ─> Start Game enabled
```

**(b) Acceleration + movement** (Pixi ticker):
```
tick(dt) ─> read WASD/arrows → (dx,dy); normalize if present → lastDirection + facing
         ─> accelerating = isShift() && hasInput
         ─> target = accelerating ? maxSpeed : cruiseSpeed
         ─> if (hasInput || hasLastDir): currentSpeed += (target - currentSpeed) * k * (dt/1000)
         ─> vx = lastDirection.dx * currentSpeed;  vy = lastDirection.dy * currentSpeed
         ─> jet.update(dt) ─> clamp ─> worldContainer.follow+clamp
no input + hasLastDir → target=cruiseSpeed, cruises in lastDirection (does NOT freeze)
spawn (no lastDir)    → no integration, currentSpeed stays 0, jet stationary
```

**(c) Combat (multi-hit + defense)**:
```
projectile ∩ enemy ─> enemy.takeDamage(projectile.damage) ─> projectile.deactivate()
                       ├─ health<=0 → enemy removed + score += SCORE_PER_KILL
                       └─ health>0  → enemy survives (multi-hit)
enemy ∩ jet ─> actualDamage = ENEMY_CONTACT_DAMAGE * (1 - jet.defense/100)
             ─> jet.takeDamage(actualDamage) ─> enemy removed ─> health<=0 → phase=gameOver
```

**(d) Score persistence with jetTypeId**:
```
gameOver ─> POST /game-records { score, durationMs, jetTypeId } + Bearer
         ─> AuthGuard (verify + UserExists) ─> PersistGameRecord
         ─> JetTypeExists(jetTypeId)? false → ValidationError 422 (no FK 500)
         ─> Score VO (rejects negative) ─> GameRecord(+jetTypeId) ─> repo.save ─> 201
```

## File Changes

| File | Action | Description |
|------|--------|-------------|
| `backend/src/contexts/jet-types/domain/JetType.ts` | Create | Aggregate (`create`/`rehydrate`), holds VOs |
| `backend/src/contexts/jet-types/domain/vo/{Speed,CruiseSpeed,AccelerationRate,Defense,Damage}.ts` | Create | VOs with validation (mirrors `Score`) |
| `backend/src/contexts/jet-types/application/ports/{JetTypeRepository,JetTypeExists}.port.ts` | Create | `findAll`/`findById` + `exists(id)` (FK validation) |
| `backend/src/contexts/jet-types/application/usecases/ListJetTypes.ts` | Create | `GET /jet-types` use case |
| `backend/src/contexts/jet-types/infrastructure/persistence/{JetTypeEntity,mappers,JetTypeRepositoryAdapter}.ts` | Create | MikroORM entity + Data Mapper + adapter (impls both ports) |
| `backend/src/contexts/jet-types/infrastructure/controllers/JetTypesController.ts` | Create | `@Public() GET /jet-types` |
| `backend/src/contexts/jet-types/infrastructure/nest-jet-types.module.ts` | Create | Module wiring (exports `JetTypeExists`) |
| `backend/src/migrations/Migration20260626000002_jet_types.ts` | Create | `jet_types` table + 3 seed rows (fixed UUIDs) |
| `backend/src/migrations/Migration20260626000003_game_records_jet_type_fk.ts` | Create | Add `jet_type_id` NOT NULL FK default Balanced |
| `backend/src/contexts/game-records/domain/GameRecord.ts` | Modify | + `jetTypeId` field + `create`/`rehydrate` signature |
| `backend/src/contexts/game-records/infrastructure/persistence/{GameRecordEntity,mappers}.ts` | Modify | + `jetTypeId` column + mapper round-trip |
| `backend/src/contexts/game-records/application/usecases/PersistGameRecord.ts` | Modify | + `jetTypeId` cmd + `JetTypeExists` validation (422) |
| `backend/src/contexts/game-records/infrastructure/controllers/GameRecordsController.ts` | Modify | DTO + response include `jetTypeId` |
| `backend/src/contexts/game-records/infrastructure/nest-game-records.module.ts` | Modify | Import `JetTypesModule` for `JetTypeExists` |
| `backend/src/app.module.ts` | Modify | Register `JetTypesModule` |
| `frontend/src/game/entities/Jet.ts` | Modify | Replace `speed` with `maxSpeed/cruiseSpeed/accelerationRate/defense/damage`; + `currentSpeed`, `lastDirection` |
| `frontend/src/game/entities/Enemy.ts` | Modify | Health default 100 (from `ENEMY_MAX_HEALTH`); `takeDamage(amount)` already exists |
| `frontend/src/game/entities/Projectile.ts` | Modify | + `damage` property (from jet at spawn) |
| `frontend/src/game/systems/MovementSystem.ts` | Rewrite | Exponential Euler + cruise-in-last-direction |
| `frontend/src/game/systems/CollisionSystem.ts` | Modify | Multi-hit (`projectile.damage`); defense % contact |
| `frontend/src/game/systems/ShootingSystem.ts` | Modify | Spawn `Projectile` with `jet.damage` |
| `frontend/src/game/input/KeyboardInput.ts` | Modify | + `isAccelerate()` (Shift); add Shift to `GAME_KEYS` |
| `frontend/src/game/state/gameStore.ts` | Modify | + `selectedJetTypeId`, `jetStats` |
| `frontend/src/game/constants.ts` | Modify | `ENEMY_MAX_HEALTH=100`; + `FALLBACK_JET_TYPES` (sync with seed) |
| `frontend/src/game/engine/GameSystems.ts` | Modify | Read `jetStats` at spawn; construct `Jet` with stats |
| `frontend/src/ui/screens/MenuScreen.tsx` | Modify | 3-card selection; fetch `getJetTypes` (+ fallback) |
| `frontend/src/ui/screens/GameOverScreen.tsx` | Modify | Pass `selectedJetTypeId` to `saveGameRecord` |
| `frontend/src/ui/api/client.ts` | Modify | + `getJetTypes`, `JetTypeDto`; `saveGameRecord(+jetTypeId)` |

## Interfaces / Contracts

**Authoritative seed-values table** (supersedes the proposal's per-ms draft — use THESE in the migration):
```
| Jet         | maxSpeed | cruiseSpeed | accelerationRate (per SECOND) | defense | damage |
|-------------|----------|-------------|-------------------------------|---------|--------|
| Interceptor | 460      | 200         | 4.0                           | 10      | 30     |
| Balanced    | 360      | 200         | 5.0                           | 35      | 45     |
| Heavy       | 280      | 180         | 6.0                           | 60      | 80     |
```
Also set `ENEMY_MAX_HEALTH = 100` (was 1 in project-bootstrap).

**Fixed seed UUIDs** (FE fallback + migration + FK default MUST match — a test asserts equality):
```
Interceptor  00000000-0000-4000-8000-000000000001
Balanced     00000000-0000-4000-8000-000000000002   ← FK default
Heavy        00000000-0000-4000-8000-000000000003
```

**Damage + health math** (enemy health = 100):
```
Interceptor dmg 30: ⌈100/30⌉ = 4 hits   (range 3-4 ✓)
Balanced    dmg 45: ⌈100/45⌉ = 3 hits   (range 2-3 ✓)
Heavy       dmg 80: ⌈100/80⌉ = 2 hits   (range 1-2 ✓)
Defense (contact, ENEMY_CONTACT_DAMAGE=20):
  Interceptor def 10 → 18 dmg/contact  (100/18 ≈ 6 contacts to die)
  Balanced    def 35 → 13 dmg/contact  (100/13 ≈ 8 contacts)
  Heavy       def 60 →  8 dmg/contact  (100/8  ≈ 13 contacts)
```

**Acceleration integrator** (MovementSystem):
```ts
// dt = Pixi Ticker.deltaMS (ms). k = jet.accelerationRate, stored PER SECOND.
// UNIT TRAP: the exploration drafted k per-ms (0.004); the /1000 formula needs
// per-second k → rescale ×1000: 0.004→4.0, 0.005→5.0, 0.006→6.0.
// Identical feel: 0.004/ms × 16.67ms = 4.0/s × 0.01667s = 0.0667 closure/frame.
const target = (input.isAccelerate() && hasInput) ? jet.maxSpeed : jet.cruiseSpeed;
if (hasInput || hasLastDir) {
  jet.currentSpeed += (target - jet.currentSpeed) * jet.accelerationRate * (dt / 1000);
}
jet.vx = jet.lastDirection.dx * jet.currentSpeed;
jet.vy = jet.lastDirection.dy * jet.currentSpeed;
// Time to 95% of target: Interceptor(k=4)≈0.75s, Balanced(k=5)≈0.60s, Heavy(k=6)≈0.50s.
```

**Defense formula** (CollisionSystem):
```ts
const actualDamage = ENEMY_CONTACT_DAMAGE * (1 - jet.defense / 100);
```

**JetType aggregate + VOs** (domain, mirrors `GameRecord`/`Score`):
```ts
class JetType extends AggregateRoot<string> {
  readonly name: string;
  readonly maxSpeed: Speed; readonly cruiseSpeed: CruiseSpeed;
  readonly accelerationRate: AccelerationRate; readonly defense: Defense; readonly damage: Damage;
  static create(...): JetType;  static rehydrate(props): JetType;
}
// VO invariants (throw ValidationError, from @shared/errors):
//   Speed: int >0 | CruiseSpeed: int >0, <maxSpeed | AccelerationRate: number >0
//   Defense: int [0,100] | Damage: int >0
//   Aggregate enforces: maxSpeed > cruiseSpeed > 0  (spec "Jet Type Properties Validation")
```

**Ports** (application, framework-agnostic — mirrors `GameRecordRepository`):
```ts
interface JetTypeRepository extends Port {
  findAll(): Promise<JetType[]>;
  findById(id: string): Promise<JetType | null>;
}
interface JetTypeExists extends Port {  // validates FK in PersistGameRecord (mirrors UserExists)
  exists(id: string): Promise<boolean>;
}
```

**JetTypeEntity** (MikroORM, snake_case via UnderscoreNamingStrategy):
```ts
@Entity({ tableName: 'jet_types' })
class JetTypeEntity {
  @PrimaryKey({ type: 'uuid' }) id!: string;
  @Property() name!: string;
  @Property() maxSpeed!: number;
  @Property() cruiseSpeed!: number;
  @Property() accelerationRate!: number;  // numeric — k is fractional
  @Property() defense!: number;
  @Property() damage!: number;
  @Property() createdAt!: Date;
}
```

**Frontend Jet** (modified — per-frame state stays in the entity per Decision #9):
```ts
class Jet {
  maxSpeed; cruiseSpeed; accelerationRate; defense; damage: number;  // from selected type
  currentSpeed = 0;             // per-frame, NOT in store
  lastDirection = { dx: 0, dy: 0 };  // per-frame; {0,0} until first input
  // x/y/vx/vy/facing/health retained; update(dt)/takeDamage()/nose()/facingVector() unchanged
}
```

**gameStore additions** (slow-state only):
```ts
selectedJetTypeId: string | null;
jetStats: JetTypeDto | null;   // cached stats of the selected type
```

## Testing Strategy

| Layer | What | Approach |
|-------|------|----------|
| Unit BE | VO validation (maxSpeed>cruiseSpeed>0, defense [0,100], damage/accel >0); `ListJetTypes` returns 3; `JetTypeExists` true/false | Jest + in-memory fakes |
| Unit BE | `PersistGameRecord` valid jetTypeId ok; invalid → `ValidationError`; negative score → `ValidationError` | Jest + `JetTypeExists` fake |
| Integration BE | Seed migration → 3 rows with fixed UUIDs; `GET /jet-types` 200 no auth; `game_records.jet_type_id` FK + default Balanced on existing rows | testcontainers Postgres |
| Unit FE | Exponential accel: hold ramps toward max; release decays toward cruise; cruise-in-last-direction (no input → keeps last dir at cruise, does NOT freeze); diagonal normalize × currentSpeed | Vitest, assert `currentSpeed`/`vx`/`vy` |
| Unit FE | Enemy multi-hit: `jet.damage` per hit; survives if health>damage; destroyed + score at health<=0 | Vitest |
| Unit FE | Defense %: contact damage reduced by `defense/100` | Vitest |
| Unit FE | Jet selection: 3 cards render; click sets `selectedJetTypeId`+`jetStats`; Start disabled without selection; fetch failure → fallback constants | Vitest + RTL |
| E2E FE | select jet → start → kill enemy (multi-hit) → gameOver → score saved with `jetTypeId` | Playwright (no canvas pixel asserts) |

## Migration / Rollout

Greenfield addition. **Order is critical**: `Migration20260626000002_jet_types` (table + 3 seed rows) MUST run BEFORE `Migration20260626000003_game_records_jet_type_fk` (the FK references the seeded rows). Existing `game_records` rows get `jet_type_id = Balanced UUID` via NOT NULL DEFAULT. Rollback: revert both migrations, drop `JetTypesModule` from `app.module`, restore game-records DTO/aggregate. Static seeds — no user data lost.

## Open Questions

None. Damage tuning + enemy health (item 1) and the acceleration formula + unit trap (item 2) are both resolved above.
