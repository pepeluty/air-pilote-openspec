# frontend-game-client Specification

## Purpose

Single-player web game client for Air-Pilote: a 2D top-down (cenital) jet game rendered with PixiJS, with HUD/menus/auth UI in React, bridged via Zustand. Defines player input, jet movement, shooting, enemy AI, scrollable camera, HUD, game phases, auth UI, and the state bridge contract.

## Requirements

### Requirement: Jet Movement

The system MUST move the jet in all 2D directions via directional input using an exponential model ramping `currentSpeed` toward `maxSpeed` while accelerate is held and decaying toward `cruiseSpeed` when released. With no directional input, the jet MUST cruise at `cruiseSpeed` in the last held direction (it does NOT freeze).

#### Scenario: Diagonal movement

- GIVEN the jet is spawned
- WHEN the player holds two perpendicular inputs
- THEN the jet moves diagonally with a normalized direction
- AND magnitude comes from `currentSpeed`, not a fixed constant
- AND the jet faces the movement direction

#### Scenario: No input

- GIVEN the jet is playing with a last held direction
- WHEN no movement input is active
- THEN the jet cruises at `cruiseSpeed` in the last held direction
- AND does NOT freeze

#### Scenario: Exponential acceleration

- GIVEN `currentSpeed` is below `maxSpeed` in a held direction
- WHEN the player holds accelerate plus a direction
- THEN `currentSpeed` ramps exponentially toward `maxSpeed` frame-rate independently

#### Scenario: Release accelerate

- GIVEN the jet moves faster than `cruiseSpeed`
- WHEN the player releases accelerate but keeps a direction
- THEN `currentSpeed` decays exponentially toward `cruiseSpeed`

#### Scenario: Cruise in last direction

- GIVEN the player releases all input after moving
- WHEN no directional input is active
- THEN the jet continues at `cruiseSpeed` in the last held direction

### Requirement: Shooting

The system MUST fire one projectile on shoot input, MUST limit fire cadence by a cooldown, and the projectile MUST deal `jet.damage` on hit.

#### Scenario: Fire projectile

- GIVEN the jet is alive in playing
- WHEN the player triggers shoot
- THEN a projectile spawns from the jet along its facing
- AND deals `jet.damage` on hit
- AND shots within the cooldown are ignored

#### Scenario: Fire blocked outside playing phase

- GIVEN the phase is paused, menu, or gameOver
- WHEN shoot input is triggered
- THEN no projectile spawns

### Requirement: Basic Enemy AI

The system MUST spawn enemies with health greater than one, MAY damage the player on contact, and MUST be destroyable by projectiles. Projectiles deal `jet.damage` per hit; an enemy is destroyed at health zero. Contact damage is reduced by `jet.defense`: `actualDamage = ENEMY_CONTACT_DAMAGE * (1 - defense / 100)`.

#### Scenario: Enemy destroyed by projectile

- GIVEN an enemy with health H overlaps a projectile
- WHEN the overlap is detected
- THEN enemy health decreases by `jet.damage`
- AND at health zero the enemy is removed and score increments

#### Scenario: Enemy damages player

- GIVEN an enemy overlaps the jet
- WHEN the overlap is detected
- THEN player health decreases by `ENEMY_CONTACT_DAMAGE * (1 - jet.defense / 100)` and the enemy is removed

#### Scenario: Enemy survives first hit

- GIVEN an enemy with health greater than `jet.damage`
- WHEN one projectile hits it
- THEN health decreases by `jet.damage` and the enemy is NOT destroyed

### Requirement: Jet Type Selection

The system MUST provide a pre-game selection screen with one card per jet type showing name and stats. The player MUST select a type before starting. The selection is stored in the game store as `selectedJetTypeId` with cached stats and applied to the spawned jet.

#### Scenario: Display three jet types

- GIVEN the player is on the selection screen
- WHEN it mounts
- THEN three cards show name and stats fetched from the backend

#### Scenario: Select a jet type

- GIVEN three cards are displayed
- WHEN the player selects a card
- THEN `selectedJetTypeId` is set and the stats are cached

#### Scenario: Cannot start without selecting

- GIVEN no jet type is selected
- WHEN the player attempts to start
- THEN the start action is disabled or rejected

#### Scenario: Fetch failure falls back to constants

- GIVEN `GET /jet-types` fails
- WHEN the screen loads
- THEN three jet types display from fallback constants so the game stays playable

### Requirement: Top-down Scrollable Camera

The system MUST scroll the camera to follow the player jet and MUST clamp camera bounds to the world map edges.

#### Scenario: Follow jet within world

- GIVEN the jet moves within world bounds
- WHEN the jet position changes
- THEN the camera translates to keep the jet in view

#### Scenario: Clamp at world edge

- GIVEN the jet approaches a world map edge
- WHEN the jet would move beyond world bounds
- THEN camera scrolling stops at the edge and the jet is clamped within world bounds

### Requirement: HUD Overlay

The system MUST display a non-moving HUD layer above the canvas showing current score and current health.

#### Scenario: Score updates on enemy destroyed

- GIVEN an enemy is destroyed in the playing phase
- WHEN the score value changes
- THEN the HUD score display updates accordingly

#### Scenario: Health updates on damage

- GIVEN the player takes damage
- WHEN health changes
- THEN the HUD health display updates accordingly

### Requirement: Game Phases

The system MUST expose phases menu, playing, paused, and gameOver, and MUST transition per defined triggers. Starting a game MUST require a selected jet type.

#### Scenario: Start game

- GIVEN the phase is menu and `selectedJetTypeId` is set
- WHEN the player starts a game
- THEN the phase transitions to playing
- AND if `selectedJetTypeId` is unset, Start Game is unavailable

#### Scenario: Player death

- GIVEN the player health reaches zero in the playing phase
- WHEN health becomes zero
- THEN the phase transitions to gameOver

### Requirement: Auth UI

The system MUST provide login and register screens that submit credentials to backend-identity and MUST transition to the menu on success.

#### Scenario: Register success

- GIVEN the player is on the register screen
- WHEN registration succeeds
- THEN the system stores authentication state and transitions to menu

#### Scenario: Login failure

- GIVEN the player is on the login screen
- WHEN credentials are invalid
- THEN the system displays an error without changing phase

### Requirement: Zustand State Bridge

The system MUST bridge slow game-facing state (score, health, phase) between Pixi and React via Zustand and MUST NOT push per-frame data into the store.

#### Scenario: Slow state publish

- GIVEN the score, health, or phase changes
- WHEN a change occurs
- THEN the Zustand store updates once per change event
- AND per-frame transform data stays inside Pixi

#### Scenario: Per-frame data isolation

- GIVEN the game loop ticks at frame rate
- WHEN the jet moves continuously
- THEN the Zustand store is NOT updated with per-frame positions