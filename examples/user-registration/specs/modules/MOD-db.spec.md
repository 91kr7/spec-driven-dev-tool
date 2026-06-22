---
id: MOD-db
name: Persistence
kind: module
module: MOD-db
status: reviewed
depends_on: []
requirements: [REQ-001]
source: []
---

> **⚠️ FROZEN EXAMPLE** — part of `examples/user-registration/`. Delete before
> real use. Conventions: `.sdd/conventions.md`. Values:
> **Markdown is the source of truth (authority); reuse over repetition (DRY).**

# Purpose

`MOD-db` is the **Persistence** module. It owns the user data model and the data
access used to read and write it. It exposes a repository abstraction so higher
modules (`MOD-api`) never touch raw storage.

# Public interface

- **Owns:** `ENT-user` (the user table/data model) and `CLS-userRepo`
  (`UserRepository`, the data-access interface).
- **Inputs:** persistence requests via `CLS-userRepo` (`existsByEmail`, `save`,
  `findByEmail`).
- **Outputs:** persisted `User` records and existence/lookup answers.
- **Errors:** none surfaced at the module boundary beyond what `CLS-userRepo`
  declares; uniqueness is enforced by the entity invariants + the migration.

# Invariants & rules

- The schema/migration for `ENT-user` is **derived** from the entity spec and
  lives in `MOD-build` — it is never hand-authored independently
  (`.sdd/conventions.md` §2).
- `ENT-user.email` uniqueness (case-insensitive) is enforced at the data layer.
- `MOD-db` depends on nothing (`depends_on: []`); it is processed first in
  topological order.

# Contents

| id | name | kind | spec |
|----|------|------|------|
| ENT-user | User | entity | specs/model/ENT-user.spec.md |
| CLS-userRepo | UserRepository | interface | specs/classes/CLS-userRepo.spec.md |

# Acceptance criteria

- **AC1** — Given a `User` with a fresh email, When it is saved through
  `CLS-userRepo.save`, Then it is persisted and retrievable by `findByEmail`.
- **AC2** — Given an email already stored (case-insensitively), When
  `CLS-userRepo.existsByEmail` is called, Then it returns `true`.
