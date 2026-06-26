# Air-Pilote — OpenSpec (SDD planning context)

Spec-Driven Development planning artifacts for **Air-Pilote**, covering both
independent git repos in the workspace.

## Contents

- `config.yaml` — OpenSpec + SDD configuration (persistence mode, strict TDD,
  testing capabilities, phase rules).
- `specs/` — consolidated capability specs (deltas merged here on archive).
- `changes/` — in-flight SDD changes.
  - `changes/<change-name>/` — `proposal.md`, `specs/`, `design.md`, `tasks.md`,
    and `exploration.md` for a change.
  - `changes/archive/` — archived (completed) changes.

## Scope

This repo is the **shared SDD context** for the Air-Pilote workspace. It is one
of three independent git repos:

- `openspec/` — this repo (planning context)
- `frontend/` — React + TS + PixiJS game client
- `backend/` — NestJS + PostgreSQL API

The workspace root is **not** a git repo by design. Planning artifacts here
describe work that lands in the `frontend/` and `backend/` repos; this repo
only tracks the specs, designs, proposals, and task lists that drive those
repos.

## Persistence mode

`hybrid` — artifacts are written to files in this repo **and** mirrored to
Engram persistent memory observations.
