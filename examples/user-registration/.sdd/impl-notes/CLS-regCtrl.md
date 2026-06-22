<!--
  FROZEN, ILLUSTRATIVE EXAMPLE — NOT part of the tool's real .sdd/.
  Part of the "User registration (Database + API)" worked example.
  Delete examples/user-registration/ before real use.
-->

# Impl-notes — `CLS-regCtrl` (RegistrationController)

> **Implementer-owned concretization. NOT part of the gated spec.** This file
> records the decisions the SCoT deliberately omits — library choices, API
> bindings, idiom mappings, edge cases, and bug-fix lessons (conventions §8;
> scot.md §9). The **spec** (`specs/classes/CLS-regCtrl.spec.md`) stays the
> authority; if anything here ever disagrees with the spec, the spec wins.
>
> **The test-writer never reads this file.** Tests are derived independently from
> the spec's ACs and SCoT branch arms, so nothing written here can leak into the
> oracle. Only the `code-implementer` (writer) and the `code-gatekeeper` (reader)
> consult it.

Spec: `specs/classes/CLS-regCtrl.spec.md` · Source: `src/api/RegistrationController.ts`

---

## Chosen libraries / frameworks

- **HTTP layer:** an **Express** router mounts `POST /register`, parses the JSON
  body into a `RegisterCmd`, and delegates to `register(cmd)`. The router is thin —
  no business logic lives in it; all rules stay in the controller per the spec.
- **Password hashing:** **argon2** (argon2id) sits **behind** `SHR-passwordHasher`.
  The controller only ever calls `SHR-passwordHasher.hash` / `.verify` (by id, per
  the SCoT); argon2 is an implementation detail of that shared abstraction and is
  not referenced directly here.

## API binding (SCoT CALL → concrete call)

| SCoT step                                   | Concrete binding                                                        |
|---------------------------------------------|-------------------------------------------------------------------------|
| `CALL CLS-userRepo.existsByEmail(cmd.email)`| `await userRepo.existsByEmail(email)` — `SELECT 1 ... WHERE lower(email)=lower($1)` |
| `CALL SHR-passwordHasher.hash(cmd.password)`| `await passwordHasher.hash(password)` (argon2id) → the stored `passwordHash` |
| `CALL ENT-user.new(cmd.email, cmd.name, hash)` | constructs the `User` (lowercases email, fills `id`/`createdAt` defaults) |
| `CALL CLS-userRepo.save(user)`              | `await userRepo.save(user)` → an `INSERT ... RETURNING *` mapped back to `User` |
| `EMIT UserRegistered({...})`                | published on the in-process event bus after the row is committed        |

## Error-style mapping

- Spec `error_style: result` → in TypeScript a **discriminated union**
  `type Result<T,E> = { ok: true; value: T } | { ok: false; error: E }`.
- `RegError` is a discriminated union of `{ kind: "EmailAlreadyTaken" }` and
  `{ kind: "WeakPassword" }`. The Express adapter maps these to HTTP `409` and
  `400` respectively (mapping lives in the router, not in `register`).
- `register` itself never throws for a rule violation — it returns `Err(...)` so
  the branch arms `B1.then` / `B2.then` stay pure returns, matching the SCoT.

## Edge cases

- **Email is compared and stored case-insensitively.** The existence check uses
  `lower(email)`; `ENT-user.new` lowercases the email before construction, so
  `Ada@Example.com` and `ada@example.com` are treated as the same account
  (consistent with `ENT-user`'s uniqueness invariant and `CLS-userRepo`).
- **Trim before checking:** the email is trimmed of surrounding whitespace prior
  to the existence check so `" ada@example.com "` cannot bypass uniqueness.
- **Password length is measured in characters**, not bytes, for the `< 8` weak
  check, so multi-byte passwords are not mis-measured.

## Bug-fix log

- (empty — no fixes recorded yet. New entries append here as
  `YYYY-MM-DD — symptom → root cause → minimal diff`, per the change policy.)
