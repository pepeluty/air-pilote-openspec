# Proposal: jet-types-and-movement

## Intent
Extends the MVP with 3 selectable jet archetypes, an exponential acceleration + cruise model, and enemy health (multi-hit) so the `damage` stat is projectile damage — not a score multiplier. Ships a backend jet-type catalog and persists the selected jet per game record.

## Scope

### In Scope
- NEW `backend-jet-types` context: 3 seed rows, public `GET /jet-types`
- `game_records.jet_type_id` FK; persist `jetTypeId` per record
- Jet selection screen (3 cards) before Start Game
- Exponential acceleration model + cruise-in-last-direction (replaces freeze-on-no-input)
- Enemy health (>1): multi-hit; projectile damage = `jet.damage`; defense reduces contact damage

### Non-Goals
Enemy variety/bosses · Jet switching mid-game · Jet unlocking/progression · Admin CRUD for jet types (seed-only).

## Capabilities

### New Capabilities
- `backend-jet-types`: jet-type catalog — `JetType` aggregate + VOs (Speed, CruiseSpeed, AccelerationRate, Defense, Damage); seed migration (Interceptor/Balanced/Heavy with fixed UUIDs); `@Public()` `GET /jet-types`; hex/DDD layout mirroring `game-records`.

### Modified Capabilities
- `backend-game-records`: `GameRecord` aggregate/entity/mapper/DTOs carry `jetTypeId` (FK to jet_types); persist + list return it; existing scenarios preserved.
- `frontend-game-client`: Jet Movement MODIFIED (exponential ramp toward maxSpeed; cruise-in-last-direction replaces "no input freezes"); Basic Enemy AI MODIFIED (enemy health >1, multi-hit, projectile damage = jet.damage); NEW Jet Type Selection requirement; Game Phases Start Game MODIFIED (precondition: jet selected); Shooting/Collision MODIFIED (defense %, jet.damage).

## Approach
- Backend: new `jet-types` context (aggregate, VOs, port, `ListJetTypes` use case, `@Public()` controller, module, mapper). Migrations: (1) `jet_types` table + 3 seed rows; (2) `game_records.jet_type_id` FK nullable→default Balanced UUID.
- Frontend movement: `Jet` gains `maxSpeed, cruiseSpeed, accelerationRate, currentSpeed, defense, damage`. `MovementSystem` rewritten with exponential Euler: `currentSpeed += (target - currentSpeed) * k * (dt/1000)`; release decays toward cruise; no-input holds last direction at cruise. Shift = accelerate.
- Frontend combat: `Enemy` takes health; `CollisionSystem` applies `jet.damage` on projectile hit; contact damage reduced by `defense` %.
- Frontend UI: `MenuScreen` adds 3-card selection; `gameStore` adds `selectedJetTypeId`; `getJetTypes()` + constants fallback; `saveGameRecord` carries `jetTypeId`.
- Scope: **shared** SDD context (frontend + backend).

## Jet Archetypes (confirmed)

| Jet | maxSpeed | cruiseSpeed | accelRate | defense | damage |
|-----|----------|-------------|-----------|---------|--------|
| Interceptor | 460 | 200 | 0.004 | 10 | TBD |
| Balanced | 360 | 200 | 0.005 | 35 | TBD |
| Heavy | 280 | 180 | 0.006 | 60 | TBD |

`damage` tuned in design with enemy health. **GET /jet-types = public (confirmed).**

## Affected Areas

| Area | Impact |
|------|--------|
| `backend/src/contexts/jet-types/` | New context |
| `backend/src/migrations/` | 2 new (seed + FK) |
| `backend/src/contexts/game-records/` | Modified — jetTypeId |
| `frontend/src/game/entities/Jet.ts` `Enemy.ts` | Modified — per-type stats, currentSpeed, enemy health |
| `frontend/src/game/systems/MovementSystem.ts` | Rewritten |
| `frontend/src/game/systems/CollisionSystem.ts` `ShootingSystem.ts` | Modified |
| `frontend/src/game/input/KeyboardInput.ts` | Modified — Shift |
| `frontend/src/game/state/gameStore.ts` | Modified — selectedJetTypeId |
| `frontend/src/ui/screens/MenuScreen.tsx` `GameOverScreen.tsx` | Modified — selection / persist |
| `frontend/src/ui/api/client.ts` | Modified — getJetTypes, jetTypeId |
| `openspec/specs/*/spec.md` | 3 delta specs |

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| No-input cruise conflicts existing "No input freezes" scenario | High | MODIFIED requirement explicitly replaces it |
| `damage` + enemy health need joint tuning | Med | design phase |
| Acceleration `k` feel subjective | Med | per-type in DB; iterate |
| Catalog fetch failure blocks menu | Med | constants fallback |
| Migration FK on existing rows | Low | nullable→default Balanced UUID |
| 400-line review budget exceeded | Med | forecast chained PRs |

## Rollback Plan
Backend: revert the two migrations, drop `JetTypesModule` from app.module, restore game-records DTO/aggregate. Static seeds — no user data lost. Frontend: revert `MovementSystem` to constant-speed, restore `Enemy` health=1 and flat damage, remove selection step. No persisted user data depends on the change.

## Dependencies
- Migration ordering: jet_types seed runs before `jet_type_id` FK.
- Frontend fallback constants MUST stay in sync with backend seed.

## Success Criteria
- [ ] `GET /jet-types` returns 3 seeded archetypes without auth
- [ ] Each persisted game record carries a valid `jetTypeId` FK
- [ ] Selecting a jet changes spawn stats; 3 distinct feels in movement + combat
- [ ] No input → jet cruises in last direction (does not freeze)
- [ ] Enemies take multiple hits; kill time scales inversely with `jet.damage`
- [ ] Contact damage scales down with `jet.defense`