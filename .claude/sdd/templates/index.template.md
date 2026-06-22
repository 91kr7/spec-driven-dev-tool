<!--
  TEMPLATE — level index (specs/indexes/<level>.index.md).
  There is ONE index PER LEVEL (not per module): modules, features, model, classes,
  ui-components. Copy this file to the level you are authoring and keep ONLY that
  level's table shape:
    - specs/indexes/modules.index.md      → base columns
    - specs/indexes/features.index.md     → base columns
    - specs/indexes/model.index.md        → base columns
    - specs/indexes/classes.index.md      → base columns
    - specs/indexes/ui-components.index.md → base columns + `layer` + `variants`
  Authority: .claude/sdd/conventions.md — §4 (index row schema), §2 (id scheme), §3 (front-matter,
             from which `source` is DERIVED), §5 (status lifecycle is owned HERE).
             ui-components form also obeys .claude/sdd/ui-schema.md (§7 layers).
  Values that bind this artifact: Markdown is the source of truth (authority); reuse over repetition (DRY).
  Placeholders below are intentional. The "## Filled example" section at the bottom shows every
  level with real rows — keep it as a reference, delete it from a real index.
-->
# `<level>` index

> **What this file is.** The canonical roster of **every** entry at one level
> (`modules` | `features` | `model` | `classes` | `ui-components`). Agents **read
> this index first** and open only the individual `*.spec.md` files they actually
> need — this is the **lazy-loading / token-saving** boundary of the workflow.
>
> **Status lives here.** The `status` column is the **canonical** lifecycle home
> for each entry (`draft → reviewed → approved`, see conventions §5). Each spec's
> `status:` front-matter merely **mirrors** this column; if they disagree, the
> index wins. Only the slash command (main session) advances `status`, from the
> latest verdict in `.sdd/state.md` — gatekeepers and authors never edit it.
>
> **`source` is derived.** The `source` column is filled **from each spec's
> `source:` front-matter** (a command derives it; never hand-edited here). The
> spec front-matter is the single authoritative spec→source mapping.

<!-- COLUMN CONTRACT (conventions §4):
     | id | name | description (WHAT, one line) | module | depends_on | spec | source | status |
     - id          — the entry's stable id (conventions §2 prefix for this level).
     - name        — human/PascalCase name.
     - description — WHAT it represents, one line; NEVER how it is built.
     - module      — the level's home module (MOD-*). For the modules index this is the module itself.
     - depends_on  — comma-separated ids this entry depends on (topological order, deps first); `—` if none.
     - spec        — relative path to the spec file; `—` for purely-compositional features with no body.
     - source      — DERIVED from the spec's `source:` front-matter; `—` when source: [] (compositional).
     - status      — draft | reviewed | approved (canonical; see §5).
     The ui-components index adds two columns AFTER `module`: `layer` (atom|molecule|organism|layout)
     and `variants` (comma-separated variant names, or `—`). -->

| id | name | description (WHAT, one line) | module | depends_on | spec | source | status |
|----|------|------------------------------|--------|------------|------|--------|--------|
| `<PREFIX-id>` | `<Name>` | `<one-line WHAT, never how>` | `<MOD-id>` | `<dep-id, dep-id | —>` | `<specs/<level>/<id>.spec.md>` | `<src/... | —>` | `<draft|reviewed|approved>` |
| `<PREFIX-id2>` | `<Name2>` | `<one-line WHAT>` | `<MOD-id>` | `<— | dep-ids>` | `<specs/<level>/<id2>.spec.md>` | `<src/...>` | `<draft>` |

<!-- For specs/indexes/ui-components.index.md ONLY, use this shape instead (note `layer` + `variants`):

| id | name | description (WHAT, one line) | module | layer | variants | depends_on | spec | source | status |
|----|------|------------------------------|--------|-------|----------|------------|------|--------|--------|
| `<COMP-id>` | `<Name>` | `<one-line WHAT>` | `<MOD-ui>` | `<atom|molecule|organism|layout>` | `<v1, v2 | —>` | `<dep-ids | —>` | `<specs/ui-components/<id>.spec.md>` | `<src/ui/...>` | `<draft>` |
-->

<!-- ===================================================================== -->
<!-- The blank template ends here. Below are complete, copy-correct        -->
<!-- examples for EVERY level so a reader sees all column variants.        -->
<!-- Delete the whole section when authoring a real index.                 -->
<!-- ===================================================================== -->

## Filled example

The five blocks below are five **separate** files in a real project (one index per
level). They are gathered here only so the template shows every column variant.

### `specs/indexes/modules.index.md` — modules

<!-- The modules index partitions the system. `module` repeats the id (a module is its own home).
     MOD-build is the mandatory infra module (build files, manifests, config, CI, derived migrations). -->

| id | name | description (WHAT, one line) | module | depends_on | spec | source | status |
|----|------|------------------------------|--------|------------|------|--------|--------|
| `MOD-build` | Build & Infra | Build files, dependency manifests, config, CI, and entity-derived DB migrations | `MOD-build` | — | `specs/modules/MOD-build.spec.md` | `package.json, db/migrations/` | approved |
| `MOD-api` | API | HTTP controllers and request/response wiring for the public API | `MOD-api` | `MOD-domain` | `specs/modules/MOD-api.spec.md` | `src/api/` | reviewed |
| `MOD-domain` | Domain | Core entities, services, and business rules of the application | `MOD-domain` | — | `specs/modules/MOD-domain.spec.md` | `src/domain/` | reviewed |

### `specs/indexes/features.index.md` — features

<!-- A use-case feature. A purely-compositional feature (no coordinator code) has spec `—` semantics
     only when it carries no body; otherwise it points at a use-case spec. `source` is `—` when source: []. -->

| id | name | description (WHAT, one line) | module | depends_on | spec | source | status |
|----|------|------------------------------|--------|------------|------|--------|--------|
| `FEAT-001` | Register user | A visitor creates an account with email, password, and display name | `MOD-api` | `CLS-regCtrl`, `CLS-userService`, `ENT-user` | `specs/features/FEAT-001.spec.md` | `src/app/RegisterUser.ts` | approved |
| `FEAT-002` | Sign in | A registered user authenticates and receives a session | `MOD-api` | `CLS-authCtrl`, `CLS-userService` | `specs/features/FEAT-002.spec.md` | `src/app/SignIn.ts` | reviewed |
| `FEAT-003` | Account dashboard | Compose the signed-in landing view from existing screens and widgets | `MOD-api` | `FEAT-002`, `CLS-dashboardScreen` | `specs/features/FEAT-003.spec.md` | — | draft |

### `specs/indexes/model.index.md` — domain model (entities)

<!-- Structural entities: fields/relations/invariants. Entities map to model files; MOD-build derives
     migrations from these specs (migrations are never hand-authored). -->

| id | name | description (WHAT, one line) | module | depends_on | spec | source | status |
|----|------|------------------------------|--------|------------|------|--------|--------|
| `ENT-user` | User | A registered person who can sign in and own resources | `MOD-domain` | — | `specs/model/ENT-user.spec.md` | `src/domain/model/User.ts` | approved |
| `ENT-session` | Session | An authenticated session bound to a user with an expiry | `MOD-domain` | `ENT-user` | `specs/model/ENT-session.spec.md` | `src/domain/model/Session.ts` | reviewed |
| `ENT-role` | Role | A named permission grouping assignable to users | `MOD-domain` | — | `specs/model/ENT-role.spec.md` | `src/domain/model/Role.ts` | draft |

### `specs/indexes/classes.index.md` — classes (services, controllers, screens)

<!-- Behavioral classes (SCoT body) plus feature-specific gui screens (kind: gui, UI-schema body).
     Shared non-UI abstractions (SHR-*) also live at this level via specs/shared/. -->

| id | name | description (WHAT, one line) | module | depends_on | spec | source | status |
|----|------|------------------------------|--------|------------|------|--------|--------|
| `CLS-userService` | UserService | Owns the user lifecycle use cases (create, fetch, validate) | `MOD-domain` | `CLS-userRepo`, `SHR-passwordHasher`, `ENT-user` | `specs/classes/CLS-userService.spec.md` | `src/domain/UserService.ts` | approved |
| `CLS-regCtrl` | RegistrationController | Handles the HTTP registration request and maps it to the use case | `MOD-api` | `CLS-userService` | `specs/classes/CLS-regCtrl.spec.md` | `src/api/RegistrationController.ts` | reviewed |
| `SHR-passwordHasher` | PasswordHasher | Hashes and verifies passwords behind a stable interface | `MOD-domain` | — | `specs/shared/SHR-passwordHasher.spec.md` | `src/domain/PasswordHasher.ts` | approved |
| `CLS-registerScreen` | RegisterScreen | The registration screen composing library components for FEAT-001 | `MOD-api` | `COMP-formField`, `COMP-button` | `specs/classes/CLS-registerScreen.spec.md` | `src/web/screens/RegisterScreen.tsx` | draft |

### `specs/indexes/ui-components.index.md` — shared UI component library

<!-- ADDS `layer` and `variants` columns (conventions §4; ui-schema §7). Higher layers compose lower
     layers BY ID. `variants` lists named variants (ui-schema §6) or `—`. -->

| id | name | description (WHAT, one line) | module | layer | variants | depends_on | spec | source | status |
|----|------|------------------------------|--------|-------|----------|------------|------|--------|--------|
| `COMP-button` | Button | A clickable action control with text and an emitted click event | `MOD-ui` | atom | `primary, secondary, ghost, danger` | — | `specs/ui-components/COMP-button.spec.md` | `src/ui/atoms/Button.tsx` | approved |
| `COMP-textInput` | TextInput | A single-line text entry control bound to a value | `MOD-ui` | atom | `text, email, password` | — | `specs/ui-components/COMP-textInput.spec.md` | `src/ui/atoms/TextInput.tsx` | approved |
| `COMP-formField` | FormField | A labelled control with an inline validation message | `MOD-ui` | molecule | — | `COMP-textInput` | `specs/ui-components/COMP-formField.spec.md` | `src/ui/molecules/FormField.tsx` | reviewed |
| `COMP-appShell` | AppShell | The app frame (header, body, footer slots) hosting every screen | `MOD-ui` | layout | — | `COMP-header`, `COMP-footer` | `specs/ui-components/COMP-appShell.spec.md` | `src/ui/layout/AppShell.tsx` | draft |
