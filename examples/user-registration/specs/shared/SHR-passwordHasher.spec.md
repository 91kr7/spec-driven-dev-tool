---
id: SHR-passwordHasher
name: PasswordHasher
kind: interface
module: MOD-api
status: reviewed
depends_on: []
requirements: [REQ-001]
source: [src/shared/passwordHasher.ts]
---

> **⚠️ FROZEN EXAMPLE** — part of `examples/user-registration/`. Delete before
> real use. Conventions: `.sdd/conventions.md` (interface = declarative,
> **signatures only, no body**; a stub is auto-derived). Values:
> **Markdown is the source of truth (authority); reuse over repetition (DRY).**

# Purpose

`SHR-passwordHasher` is a **shared non-UI abstraction** that turns a plaintext
password into an opaque, verifiable hash and checks a candidate against it. It
hides the concrete hashing library behind a stable contract so callers
(`CLS-regCtrl`) depend on behavior, not a vendor.

# Public interface

Signatures only — **no body** (interface). A stub returning defaults is
auto-derived from this spec for not-yet-ready dependencies and for tests
(`.sdd/conventions.md` §3).

| Operation | Signature                                  | Notes                                          |
|-----------|--------------------------------------------|------------------------------------------------|
| `hash`    | `hash(plain: String) -> String`            | returns an opaque hash; never returns `plain`  |
| `verify`  | `verify(plain: String, hash: String) -> Bool` | `true` iff `plain` produced `hash`          |

- **Inputs:** a plaintext password (`hash`); a plaintext + a stored hash (`verify`).
- **Outputs:** an opaque hash string; a boolean match.
- **Errors:** none declared at the contract level (a malformed `hash` argument to
  `verify` yields `false`, not an error).

# Invariants & rules

1. **Round-trip:** for any `plain`, `verify(plain, hash(plain))` is `true`.
2. **Rejection:** for `wrong != plain`, `verify(wrong, hash(plain))` is `false`.
3. **Opacity:** `hash(plain)` never equals `plain` and is not reversible by this
   contract.

> The concrete algorithm (e.g. argon2) is a **concretization** chosen at
> implementation time and recorded in the caller's impl-notes — never in this
> spec (`.sdd/scot.md` §9).

# Acceptance criteria

- **AC1** — Given any password `plain`, When `verify(plain, hash(plain))` is
  evaluated, Then it returns `true`.
- **AC2** — Given a password `plain` and a different `wrong`, When
  `verify(wrong, hash(plain))` is evaluated, Then it returns `false`.
