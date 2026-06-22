---
id: ENT-user
name: User
kind: entity
module: MOD-db
status: reviewed
depends_on: []
requirements: [REQ-001]
source: [src/db/User.ts]
---

> **⚠️ FROZEN EXAMPLE** — part of `examples/user-registration/`. Delete before
> real use. Conventions: `.sdd/conventions.md` (entity = declarative: field table,
> relations, invariants). Values:
> **Markdown is the source of truth (authority); reuse over repetition (DRY).**

# Purpose

`ENT-user` is the **User** domain entity: a registered person identified by a
stable `id`, who logs in with a unique `email`, has a display `name`, and whose
credential is stored only as a `passwordHash`.

**Derivation note:** this single spec is the source of truth for **two** derived
artifacts — the TypeScript entity (`src/db/User.ts`, `MOD-db`) **and** the
Postgres migration for the `users` table (`MOD-build`,
`migrations/0001_create_users.ts`). Neither is hand-authored independently.

# Public interface

A factory is referenced by behavioral specs as `ENT-user.new(...)`:

| Operation | Signature                                              | Notes                                  |
|-----------|--------------------------------------------------------|----------------------------------------|
| `new`     | `new(email: String, name: String, passwordHash: String) -> User` | sets `id` (fresh UUID), `createdAt` (now), normalizes `email` to lowercase |

- **Inputs:** `email`, `name`, `passwordHash`.
- **Outputs:** a `User` satisfying all invariants below.
- **Errors:** construction with a value that violates an invariant is rejected
  (the caller — e.g. `CLS-regCtrl` — guards rule violations before constructing).

# Fields

| Field          | Type     | Required | Key  | Rules                                          |
|----------------|----------|----------|------|------------------------------------------------|
| `id`           | UUID     | yes      | pk   | system-generated, immutable                    |
| `email`        | String   | yes      | uniq | `format = email`; **stored lowercase**         |
| `name`         | String   | yes      | —    | length `1..100`                                |
| `passwordHash` | String   | yes      | —    | opaque hash (produced by `SHR-passwordHasher`) |
| `createdAt`    | DateTime | yes      | —    | default `now`                                  |

# Relations

None — `ENT-user` is standalone for this example.

# Invariants & rules

1. **Email uniqueness (case-insensitive):** no two users may share an email when
   compared case-insensitively.
2. **Email format:** `email` matches the email format.
3. **Name length:** `1 <= length(name) <= 100`.
4. **Hash, never plaintext:** the entity stores `passwordHash` only; it never
   holds or accepts a plaintext password.

# Acceptance criteria

- **AC1** — Given a stored user with email `a@x.com`, When a second user is
  created with email `A@X.com`, Then it is **rejected** (duplicate email,
  case-insensitive).
- **AC2** — Given an `email` of `"not-an-email"`, When a `User` is created, Then
  it is **rejected** (invalid email format).
- **AC3** — Given a `name` of 101 characters, When a `User` is created, Then it
  is **rejected** (name length exceeds 100).
