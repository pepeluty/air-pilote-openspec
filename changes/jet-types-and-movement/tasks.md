# Tasks: jet-types-and-movement

## Review Workload Forecast

| Field | Value |
|-------|-------|
| Estimated changed lines | ~1500-2500 (15 Create + 14 Modify + 1 Rewrite + 5 VOs) |
| 400-line budget risk | High |
| Chained PRs recommended | Yes |
| Suggested split | PR 1 → PR 2 → PR 3 → PR 4 → PR 5 → PR 6 → PR 7 |
| Delivery strategy | ask-on-risk |
| Chain strategy | stacked-to-main |

Decision needed before apply: No
Chained PRs recommended: Yes
Chain strategy: stacked-to-main
400-line budget risk: High

### Suggested Work Units

| Unit | Goal | Likely PR | Notes |
|------|------|-----------|-------|
| 1 | jet-types context + GET /jet-types + seed migration | PR 1 [backend] | base=main; autonomous; tests/docs included |
| 2 | game-records +jetTypeId, FK migration, JetTypeExists validation (422) | PR 2 [backend] | base=PR 1 branch; depends on JetTypeExists port |
| 3 | API client getJetTypes + gameStore selectedJetTypeId/jetStats + constants fallback | PR 3 [frontend] | base=main; independent of backend |
| 4 | MovementSystem rewrite (exponential Euler + cruise-in-last-direction) + Jet entity stats + Shift input | PR 4 [frontend] | base=PR 3 branch; reads jetStats from store |
| 5 | Enemy health 100 + CollisionSystem multi-hit + ShootingSystem jet.damage + defense % | PR 5 [frontend] | base=PR 4 branch; needs Jet.damage/defense |
| 6 | MenuScreen 3-card selection UI + GameOverScreen jetTypeId persist + Start precondition wiring | PR 6 [frontend] | base=PR 5 branch; needs working game |
| 7 | BE unit + integration tests; FE unit + E2E tests | PR 7 [shared] | base=PR 6 branch; covers all spec scenarios |

## Phase 1: Backend jet-types Context [backend]

- [x] 1.1 Create `backend/src/contexts/jet-types/domain/vo/{Speed,CruiseSpeed,AccelerationRate,Defense,Damage}.ts` — VOs with invariants (defense [0,100], maxSpeed>cruiseSpeed>0, accel/damage >0)
- [x] 1.2 Create `backend/src/contexts/jet-types/domain/JetType.ts` — aggregate `create`/`rehydrate` enforcing maxSpeed>cruiseSpeed>0
- [x] 1.3 Create `backend/src/contexts/jet-types/application/ports/{JetTypeRepository,JetTypeExists}.port.ts` — `findAll`/`findById` + `exists(id)`
- [x] 1.4 Create `backend/src/contexts/jet-types/application/usecases/ListJetTypes.ts` — returns the 3 jet types
- [x] 1.5 Create `backend/src/contexts/jet-types/infrastructure/persistence/{JetTypeEntity,mappers,JetTypeRepositoryAdapter}.ts` — MikroORM entity + Data Mapper + adapter impls both ports
- [x] 1.6 Create `backend/src/contexts/jet-types/infrastructure/controllers/JetTypesController.ts` — `@Public() GET /jet-types`
- [x] 1.7 Create `backend/src/contexts/jet-types/infrastructure/nest-jet-types.module.ts` — exports `JetTypeExists`
- [x] 1.8 Create `backend/src/migrations/Migration20260626000002_jet_types.ts` — `jet_types` table + 3 seed rows with fixed UUIDs and values 460/200/4.0/10/30, 360/200/5.0/35/45, 280/180/6.0/60/80
- [x] 1.9 Modify `backend/src/app.module.ts` — register `JetTypesModule`

## Phase 2: Backend game-records Extension [backend]

- [x] 2.1 Modify `backend/src/contexts/game-records/domain/GameRecord.ts` — add `jetTypeId` field + update `create`/`rehydrate` signatures
- [x] 2.2 Modify `backend/src/contexts/game-records/infrastructure/persistence/{GameRecordEntity,mappers}.ts` — add `jet_type_id` column + mapper round-trip
- [x] 2.3 Modify `backend/src/contexts/game-records/application/usecases/PersistGameRecord.ts` — add `jetTypeId` cmd + `JetTypeExists` validation (reject unknown → `ValidationError` 422, no FK 500)
- [x] 2.4 Modify `backend/src/contexts/game-records/infrastructure/controllers/GameRecordsController.ts` — DTO + response include `jetTypeId`
- [x] 2.5 Modify `backend/src/contexts/game-records/infrastructure/nest-game-records.module.ts` — import `JetTypesModule` for `JetTypeExists`
- [x] 2.6 Create `backend/src/migrations/Migration20260626000003_game_records_jet_type_fk.ts` — add `jet_type_id` NOT NULL FK default Balanced UUID (MUST run after 1.8)

## Phase 3: Frontend API + Store + Constants [frontend]

- [x] 3.1 Modify `frontend/src/ui/api/client.ts` — add `getJetTypes()`, `JetTypeDto`; extend `saveGameRecord` with `jetTypeId`
- [x] 3.2 Modify `frontend/src/game/state/gameStore.ts` — add `selectedJetTypeId` + `jetStats` (cached stats)
- [x] 3.3 Modify `frontend/src/game/constants.ts` — set `ENEMY_MAX_HEALTH=100`; add `FALLBACK_JET_TYPES` matching seed UUIDs+values exactly

## Phase 4: Frontend MovementSystem + Jet Entity [frontend]

- [x] 4.1 Modify `frontend/src/game/entities/Jet.ts` — replace `speed` with `maxSpeed/cruiseSpeed/accelerationRate/defense/damage`; add `currentSpeed=0`, `lastDirection={0,0}`
- [x] 4.2 Rewrite `frontend/src/game/systems/MovementSystem.ts` — exponential Euler `currentSpeed += (target-currentSpeed)*k*(dt/1000)`; cruise-in-last-direction; no-input holds last dir at cruise (does NOT freeze); spawn (no lastDir) stays stationary
- [x] 4.3 Modify `frontend/src/game/input/KeyboardInput.ts` — add `isAccelerate()` (Shift); add Shift to `GAME_KEYS`
- [x] 4.4 Modify `frontend/src/game/engine/GameSystems.ts` — read `jetStats` at spawn; construct `Jet` with selected type stats

## Phase 5: Frontend Combat Multi-hit [frontend]

- [ ] 5.1 Modify `frontend/src/game/entities/Enemy.ts` — health default 100 from `ENEMY_MAX_HEALTH`; verify `takeDamage(amount)` decrements + flags destroyed at health<=0
- [ ] 5.2 Modify `frontend/src/game/entities/Projectile.ts` — add `damage` property set from jet at spawn
- [ ] 5.3 Modify `frontend/src/game/systems/CollisionSystem.ts` — projectile hit applies `projectile.damage` (multi-hit, destroy+score at health<=0); contact `actualDamage = ENEMY_CONTACT_DAMAGE*(1-jet.defense/100)`
- [ ] 5.4 Modify `frontend/src/game/systems/ShootingSystem.ts` — spawn `Projectile` carrying `jet.damage`

## Phase 6: Frontend Selection UI + Wiring [frontend]

- [ ] 6.1 Modify `frontend/src/ui/screens/MenuScreen.tsx` — 3-card selection; fetch `getJetTypes` on mount with `FALLBACK_JET_TYPES` on failure; click sets `selectedJetTypeId`+`jetStats`
- [ ] 6.2 Modify `frontend/src/ui/screens/GameOverScreen.tsx` — pass `selectedJetTypeId` to `saveGameRecord`
- [ ] 6.3 Wire Start Game precondition — disable Start until `selectedJetTypeId` set (spec "Start game" scenario: unset → unavailable)

## Phase 7: Testing & Verification [shared]

- [ ] 7.1 [backend] Unit: VO invariants (maxSpeed>cruiseSpeed>0, defense [0,100], damage/accel >0); `ListJetTypes` returns 3; `JetTypeExists` true/false — Jest + fakes
- [ ] 7.2 [backend] Unit: `PersistGameRecord` valid jetTypeId ok; invalid → `ValidationError`; negative score → `ValidationError` — Jest + `JetTypeExists` fake
- [ ] 7.3 [backend] Integration: seed migration → 3 rows fixed UUIDs; `GET /jet-types` 200 no auth; `game_records.jet_type_id` FK + default Balanced on existing rows — testcontainers Postgres
- [ ] 7.4 [frontend] Unit: exponential accel (ramp→max, decay→cruise, cruise-in-last-direction, diagonal normalize×currentSpeed); Enemy multi-hit (survives >damage, destroy+score at <=0); defense % contact; Jet selection (3 cards, click sets store, Start disabled, fetch→fallback) — Vitest + RTL
- [ ] 7.5 [frontend] E2E: select jet → start → kill enemy (multi-hit) → gameOver → score saved with `jetTypeId` — Playwright (no canvas pixel asserts)
