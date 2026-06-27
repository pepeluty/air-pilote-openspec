# backend-jet-types Specification

## Purpose

Catalog of selectable jet archetypes for Air-Pilote. Exposes a read-only, public list of seeded jet types — each defining the movement and combat stats applied to a player's jet for a session. Seed-only in this change; no admin CRUD is exposed.

## Requirements

### Requirement: Jet Type Catalog

The system MUST expose the available jet types via a public `GET /jet-types` endpoint that requires no authentication. The response MUST be an array where each entry contains `id`, `name`, `maxSpeed`, `cruiseSpeed`, `accelerationRate`, `defense`, and `damage`.

#### Scenario: GET /jet-types returns three types without auth

- GIVEN no access token is presented
- WHEN a client requests `GET /jet-types`
- THEN the response is HTTP 200 with an array of exactly three jet types
- AND no authentication is required

#### Scenario: Each type exposes all required properties

- GIVEN the `GET /jet-types` response
- WHEN each array entry is inspected
- THEN it contains `id`, `name`, `maxSpeed`, `cruiseSpeed`, `accelerationRate`, `defense`, and `damage`

### Requirement: Jet Type Seed Data

The system MUST seed exactly three jet types — Interceptor, Balanced, and Heavy — via a database migration using fixed UUIDs so identity is stable across environments. The seed MUST be immutable in this change; no create, update, or delete operations on jet types are exposed.

#### Scenario: Migration produces three seeded rows with expected stats

- GIVEN the database has run the jet-types seed migration
- WHEN the catalog is queried
- THEN three rows exist named Interceptor, Balanced, and Heavy
- AND each row carries its predefined maxSpeed, cruiseSpeed, accelerationRate, defense, and damage with stable UUIDs

### Requirement: Jet Type Properties Validation

Each jet type MUST satisfy invariant constraints: `maxSpeed > cruiseSpeed > 0`, `accelerationRate > 0`, `defense` in `[0, 100]`, and `damage > 0`. Any jet type violating these constraints MUST be rejected with a validation error.

#### Scenario: Invalid jet type properties are rejected

- GIVEN a jet type with `cruiseSpeed` greater than or equal to `maxSpeed`, or `defense` outside `[0, 100]`, or non-positive `damage` or `accelerationRate`
- WHEN the type is constructed or validated
- THEN a validation error is raised and the type is not persisted
