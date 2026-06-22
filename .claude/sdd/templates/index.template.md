<!--
  TEMPLATE — level index (specs/indexes/<level>.index.md). One index PER LEVEL:
  modules, features, model, classes, ui-components. Authority: conventions §4 (row schema),
  §2 (ids), §3 (front-matter — `source` is DERIVED from it), §5 (status lives HERE).
  Markdown is the source of truth (authority); reuse over repetition (DRY).
  Delete the "## Filled example" when authoring a real index.
-->
# `<level>` index

> Canonical roster of **every** entry at one level. Agents read this first, then open
> only the specs they need (lazy loading). `status` is the **canonical** lifecycle home
> (only the command advances it). `source` is **derived** from each spec's `source:`
> front-matter by the authoring agent. The authoring agent fills **every** column.

**Columns (conventions §4):** `id` (§2 prefix) · `name` · `description` (WHAT, one line, never how) · `module` (home MOD-*) · `depends_on` (comma-separated ids, deps first, `—` if none) · `spec` (path, `—` if none) · `source` (derived, `—` when `source: []`) · `status` (draft|reviewed|approved).
**ui-components.index.md only** adds two columns after `module`: `layer` (atom|molecule|organism|layout) + `variants` (or `—`).

| id | name | description (WHAT, one line) | module | depends_on | spec | source | status |
|----|------|------------------------------|--------|------------|------|--------|--------|
| `<PREFIX-id>` | `<Name>` | `<one-line WHAT>` | `<MOD-id>` | `<dep-id, … | —>` | `<specs/<level>/<id>.spec.md>` | `<src/… | —>` | `<draft|reviewed|approved>` |

---

## Filled example

**Base columns** (modules / features / model / classes all share this shape; `classes.index.md` also hosts `SHR-*` rows):

| id | name | description (WHAT, one line) | module | depends_on | spec | source | status |
|----|------|------------------------------|--------|------------|------|--------|--------|
| `CLS-userService` | UserService | Owns the user lifecycle use cases | `MOD-domain` | `CLS-userRepo, SHR-passwordHasher, ENT-user` | `specs/classes/CLS-userService.spec.md` | `src/domain/UserService.ts` | approved |
| `SHR-passwordHasher` | PasswordHasher | Hashes/verifies passwords behind a stable interface | `MOD-domain` | — | `specs/shared/SHR-passwordHasher.spec.md` | `src/domain/PasswordHasher.ts` | reviewed |
| `FEAT-003` | Account dashboard | Compose the signed-in landing view (compositional) | `MOD-api` | `FEAT-002, CLS-dashboardScreen` | `specs/features/FEAT-003.spec.md` | — | draft |

**ui-components.index.md** (adds `layer` + `variants`; higher layers compose lower by id):

| id | name | description (WHAT, one line) | module | layer | variants | depends_on | spec | source | status |
|----|------|------------------------------|--------|-------|----------|------------|------|--------|--------|
| `COMP-button` | Button | A clickable action control | `MOD-ui` | atom | `primary, secondary, ghost, danger` | — | `specs/ui-components/COMP-button.spec.md` | `src/ui/atoms/Button.tsx` | approved |
| `COMP-formField` | FormField | A labelled control with inline validation | `MOD-ui` | molecule | — | `COMP-textInput` | `specs/ui-components/COMP-formField.spec.md` | `src/ui/molecules/FormField.tsx` | draft |
