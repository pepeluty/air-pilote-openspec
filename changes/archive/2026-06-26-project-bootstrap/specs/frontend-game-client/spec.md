# frontend-game-client Specification

## Purpose

Single-player web game client for Air-Pilote: a 2D top-down (cenital) jet game rendered with PixiJS, with HUD/menus/auth UI in React, bridged via Zustand. Defines player input, jet movement, shooting, enemy AI, scrollable camera, HUD, game phases, auth UI, and the state bridge contract.

## Requirements

### Requirement: Jet Movement

The system MUST move the player jet in all 2D directions (up, down, left, right, and diagonals) in response to active input, at a fixed speed calibrated for responsive control.

#### Scenario: Diagonal movement

- GIVEN the player jet is spawned in the play field
- WHEN the player holds movement inputs for two perpendicular directions simultaneously
- THEN the jet moves diagonally at a normalized combined magnitude
- AND the jet faces or orient toward the movement direction

#### Scenario: No input

- GIVEN the jet is in the playing phase
- WHEN no movement input is active
- THEN the jet maintains its current position within the world

### Requirement: Shooting

The system MUST fire one player projectile from the jet on shoot input and MUST limit fire cadence by a cooldown.

#### Scenario: Fire projectile

- GIVEN the jet is alive in the playing phase
- WHEN the player triggers shoot input
- THEN a single projectile spawns from the jet and travels along its facing direction
- AND subsequent shots within the cooldown are ignored

#### Scenario: Fire blocked outside playing phase

- GIVEN the phase is paused, menu, or gameOver
- WHEN shoot input is triggered
- THEN no projectile spawns

### Requirement: Basic Enemy AI

The system MUST spawn enemies that move within the world, MAY damage the player on contact, and MUST be destroyable by player projectiles.

#### Scenario: Enemy destroyed by projectile

- GIVEN an enemy exists and a player projectile overlaps it
- WHEN the overlap is detected
- THEN the enemy is removed and the player score increments

#### Scenario: Enemy damages player

- GIVEN an enemy exists and overlaps the player jet
- WHEN the overlap is detected
- THEN the player health decreases by a defined amount and the enemy is removed

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

The system MUST expose phases menu, playing, paused, and gameOver, and MUST transition between them per defined triggers.

#### Scenario: Start game

- GIVEN the phase is menu
- WHEN the player starts a new game
- THEN the phase transitions to playing

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