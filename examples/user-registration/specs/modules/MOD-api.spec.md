---
id: MOD-api
name: HTTP API
kind: module
module: MOD-api
status: reviewed
depends_on: [MOD-db]
requirements: [REQ-001]
source: []
---

> **⚠️ FROZEN EXAMPLE** — part of `examples/user-registration/`. Delete before
> real use. Conventions: `.sdd/conventions.md`. Values:
> **Markdown is the source of truth (authority); reuse over repetition (DRY).**

# Purpose

`MOD-api` is the **HTTP API** module. It owns the registration use-case
controller, which validates the registration command and coordinates persistence
and password hashing. It is the home module of the `FEAT-001` feature.

# Public interface

- **Owns:** `CLS-regCtrl` (`RegistrationController`).
- **Inputs:** a `RegisterCmd` (`{ email, password, name }`) from the web layer.
- **Outputs:** `Result<UserView, RegError>` — `Ok(UserView)` on success.
- **Errors:** `EmailAlreadyTaken`, `WeakPassword` (declared by `CLS-regCtrl`).
- **Depends on:** `MOD-db` (for `CLS-userRepo` + `ENT-user`) and the shared
  `SHR-passwordHasher`.

# Invariants & rules

- The controller uses `error_style: result` (`.sdd/scot.md` §6): no exceptions
  cross the module boundary for rule violations — they are returned as `Err`.
- Business rules (email uniqueness, password length) are enforced here by
  delegating to `MOD-db` and `SHR-passwordHasher`; the module never re-checks
  what an entity invariant already guarantees.
- On success the controller emits a `UserRegistered` domain event.

# Contents

| id | name | kind | spec |
|----|------|------|------|
| CLS-regCtrl | RegistrationController | controller | specs/classes/CLS-regCtrl.spec.md |

# Acceptance criteria

- **AC1** — Given a `RegisterCmd` with a fresh email and an acceptable password,
  When it reaches `CLS-regCtrl.register`, Then the response is `Ok(UserView)` and
  a user is persisted via `MOD-db`.
- **AC2** — Given a `RegisterCmd` whose email already exists, When it reaches
  `CLS-regCtrl.register`, Then the response is `Err(EmailAlreadyTaken)` and
  nothing is persisted.
