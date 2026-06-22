<!--
  FROZEN, ILLUSTRATIVE EXAMPLE — NOT part of the tool's real specs/.
  Part of the "User registration (Database + API)" worked example.
  Delete examples/user-registration/ before real use.
  Authority: .sdd/conventions.md (ids §2, front-matter §3, kind→form §3),
             .sdd/scot.md (the ONLY behavioral grammar — branch ids [Bn], §6 error styles).
  error_style: result MUST equal the SCoT `error-style:` below.
  Markdown is the source of truth (authority); reuse over repetition (DRY).
  Concretization (express router, argon2, repo insert) lives in
  .sdd/impl-notes/CLS-regCtrl.md — NOT here, and the test-writer never reads it.
-->
---
id: CLS-regCtrl
name: RegistrationController
kind: controller
module: MOD-api
status: reviewed
depends_on: [CLS-userRepo, SHR-passwordHasher, ENT-user]
requirements: [REQ-001]
source: [src/api/RegistrationController.ts]
owns_sections: []
error_style: result
---

# Purpose

`RegistrationController` is the registration use-case entry point on the HTTP API
side. It enforces the two registration rules — email must be free, password must
be strong enough — then creates the user (with a hashed password), persists it,
emits a domain event, and returns a `UserView`. It is the single place that
orchestrates `CLS-userRepo`, `SHR-passwordHasher`, and `ENT-user` for sign-up.

# Public interface

| Method                          | Inputs                                              | Output                      | Errors                                                            |
|---------------------------------|-----------------------------------------------------|-----------------------------|------------------------------------------------------------------|
| `register(cmd: RegisterCmd)`    | `cmd` — `{ email: String, password: String, name: String }` | `Result<UserView, RegError>` | `EmailAlreadyTaken` — an account with that email exists; `WeakPassword` — password shorter than 8 chars |

# Invariants & rules

- A user is persisted **only** when the email is free **and** the password is
  acceptable; either rule violation returns an `Err` and saves nothing.
- The password is hashed via `SHR-passwordHasher` before the user is built — the
  controller never holds or stores plaintext beyond the hashing call.
- Email existence is checked with `CLS-userRepo.existsByEmail` (case-insensitive),
  consistent with `ENT-user`'s uniqueness invariant.
- "Strong enough" for this example means **length ≥ 8**; anything shorter is
  `WeakPassword`.
- On success a `UserRegistered` event is emitted carrying the new user's id and
  email.

# SCoT

```
error-style: result

FUNCTION register(cmd: RegisterCmd) -> Result<UserView, RegError>
  INPUT:  cmd — { email: String, password: String, name: String }
  OUTPUT: Ok(UserView) on success; Err(RegError) on a rule violation

  [B1] IF CALL CLS-userRepo.existsByEmail(cmd.email) THEN
    RETURN Err(EmailAlreadyTaken)            # arm B1.then
  END
  # arm B1.else — email free

  [B2] IF length(cmd.password) < 8 THEN
    RETURN Err(WeakPassword)                 # arm B2.then
  END
  # arm B2.else — password acceptable

  hash  <- CALL SHR-passwordHasher.hash(cmd.password)
  user  <- CALL ENT-user.new(cmd.email, cmd.name, hash)
  saved <- CALL CLS-userRepo.save(user)
  EMIT UserRegistered({ userId: saved.id, email: saved.email })
  RETURN Ok(toUserView(saved))
END
```

Coverage set for `register`: `B1.then`, `B1.else`, `B2.then`, `B2.else`. Together
with the acceptance criteria below these are the units the test-writer must cover
(scot.md §7).

# Acceptance criteria

- **AC1** — Given no user exists with `cmd.email` and `cmd.password` is at least 8
  characters, When `register(cmd)` is called, Then the result is `Ok(UserView)`
  and the user is saved via `CLS-userRepo.save`. *(covers B1.else, B2.else)*
- **AC2** — Given a user already exists with `cmd.email` (any casing), When
  `register(cmd)` is called, Then the result is `Err(EmailAlreadyTaken)` and
  nothing is saved. *(covers B1.then)*
- **AC3** — Given no user exists with `cmd.email` but `cmd.password` is shorter
  than 8 characters, When `register(cmd)` is called, Then the result is
  `Err(WeakPassword)` and nothing is saved. *(covers B2.then)*
- **AC4** — Given a successful registration, When `register(cmd)` completes, Then
  a `UserRegistered` event carrying `{ userId, email }` is emitted exactly once.
  *(covers the success path B1.else, B2.else)*
