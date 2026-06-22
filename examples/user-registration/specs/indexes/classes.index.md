<!--
  FROZEN, ILLUSTRATIVE EXAMPLE — NOT part of the tool's real specs/.
  Part of the "User registration (Database + API)" worked example.
  Delete examples/user-registration/ before real use.
  Authority: .sdd/conventions.md (§4 index row schema, §5 status lifecycle).
  Markdown is the source of truth (authority); reuse over repetition (DRY).
-->

# Classes index — `CLS-*` (worked example)

> One index row PER class/service/screen — and per shared non-UI abstraction
> (`SHR-*`), which are implementation units indexed here too (conventions §4).
> Agents read this index
> first and open only the specs they need. `source` is **derived** from each
> spec's `source:` front-matter — never hand-edited independently. `status` is the
> **canonical** lifecycle home for each entry; the spec front-matter mirrors it.

| id | name | description | module | depends_on | spec | source | status |
|----|------|-------------|--------|------------|------|--------|--------|
| `CLS-userRepo` | UserRepository | Persistence boundary for users: existence check by email, save, lookup by email | `MOD-db` | `ENT-user` | `specs/classes/CLS-userRepo.spec.md` | `src/db/UserRepository.ts` | reviewed |
| `CLS-regCtrl` | RegistrationController | Registration use-case entry: validates rules, hashes, persists, returns a user view or a typed error | `MOD-api` | `CLS-userRepo`, `SHR-passwordHasher`, `ENT-user` | `specs/classes/CLS-regCtrl.spec.md` | `src/api/RegistrationController.ts` | reviewed |
| `CLS-registerScreen` | RegisterScreen | Account-creation screen: collects email/password/name, submits via the controller, navigates on success | `MOD-web` | `CLS-regCtrl`, `COMP-formField`, `COMP-textInput`, `COMP-button` | `specs/classes/CLS-registerScreen.spec.md` | `src/web/screens/RegisterScreen.tsx` | reviewed |
| `SHR-passwordHasher` | PasswordHasher | Shared one-way password hashing + verification contract (interface; stub auto-derived) | `MOD-api` | — | `specs/shared/SHR-passwordHasher.spec.md` | `src/shared/passwordHasher.ts` | reviewed |

## Notes

- `CLS-userRepo` is an `interface` (kind: interface) — a **stub/mock is
  auto-derived** from it (conventions §3); it is never specced separately.
- `CLS-regCtrl` carries a full SCoT body with branch ids `B1`/`B2`; its coverage
  set (`B1.then, B1.else, B2.then, B2.else`) plus its ACs are what the test-writer
  must cover (scot.md §7).
- `CLS-registerScreen` is a `gui` screen — it **composes UI components by id** and
  never re-describes them (ui-schema.md §3). The layout primitives it uses
  (`COMP-appShell/header/body/footer/panel/stack`) come from the shared SDD UI
  library and are referenced, not duplicated, so they do **not** appear as rows in
  this example's `ui-components.index.md`.
- `SHR-passwordHasher` is a **shared non-UI abstraction** (`specs/shared/`,
  `kind: interface`). Shared abstractions are implementation units, so they live
  in **this** index — there is no separate shared index (conventions §4); a stub
  is auto-derived from the interface.
