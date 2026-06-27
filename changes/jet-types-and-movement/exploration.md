# Exploration: jet-types-and-movement

> Change: `jet-types-and-movement` · Project: Air-Pilote · Mode: hybrid (openspec + Engram)
> Scope: workspace-level (backend repo + frontend repo + shared openspec context)
> Architecture forks are LOCKED upstream (frontend stack, backend hex/DDD, 3 repos) — this exploration defines HOW to add 3 jet types + exponential acceleration + cruise speed, not WHETHER to re-debate the stack.
> Product decision LOCKED upstream: jet selection is PRE-GAME (player picks among 3 types before a match; fixed for the whole session).

## Current State

A working MVP exists from the `project-bootstrap` change. The relevant pieces this change must EXTEND (not greenfield):

### Backend (NestJS + PostgreSQL + MikroORM, hex/DDD per bounded context)
- **Two bounded contexts exist**: `identity` (auth/JWT, users + refresh_tokens) and `game-records` (game_records table).
- **Aggregate pattern** (`backend/src/contexts/game-records/domain/GameRecord.ts`): `AggregateRoot<string>` with `create()` factory (invariant enforcement) + `rehydrate()` (persistence reconstruction). Pure domain, no ORM decorators. Holds a `Score` VO.
- **VO pattern** (`...domain/vo/Score.ts`): extends `ValueObject`, private constructor, `static create(raw)` that throws `ValidationError` (from `@shared/errors`) on bad input, `equals()` value-based.
- **Entity pattern** (`...infrastructure/persistence/GameRecordEntity.ts`): MikroORM `@Entity`, snake_case columns via UnderscoreNamingStrategy, plain scalar columns (the Score VO is stored as a plain int).
- **Mapper pattern** (`...infrastructure/persistence/mappers.ts`): static `toDomain()` / `toPersistence()` — Data Mapper (design Decision #2). Domain never sees ORM types.
- **Repository port** (`...application/ports/GameRecordRepository.port.ts`): extends `Port`, application-layer, framework-agnostic.
- **Use case pattern** (`...application/usecases/*.ts`): extend `UseCase<TInput,TOutput>`, plain TS class (no `@Injectable`), instantiated in the module via `useFactory` with the adapter injected by concrete class token.
- **Module wiring** (`...infrastructure/nest-game-records.module.ts`): `MikroOrmModule.forFeature([Entity])`, controller + adapter + use-case factories + exports.
- **Controller pattern** (`GameRecordsController.ts`): `@Controller('game-records')`, ALL routes protected (no `@Public`); userId ALWAYS from `req.user` set by the global `AuthGuard` (never from body/query).
- **Auth**: global `AuthGuard` (`@shared/AuthGuard.ts`) with `@Public()` decorator for opt-out routes (used by identity register/login/refresh). `AuthenticatedUser = { userId }`.
- **Migrations**: 2 existing (`Migration20260626000000_initial.ts` = users + refresh_tokens; `Migration20260626000001_game_records.ts` = game_records). Snake_case, CHECK constraints for invariants, indexes on hot read paths. Snapshot file `.snapshot-air-pilote.json`.
- **GameRecord aggregate** currently stores: `id, userId, score (VO), durationMs, timestamp`. **No jet type reference yet.**

### Frontend (React + TS + PixiJS v8, manual Application + React overlay, Zustand slow-state bridge)
- **Jet entity** (`frontend/src/game/entities/Jet.ts`): owns Pixi `Graphics` (cyan triangle), per-frame `x/y/vx/vy/facing`, `health`, and a single `readonly speed: number` (constant magnitude). `update(dt)` integrates position; `takeDamage()`/`nose()`/`facingVector()` helpers. The constructor accepts `{ x?, y?, health?, speed? }` — so feeding per-type stats requires extending the options, not rewriting the entity.
- **MovementSystem** (`...systems/MovementSystem.ts`): each tick reads WASD/arrows into `(dx,dy)`, NORMALIZES the vector, multiplies by `jet.speed` (constant), sets `jet.vx/vy`, updates `facing`. **No-input → velocity zeroed** (jet stops dead). **This is the core file to rewrite** for acceleration: the constant-speed model becomes an exponential-ramp-toward-max + cruise baseline.
- **ShootingSystem** (`...systems/ShootingSystem.ts`): cooldown-gated, spawns `Projectile` at `jet.nose()` along `facing`. Projectile damage is implicit — CollisionSystem deals `1` damage per hit. **No per-jet `damage` stat wired in yet.**
- **CollisionSystem** (`...systems/CollisionSystem.ts`): (a) projectile↔enemy → `enemy.takeDamage(1)`, on death `score += SCORE_PER_KILL`; (b) enemy↔jet → enemy dies, `jet.takeDamage(ENEMY_CONTACT_DAMAGE)` (fixed constant `20`). **Defense is not applied; enemy health is a flat `1`.** Store writes only on change events.
- **constants.ts**: `JET_MAX_HEALTH=100`, `JET_SPEED=320`, `JET_RADIUS=18`, `ENEMY_CONTACT_DAMAGE=20`, `ENEMY_MAX_HEALTH=1`, `SCORE_PER_KILL=100`, fire/spawn/radii. All tuning in one place — natural home for the 3 jet-type stat triplets (or fetched from backend).
- **gameStore** (`...state/gameStore.ts`): Zustand, slow state only — `phase, score, health, isAuthenticated, playStartedAt, set()`. **No `selectedJetTypeId` / `jetStats` yet.** Per-frame data explicitly forbidden here (Design Decision #9).
- **Input** (`...input/KeyboardInput.ts`): `InputState` interface with `isUp/isDown/isLeft/isRight/isShoot`. Movement = WASD/arrows, Shoot = Space. **No acceleration key yet, and no `isBoost()`/`isAccelerate()`.** Space is taken by shoot → Shift is the natural accelerator key.
- **MenuScreen** (`...ui/screens/MenuScreen.tsx`): title, high score fetch on mount, Start Game button (flips phase + resets score/health + stamps `playStartedAt`), Logout. **No jet selection UI.**
- **GameOverScreen**: on mount POSTs `{ score, durationMs }` to `/game-records` via `api.saveGameRecord()`. **If we add `jetTypeId` to the record, both the DTO shape and this call change.**
- **App.tsx**: phase router with auth gate. `GameCanvas` mounts the `Engine` (on playing/paused), destroyed on unmount. Engine options (`mapWidth/mapHeight`) are hardcoded here.
- **Engine + GameSystems** (`...engine/Engine.ts`, `GameSystems.ts`): on phase→playing (from menu/gameOver), `GameSystems` is rebuilt: spawns the Jet at world centre with `JET_MAX_HEALTH` (hardcoded), attaches `KeyboardInput`, builds `GameContext`, registers the 5 systems. **The jet spawn is where the selected jet type's stats must be injected** — `GameSystems` needs the selected type (from the store) to construct the `Jet` with the right `maxSpeed/cruiseSpeed/accelerationRate/defense/damage`.
- **API client** (`...ui/api/client.ts`): `get/post/put/del` verbs with refresh-on-401, `ApiError`, module-memory access token, httpOnly refresh cookie. `getHighScore()`/`saveGameRecord()` are the game-records helpers. **No `getJetTypes()` helper yet** — trivial to add following the existing pattern.
- **GameContext** (`...game/GameContext.ts`): closure-injected bundle (`jet, worldContainer, projectiles, enemies, viewport, mapBounds, input`). **No jet stats reference** — systems read `jet.speed` directly; the Jet is the natural carrier of per-type stats.

### Existing specs (impact surface)
- `openspec/specs/frontend-game-client/spec.md` — "Jet Movement" requirement (fixed speed; no input → jet maintains position) and "Basic Enemy AI" (enemy dies in 1 hit, fixed contact damage) will be MODIFIED by this change. "Game Phases — Start game" scenario gains a jet-selection precondition.
- `openspec/specs/backend-game-records/spec.md` — the persist/list/high-score requirements will be MODIFIED to include `jetTypeId` on the record (ADDED field) and a new domain `backend-jet-types` (or sub-requirements inside game-records) for the catalog endpoint.
- `openspec/config.yaml` — `persistence: hybrid`, rules require Given/When/Then + RFC 2119, front/backend concerns kept separate in design.

## Affected Areas

### Backend (NEW bounded context + migration + record extension)
- `backend/src/contexts/jet-types/` — **NEW context** (domain + application + infrastructure), mirroring the game-records layout.
- `backend/src/contexts/jet-types/domain/JetType.ts` — **NEW aggregate** (or readonly catalog entity).
- `backend/src/contexts/jet-types/domain/vo/*.ts` — **NEW VOs** (Speed, Defense, Damage, AccelerationRate) following the `Score` VO pattern.
- `backend/src/contexts/jet-types/application/ports/JetTypeRepository.port.ts` — **NEW** outbound port.
- `backend/src/contexts/jet-types/application/usecases/ListJetTypes.ts` — **NEW** use case (`GET /jet-types`).
- `backend/src/contexts/jet-types/infrastructure/persistence/JetTypeEntity.ts` + mapper + `JetTypeRepositoryAdapter.ts` — **NEW**.
- `backend/src/contexts/jet-types/infrastructure/controllers/JetTypesController.ts` — **NEW** (public or authenticated route).
- `backend/src/contexts/jet-types/infrastructure/nest-jet-types.module.ts` — **NEW** module wiring.
- `backend/src/migrations/MigrationYYYYMMDD000002_jet_types.ts` — **NEW migration** (jet_types table + seed of 3 rows) + optional `game_records.jet_type_id` column on a second migration.
- `backend/src/contexts/game-records/domain/GameRecord.ts` — **MODIFIED** to carry `jetTypeId` (or a `JetType` reference VO), if we persist the chosen type on the record.
- `backend/src/contexts/game-records/infrastructure/persistence/GameRecordEntity.ts` + `mappers.ts` — **MODIFIED** for the new column.
- `backend/src/contexts/game-records/application/usecases/PersistGameRecord.ts` + controller DTO — **MODIFIED** to accept/validate `jetTypeId`.
- `backend/src/app.module.ts` (or equivalent root) — **MODIFIED** to register the new `JetTypesModule`.

### Frontend (game layer + UI)
- `frontend/src/game/entities/Jet.ts` — **MODIFIED** to carry `maxSpeed, cruiseSpeed, accelerationRate, defense, damage` (replace single `speed`), plus per-frame `currentSpeed` for the acceleration model.
- `frontend/src/game/systems/MovementSystem.ts` — **REWRITTEN** core: exponential ramp toward `maxSpeed` while accelerating; decay back to `cruiseSpeed` on release; diagonal normalization still applies on the direction, magnitude comes from the model.
- `frontend/src/game/systems/CollisionSystem.ts` — **MODIFIED** to apply `jet.defense` to incoming `ENEMY_CONTACT_DAMAGE`; apply `jet.damage` to enemy on projectile hit (if enemies gain health).
- `frontend/src/game/systems/ShootingSystem.ts` — **MODIFIED** if projectile damage derives from `jet.damage` (pass it into `Projectile`, or have CollisionSystem read `jet.damage`).
- `frontend/src/game/constants.ts` — **MODIFIED** (or superseded): jet stat source moves to backend catalog; constants keep engine tuning (radii, cooldowns, enemy health, spawn). Possibly per-type fallback defaults here.
- `frontend/src/game/input/KeyboardInput.ts` — **MODIFIED**: add `isShift()`/`isAccelerate()` (Shift key) to `InputState`; add Shift to `GAME_KEYS`.
- `frontend/src/game/state/gameStore.ts` — **MODIFIED**: add `selectedJetTypeId: string | null` (+ optional cached `jetTypes` catalog) to slow state.
- `frontend/src/game/engine/GameSystems.ts` — **MODIFIED**: read `selectedJetTypeId` + cached stats from the store at spawn; construct `Jet` with the selected type's numbers.
- `frontend/src/ui/screens/MenuScreen.tsx` — **MODIFIED/EXTENDED**: add a jet-selection step (3 cards with stats) before Start Game; store the selection on confirm.
- `frontend/src/ui/api/client.ts` — **MODIFIED**: add `getJetTypes()` (+ `JetTypeDto`); extend `saveGameRecord` to accept `jetTypeId`.
- `frontend/src/ui/screens/GameOverScreen.tsx` — **MODIFIED** if `jetTypeId` is sent on persist.
- `frontend/src/game/entities/Enemy.ts` — **POSSIBLY MODIFIED** if enemies gain health (so `jet.damage` matters across multiple hits).

### Specs
- `openspec/changes/jet-types-and-movement/specs/backend-jet-types/spec.md` — **NEW delta domain** (jet types catalog).
- `openspec/changes/jet-types-and-movement/specs/backend-game-records/spec.md` — **MODIFIED** delta (add jetTypeId to persist + the existing scenarios preserved).
- `openspec/changes/jet-types-and-movement/specs/frontend-game-client/spec.md` — **MODIFIED** delta (Jet Movement → acceleration model; Basic Enemy AI → defense/damage; new "Jet Type Selection" requirement; Game Phases Start Game → precondition of selection).

## Approaches

### 1. Jet types backend modeling

| Approach | Pros | Cons | Effort |
|----------|------|------|--------|
| **A. New `jet-types` bounded context + seed migration (3 rows)** | Matches hex/DDD discipline already in the repo (identity, game-records are siblings); clean `GET /jet-types` read endpoint; jet types are a first-class catalog (extensible to more types/admin tooling later without touching game-records); domain purity preserved (JetType aggregate + VOs for Speed/Defense/Damage/AccelerationRate); reuses existing port/use-case/controller/module/mapper shape verbatim | More files (a full context for 3 static rows feels heavy to some reviewers); one more migration + module registration | Medium |
| **B. Extend `game-records` context with a JetType entity + endpoint** | Fewer contexts; jet types only matter alongside records; one less module | Violates single-responsibility of the `game-records` aggregate (records are about completed sessions, not a jet catalog); couples a static catalog to a transactional write model; harder to grow into admin-managed catalog later; mixes read-catalog with per-user writes | Low-Medium |
| **C. Hardcode the 3 jet types as frontend constants only (no backend)** | Simplest; zero backend work; no migration | Violates the explicit user requirement ("These MUST be registered in the database"); records can't reference a persisted jet type; can't evolve types without a frontend redeploy; no single source of truth | Low (but NON-COMPLIANT with the request) |

**Recommendation: A — new `jet-types` bounded context with a seed migration.**
The user explicitly requires DB registration ("MUST be registered in the database"), approach C is non-compliant. Between A and B, the existing repo already commits to one-context-per-domain (identity ≠ game-records); bolting a static catalog onto `game-records` breaks that boundary and couples a read-only catalog to a per-user write model. The extra files in A are boilerplate that mirrors an existing, proven pattern — low marginal effort, high architectural consistency.

**Concrete jet-type entity (proposed for the proposal phase to confirm):**
- `id: uuid`, `name: text` (e.g. "Interceptor", "Heavy", "Balanced"), `max_speed: int`, `cruise_speed: int`, `acceleration_rate: numeric` (exponential factor `k`), `defense: int` (0–100 percentage), `damage: int`, `created_at`.
- 3 seed rows in the migration's `up()` (idempotent insert with fixed UUIDs so the frontend can't rely on DB-assigned order).
- `GET /jet-types` → `@Public()` (read-only catalog, no user-scoped data) OR authenticated — **proposal decision** (recommend `@Public` for MVP simplicity; it's not user-scoped).

---

### 2. Exponential acceleration model (frontend)

Current: `MovementSystem` sets `jet.vx = dx * jet.speed` (constant). The new model separates **direction** (normalized, from input) from **magnitude** (from the acceleration state machine).

| Approach | Formula | Pros | Cons | Effort |
|----------|---------|------|------|--------|
| **A. Exponential approach toward max** (discrete Euler) | `currentSpeed += (maxSpeed - currentSpeed) * k * dt`; on release `currentSpeed += (cruiseSpeed - currentSpeed) * k * dt` (same `k` or separate decay `k2`) | Frame-rate independent with `dt`; stable (never overshoots max); symmetric ramp/decay option; trivially tunable via one `k`; matches "exponential" wording naturally (asymptotic curve) | Need a `currentSpeed` field on the Jet + a per-tick integrator; two `k` values to tune (accel + decay) | Low-Medium |
| **B. Closed-form exponential** `v(t) = maxSpeed * (1 - e^(-k*t))` | Track `t` accelerating, compute `v(t)` directly | Mathematically exact exponential curve; pure function of elapsed accel time | Must track accumulate/decay time separately; harder to reverse the decay branch cleanly (different asymptote cruiseSpeed); more state than A | Medium |
| **C. Quadratic/polynomial ramp** `v = min(maxSpeed, cruiseSpeed + a*t²)` | Easy to reason; linear acceleration of acceleration | Not truly exponential (it's polynomial); unbounded growth needs a clamp; asymmetric vs the "exponential" requirement wording | Low |
| **D. Keep cruise as the baseline always, boost ramps on top** | Cruise = idle magnitude when no input OR no boost; boost multiplies toward max | Clean separation: direction always from input, magnitude from (cruise + boost ramp); natural "accelerator pedal" feel | Slightly more state (cruise is the floor, boost is the delta) | Low-Medium |

**Recommendation: A (exponential approach / Euler integrator) with cruise as the floor — a blend of A and D.**

State machine on the Jet (per frame, ms-based):
```
// direction (unchanged, from WASD/arrows), normalized:
if (dx,dy) present: dir = normalize(dx,dy); facing = atan2(dy,dx)
// magnitude (NEW):
accelerating = input.isShift() && direction present
target  = accelerating ? maxSpeed : cruiseSpeed
currentSpeed += (target - currentSpeed) * k * (dt/1000)
// velocity:
jet.vx = dir.x * currentSpeed
jet.vy = dir.y * currentSpeed
// NO input → keep facing, but magnitude STILL decays toward cruise (jet drifts at cruise, doesn't freeze)
```
Key decisions for the proposal phase:
- **No-input behavior change**: today no-input zeroes velocity (jet freezes). With a cruise baseline, the spec scenario "No input → jet maintains its current position" conflicts. **Proposal must decide**: keep freeze-on-no-input (cruise only applies while a direction is held) vs. drift-at-cruise-when-released. Recommend: **freeze when no direction held; cruise is the held-direction baseline; boost ramps above cruise while Shift is held.** This preserves the existing "No input" scenario and adds the new acceleration behavior only while moving.
- Tunable `k` (e.g. `0.005`/ms for accel, `0.008`/ms for decay) stored per jet type → different jets "feel" different.
- `currentSpeed` lives on the Jet entity (per-frame state, per Decision #9 — NOT in the Zustand store).

---

### 3. Defense & damage application

| Approach | Pros | Cons | Effort |
|----------|------|------|--------|
| **Defense = percentage 0–100; Damage = per-hit projectile damage; enemy health stays 1 (damage = score multiplier on kill)** | Minimal system changes; defense cleanly reduces contact damage: `actual = ENEMY_CONTACT_DAMAGE * (1 - defense/100)`; damage multiplies `SCORE_PER_KILL` so a high-damage jet earns more per kill; enemies stay 1-hit (no Enemy rewrite) | "damage" as a score multiplier is a naming stretch (players expect damage = hurt); less tactical depth (every kill is still 1 hit) | Low |
| **B. Defense = percentage 0–100; enemies gain health (>1); projectile `damage` reduces enemy health over multiple hits** | `damage` reads naturally; enables tanky enemies later; shooting a tougher enemy with a low-damage jet takes more hits — real tradeoff vs the fast/fragile jet | Requires Enemy rewrite (constructor takes health, `takeDamage(jet.damage)`), SpawnSystem tuning, possibly enemy-type variety; bigger blast radius | Medium-High |
| **C. Defense = flat reduction (int); damage = flat added to projectile** | Trivial arithmetic; no percentage edge cases | Hard to balance across wildly different magnitudes; flat reduction can produce 0 or negative damage (needs flooring); less intuitive | Low |
| **D. Defense affects projectile immunity chance / damage thresholds** | Rich mechanics | Over-engineered for an MVP; complex to spec/test | High |

**Recommendation: A for MVP (defense = % damage reduction, damage = score multiplier on kill) — revisit B in a later change.**
Reasoning: the user request lists the 3 properties (speed/defense/damage) but does NOT mention enemy health or multi-hit enemies. Approach B is the "correct" long-term design but pulls enemy spawning, enemy types, and CollisionSystem rewrite into scope — blowing the 400-line review budget. Approach A delivers all 3 stats with measurable, distinct effects per jet type:
- **Defense**: `actualDamage = ENEMY_CONTACT_DAMAGE * (1 - defense/100)` → a Heavy (defense 60) takes 8 dmg per hit vs an Interceptor (defense 10) taking 18. Distinct survival feel.
- **Damage**: `scoreFor(kill) = SCORE_PER_KILL * (damage / BASE_DAMAGE)` → a Heavy (damage 1.5) earns 150/kill vs Interceptor (damage 0.7) earning 70/kill. Distinct scoring incentive.
- **Speed/accel**: distinct maneuvering feel via the model above.
**Proposal must surface this as an explicit scope decision** (A keeps MVP small; B reserved for a follow-up change).

---

### 4. Concrete jet type triplets (proposal input)

The 3 types should form a clear rock-paper-scissors-ish tradeoff. Suggested starting stats (units: px/sec for speeds, k in 1/ms-ish, defense %, damage = multiplier of SCORE_PER_KILL):

| Jet | maxSpeed | cruiseSpeed | accelRate (k) | defense | damage | Feel |
|-----|----------|-------------|--------------|---------|--------|------|
| Interceptor | 460 | 200 | 0.004 | 10 | 0.7x | Fast, fragile, low score/kill |
| Balanced | 360 | 200 | 0.005 | 35 | 1.0x | Middle of the road |
| Heavy | 280 | 180 | 0.006 | 60 | 1.5x | Slow to top speed, tough, high score/kill |

(Exact numbers go in the proposal/design after we confirm approach 3-A and the formula in 2-A. Interceptor ramps fast in `k` but to a high max; Heavy ramps slightly faster in `k` but to a low max so it feels deliberate.)

---

### 5. Selection screen + persisted record

| Approach | Pros | Cons | Effort |
|----------|------|------|--------|
| **A. Selection step inside MenuScreen (3 cards → Start)** | Minimal router changes; one screen; stats fetched from `getJetTypes()` on mount; `selectedJetTypeId` + cached stats live in the Zustand store; Start becomes a 2-step (pick → confirm) flow | MenuScreen grows (still well under any review budget) | Low |
| **B. Separate `JetSelectScreen` (menu → select → playing)** | Cleaner separation if selection grows (previews, skins) | New route in App.tsx + new transition; more moving parts for a 3-card pick | Medium |
| **C. Inline on GameOverScreen "Play Again"** | N/A | Re-asks every run; conflicts with LOCKED "fixed for the whole session" | — |

**Recommendation: A** (inline selection in MenuScreen). The "fixed for the session" product rule means selection happens once per menu visit; a dedicated screen is overkill for 3 cards.

**Persisted record**: add `jet_type_id uuid NOT NULL REFERENCES jet_types(id)` to `game_records` via a 3rd migration. `saveGameRecord(score, durationMs, jetTypeId)` carries the chosen type. Enables future "high score per jet type" queries with zero extra columns.

## Recommendation (consolidated)

1. **Backend**: new `jet-types` bounded context mirroring `game-records` shape (aggregate + VOs + port + use case `ListJetTypes` + controller + module + mapper). One seed migration inserts the 3 jet types with fixed UUIDs. A *second* migration adds `game_records.jet_type_id` (FK). `GET /jet-types` returns the catalog (public for MVP; proposal decides auth).
2. **Frontend movement**: rewrite `MovementSystem` with an exponential-approach integrator (Approach 2-A). Jet entity gains `maxSpeed, cruiseSpeed, accelerationRate (k), defense, damage, currentSpeed`. Direction stays normalized from WASD/arrows; magnitude comes from the integrator. **No-input still freezes the jet** (cruise only applies while a direction is held). Shift = accelerator key.
3. **Frontend combat**: `CollisionSystem` applies `defense` as a % reduction on `ENEMY_CONTACT_DAMAGE`; `damage` multiplies `SCORE_PER_KILL` on each kill (Approach 3-A). No enemy-health rewrite this change (reserved for a follow-up).
4. **Frontend UI**: extend `MenuScreen` with a 3-card jet selection (Approach 5-A), fetch via new `getJetTypes()`. `gameStore` gains `selectedJetTypeId` (+ cached catalog). `GameSystems` reads the selection at spawn and builds the `Jet` with the chosen stats. `saveGameRecord` carries `jetTypeId`; `GameOverScreen` passes it through.
5. **Specs**: NEW `backend-jet-types` delta domain; MODIFIED `backend-game-records` (add jetTypeId); MODIFIED `frontend-game-client` (Jet Movement, Basic Enemy AI, new Jet Type Selection, Game Phases precondition).

## Risks

- **Spec conflict on "No input → jet maintains position"**: the existing frontend spec scenario explicitly says no input freezes the jet. Implementing true cruise-while-released would violate it. Mitigation: keep freeze-on-no-input; cruise is the held-direction baseline only. **Flag for spec phase** — the MODIFIED "Jet Movement" requirement must restate this precisely.
- **Scope creep into enemy health**: the natural reading of "damage" leads to approach 3-B (enemy health + multi-hit), which rewrites Enemy + SpawnSystem + CollisionSystem + adds enemy variety — likely exceeds the 400-line review budget. Risk: a reviewer pushes for B mid-implementation. Mitigation: the proposal explicitly scopes 3-A and lists B as a follow-up change.
- **Acceleration feel is subjective**: exponential `k` values need playtesting; wrong `k` → jet feels mushy or snap-to-max. Mitigation: keep `k` per jet type in the DB so it's tunable without redeploy; start conservative and iterate.
- **`GET /jet-types` auth decision**: marking it `@Public` exposes the catalog without auth; if we later add unreleased jets, that leaks. Mitigation: MVP-safe to be public (only 3 known types); proposal can lock `@Public` and revisit. Alternately keep it authenticated (consistent with every other route) for zero leakage risk.
- **Migration ordering / FK from game_records → jet_types**: the seed migration must run BEFORE the `game_records.jet_type_id` FK migration, and existing rows would violate a NOT NULL FK. The MVP has no production data, but the migration must still be written to add the column as nullable (or default to the Balanced UUID) to avoid breaking local dev DBs with old records. **Flag for design phase.**
- **Frontend hard fall-back if `getJetTypes()` fails**: if the catalog fetch fails on MenuScreen, the player can't proceed. Mitigation: keep a constants-based fallback of the same 3 jets in `constants.ts` so the game is playable offline / during backend downtime (and to keep the unit-testable surface stable).
- **Pixi Ticker `dt` units**: `Ticker.deltaMS` is ms; the integrator `currentSpeed += (target - currentSpeed) * k * (dt/1000)` requires `k` to be expressed in 1/sec, not 1/ms — easy to get wrong by a factor of 1000. Mitigation: document the unit in the Jet entity and the constants; add a unit test for the integrator.
- **Jet stat drift between frontend fallback and DB seed**: if the backend seed and the frontend fallback constants diverge, gameplay differs between online and offline. Mitigation: single source of truth = backend; frontend fallback is a copy with a `// keep in sync with backend seed migration` comment + a test asserting equality.
- **400-line review budget**: this change touches a new backend context (≈10 files) + a migration + a 2nd migration + record extension + frontend movement rewrite + combat tuning + selection UI + store + API client + 3 spec deltas. Likely a chained/sliced PR is needed (e.g. PR1 backend jet-types catalog; PR2 record extension + persist jetTypeId; PR3 frontend movement model; PR4 selection UI + combat). **Forecast for the tasks phase.**

## Ready for Proposal

**Yes.** The exploration has produced definitive recommendations on all 5 design forks (backend modeling, acceleration model, defense/damage model, selection placement, record persistence). The orchestrator should tell the user:

1. **Confirm the scope decision on `damage`**: MVP = score multiplier on kill (Approach 3-A); multi-hit enemies (3-B) deferred to a follow-up change. This is the single most consequential scope call — get explicit sign-off before spec.
2. **Confirm the "freeze-on-no-input vs cruise-on-release" behavior** for the movement model (recommendation keeps freeze; cruise applies only while a direction is held).
3. **Confirm the 3 concrete jet triplets** (Interceptor / Balanced / Heavy) and the starting stat numbers — exact tuning lands in design, but the archetypes should be agreed now.
4. **Confirm `GET /jet-types` is public** (recommendation) or authenticated.

With those four confirmations, the proposal phase can lock scope and the spec phase can write the three delta specs (new `backend-jet-types`, modified `backend-game-records`, modified `frontend-game-client`) with Given/When/Then scenarios and RFC 2119 keywords per the project rules.