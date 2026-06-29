# backend-game-records Specification

## Purpose

Persists completed Air-Pilote game session records and exposes per-player history and high score. All operations are authenticated. Records belong to a single player (the authenticated caller).

## Requirements

### Requirement: Persist Game Record

The system MUST persist exactly one game record per completed session for the authenticated player containing `userId`, `score`, `durationMs`, `timestamp`, and `jetTypeId`, MUST reject records for unauthenticated callers, MUST reject non-negative-score violations (score MUST be non-negative), and MUST reject records whose `jetTypeId` does not reference an existing jet type.

#### Scenario: Successful persistence

- GIVEN an authenticated player completes a session with score 1200, durationMs 45000, and a valid `jetTypeId`
- WHEN a persist request is submitted
- THEN a record is stored with the userId, score, durationMs, jetTypeId, and a server-assigned timestamp

#### Scenario: Invalid jetTypeId rejected

- GIVEN an authenticated player submits a record with a `jetTypeId` that does not reference an existing jet type
- WHEN the persist request is processed
- THEN the request is rejected with a validation error and no record is stored

#### Scenario: Unauthenticated request rejected

- GIVEN no valid access token is presented
- WHEN a persist request is submitted
- THEN the request is rejected as unauthenticated and no record is stored

#### Scenario: Negative score rejected

- GIVEN an authenticated player submits a record with score -5
- WHEN the persist request is processed
- THEN the request is rejected with a validation error and no record is stored

#### Scenario: Non-existent user rejected

- GIVEN the access token resolves to no known player
- WHEN a persist request is submitted
- THEN the request is rejected and no record is stored

### Requirement: High Score per Player

The system MUST return the highest score recorded for a given authenticated player and MUST require authentication.

#### Scenario: Retrieve high score

- GIVEN a player has records with scores 1200, 950, and 3000
- WHEN the player requests their high score
- THEN the system returns 3000

#### Scenario: No records exist

- GIVEN a player has no stored records
- WHEN the player requests their high score
- THEN the system returns a zero/null high score indicatING no records exist

### Requirement: List Game Records by User

The system MUST list a player's own game records ordered by most recent first, MUST require authentication, and MAY support pagination. Each returned record MUST include `jetTypeId`.

#### Scenario: Paginated history

- GIVEN a player has 25 stored records
- WHEN the player requests the first page of size 10
- THEN the 10 most recent records are returned, each including `jetTypeId`, with a cursor or total to fetch the next page

#### Scenario: Cannot access other players' records

- GIVEN an authenticated player requests records belonging to another userId
- WHEN the list request is processed
- THEN the system rejects the request or returns only that caller's own matching records

#### Scenario: Pagination past the end

- GIVEN a player has 5 records and requests page 2 of size 10
- WHEN the request is processed
- THEN an empty result set is returned without error

### Requirement: Authentication Enforcement

The system MUST authenticate every persist and retrieve operation via backend-identity access tokens and MUST NOT allow anonymous access.

#### Scenario: Token from another context rejected

- GIVEN a token not issued by backend-identity is presented
- WHEN a persist or retrieve request is submitted
- THEN the request is rejected as unauthenticated