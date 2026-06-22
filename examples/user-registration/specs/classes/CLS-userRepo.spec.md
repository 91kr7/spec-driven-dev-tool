<!--
  FROZEN, ILLUSTRATIVE EXAMPLE — NOT part of the tool's real specs/.
  Part of the "User registration (Database + API)" worked example.
  Delete examples/user-registration/ before real use.
  Authority: .sdd/conventions.md (ids §2, front-matter §3, kind→form §3).
  kind: interface → DECLARATIVE, signatures ONLY, NO SCoT body (conventions §3).
  A stub/mock is auto-derived from this interface; it is never specced separately.
  Markdown is the source of truth (authority); reuse over repetition (DRY).
-->
---
id: CLS-userRepo
name: UserRepository
kind: interface
module: MOD-db
status: reviewed
depends_on: [ENT-user]
requirements: [REQ-001]
source: [src/db/UserRepository.ts]
owns_sections: []
---

# Purpose

`UserRepository` is the persistence boundary for `ENT-user`. It hides the storage
mechanism behind a small, intention-revealing contract: check whether an email is
already taken, persist a user, and look a user up by email. Callers (the
registration controller) depend on **this interface**, never on a concrete store,
so a stub can be auto-derived from it for tests and for not-yet-ready
dependencies.

# Public interface

<!-- Signatures only — kind: interface carries NO behavioral body (no SCoT). -->

| Method                          | Inputs                                  | Output            | Errors                              |
|---------------------------------|-----------------------------------------|-------------------|-------------------------------------|
| `existsByEmail(email: String)`  | `email` — address to test (case-insensitive) | `Bool`       | — (a pure existence query; never throws on a missing row) |
| `save(user: User)`              | `user` — a valid `ENT-user` to persist  | `User`            | — (the controller pre-checks uniqueness; a store-level conflict surfaces as a thrown error mapped in impl-notes) |
| `findByEmail(email: String)`    | `email` — address to look up (case-insensitive) | `Option<User>` | — (absence is `None`, not an error) |

- `existsByEmail` and `findByEmail` compare email **case-insensitively**, matching
  the `ENT-user` uniqueness invariant.
- `save` returns the persisted `User` (e.g. with its generated identity) so the
  caller can build a view from the saved record.

# Invariants & rules

- The repository never stores plaintext passwords; it persists whatever
  `ENT-user` carries, and `ENT-user` only ever holds a `passwordHash`.
- `existsByEmail(e)` is `true` **iff** `findByEmail(e)` would return a user — the
  two are consistent for the same email.
- Email comparison is case-insensitive throughout, so `Ada@Example.com` and
  `ada@example.com` are the same user (mirrors `ENT-user`'s uniqueness invariant).
- This is a contract only; the concrete SQL / driver / mapping is a
  concretization recorded in the implementer's notes, not here (scot.md §9).

# Acceptance criteria

<!-- An interface is declarative; these ACs are contract expectations the derived
     stub and any real implementation must satisfy. -->

- **AC1** — Given a store containing a user with email `ada@example.com`, When
  `existsByEmail("ADA@example.com")` is called, Then it returns `true`
  (case-insensitive match).
- **AC2** — Given an empty store and a valid `ENT-user`, When `save(user)` is
  called, Then the returned `User` is persisted and a subsequent
  `findByEmail(user.email)` returns `Some(user)`.
- **AC3** — Given a store with no matching user, When `findByEmail("none@example.com")`
  is called, Then it returns `None` (absence is not an error).
