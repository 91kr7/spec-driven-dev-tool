<!--
  FROZEN, ILLUSTRATIVE EXAMPLE — NOT part of the tool's real specs/.
  Part of the "User registration (Database + API)" worked example.
  Delete examples/user-registration/ before real use.
  Authority: .sdd/conventions.md (ids §2, front-matter §3, status §5),
             .sdd/scot.md (orchestration grammar — cross-class CALLs only).
  Markdown is the source of truth (authority); reuse over repetition (DRY).
  Collaborators are named BY ID only — never re-described here.
-->
---
id: FEAT-001
name: User registration
kind: use-case
module: MOD-api
status: reviewed
depends_on: [CLS-registerScreen, CLS-regCtrl, CLS-userRepo, ENT-user, SHR-passwordHasher]
requirements: [REQ-001]
source: []
error_style: result
---

# Purpose

A first-time visitor creates an account. From the registration screen they enter
an email, a password, and a display name and submit. The system rejects a
duplicate email or a too-weak password with a precise message and persists
nothing; on a clean submission it stores exactly one new user (with a hashed
password) and the visitor lands on the welcome screen. This feature is **purely
compositional** — it owns no coordinator code of its own (`source: []`); it wires
existing collaborators together and is verified by integration tests.

# Public interface

- **Inputs:** `RegisterCmd` — `{ email: String, password: String, name: String }`
- **Outputs:** `Result<UserView, RegError>` — `Ok(UserView)` for the newly
  created user on success; the screen then navigates to `/welcome`.
- **Errors:**
  - `EmailAlreadyTaken` — a user with the same email (case-insensitive) exists
  - `WeakPassword` — the password is shorter than 8 characters

# Collaborators

| Id                   | Kind       | Role in this flow                                                       |
|----------------------|------------|-------------------------------------------------------------------------|
| `CLS-registerScreen` | gui        | Collects the form, calls the controller, navigates on success           |
| `CLS-regCtrl`        | controller | The registration entry point: enforces rules, orchestrates persistence  |
| `CLS-userRepo`       | interface  | Checks email existence and persists the new user                        |
| `ENT-user`           | entity     | The domain user created from the command and the password hash          |
| `SHR-passwordHasher` | interface  | Hashes the plaintext password before it is stored                       |

# Invariants & rules

- At most one user row results from a successful registration; a rejected
  registration (duplicate email or weak password) persists **nothing**.
- The stored password is always a hash, never the plaintext (`SHR-passwordHasher`).
- Email uniqueness is enforced **case-insensitively** (the rule lives with
  `ENT-user`; the controller queries `CLS-userRepo.existsByEmail` before saving).
- On success the caller receives a `UserView` and the screen navigates to
  `/welcome`; on any error the screen shows the message and stays put.

# Orchestration SCoT

```
error-style: result

FUNCTION register(cmd: RegisterCmd) -> Result<UserView, RegError>
  INPUT:  cmd — { email: String, password: String, name: String }
  OUTPUT: Ok(UserView) on success; Err(RegError) on a rule violation
  PRECONDITION: cmd is structurally valid (the screen has collected all fields)

  # The screen submits; the controller is the single orchestration point.
  result <- CALL CLS-regCtrl.register(cmd)

  [B1] IF result IS Ok THEN
    CALL CLS-registerScreen.navigate("/welcome")      # arm B1.then — success path
    RETURN result
  ELSE
    CALL CLS-registerScreen.showError(result.error)   # arm B1.else — EmailAlreadyTaken | WeakPassword
    RETURN result
  END
END
```

Coverage set for `register`: `B1.then`, `B1.else`. The per-rule branches
(`EmailAlreadyTaken`, `WeakPassword`) are owned by `CLS-regCtrl`'s SCoT and are
covered there; the integration ACs below exercise them end to end.

# Integration acceptance criteria

- **AC1** — *Given* no user exists with email `ada@example.com`, *When* `register`
  is invoked with a fresh email, an 8+ character password, and a valid name,
  *Then* exactly one user is persisted via `CLS-userRepo`, a `UserRegistered`
  event is emitted, the result is `Ok(UserView)`, and the screen navigates to
  `/welcome`. *(covers B1.then)*
- **AC2** — *Given* a user already exists with email `ada@example.com`, *When*
  `register` is invoked with the same email (any casing), *Then* the result is
  `Err(EmailAlreadyTaken)`, **no** new user is persisted, and the screen shows the
  "email already taken" message. *(covers B1.else)*
- **AC3** — *Given* no user exists with the given email, *When* `register` is
  invoked with a password shorter than 8 characters, *Then* the result is
  `Err(WeakPassword)`, **no** user is persisted, and the screen shows the "weak
  password" message. *(covers B1.else)*
