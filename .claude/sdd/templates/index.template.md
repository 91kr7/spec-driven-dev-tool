<!--
  TEMPLATE — index files. TWO kinds (conventions §4):
    1. GLOBAL      .sdd/specs/modules.index.md          — one row per MODULE (the skeleton).
    2. PER-MODULE  .sdd/specs/<MOD>/<MOD>.index.md       — one row per entity in that module, ALL levels (lazy).
  Authority: conventions §1 (layout), §4 (row schema), §2 (ids), §3 (front-matter — `source` DERIVED), §5 (status lives HERE).
  Markdown = source of truth (authority); reuse over repetition (DRY).
  Delete "## Filled example" when authoring a real index.
-->

# Global module index — `.sdd/specs/modules.index.md`

> Architectural skeleton: list **every** `MOD-*` with its `depends_on` and `status`. Agents read this
> first to map the module graph, then open only the per-module index they need (lazy loading).

**Columns (conventions §4):** `id` (`MOD-*`) · `name` · `description` (WHAT, one line) · `depends_on` (comma-separated `MOD-*`, deps first, `—` if none) · `spec` (`.sdd/specs/<MOD>/<MOD>.spec.md`) · `source` (derived, `—` when `source: []`) · `status` (draft|reviewed|implemented|approved).

| id | name | description (WHAT, one line) | depends_on | spec | source | status |
|----|------|------------------------------|------------|------|--------|--------|
| `<MOD-id>` | `<Name>` | `<one-line WHAT>` | `<MOD-dep, … \| —>` | `.sdd/specs/<MOD-id>/<MOD-id>.spec.md` | `<src/… \| —>` | `<draft\|reviewed\|implemented\|approved>` |

---

# Per-module index — `.sdd/specs/<MOD>/<MOD>.index.md`

> Roster of **every** entity in this module, across all levels. **Drops** the `module`
> column (the folder IS the module); **adds** `level`. If the module contains any `ui-component` row,
> **append** `layer` + `variants` after `level`. `status` = canonical lifecycle home (only the
> command advances it); `source` **derived** from each spec's `source:`. Fill **every** column.

**Columns:** `id` (§2 prefix) · `name` · `description` (WHAT, one line) · `level` (feature|class|entity|ui-component|shared) · `depends_on` (ids, deps first, `—` if none) · `spec` (path) · `source` (derived, `—` when `source: []`) · `status`.

| id | name | description (WHAT, one line) | level | depends_on | spec | source | status |
|----|------|------------------------------|-------|------------|------|--------|--------|
| `<PREFIX-id>` | `<Name>` | `<one-line WHAT>` | `<level>` | `<dep-id, … \| —>` | `.sdd/specs/<MOD>/<level>/<id>.spec.md` | `<src/… \| —>` | `<draft\|reviewed\|implemented\|approved>` |

---

## Filled example

**Global `modules.index.md`:**

| id | name | description (WHAT, one line) | depends_on | spec | source | status |
|----|------|------------------------------|------------|------|--------|--------|
| `MOD-build` | Build & scaffolding | Build files, manifests, config, app entry | — | `.sdd/specs/MOD-build/MOD-build.spec.md` | `package.json, tsconfig.json` | approved |
| `MOD-shared` | Shared | Domain-agnostic primitives — design-system kit + generic utils (dependency sink) | `MOD-build` | `.sdd/specs/MOD-shared/MOD-shared.spec.md` | — | reviewed |
| `MOD-domain` | Domain | User lifecycle + persistence | `MOD-build` | `.sdd/specs/MOD-domain/MOD-domain.spec.md` | — | approved |

**Per-module `MOD-domain.index.md`** (no UI → base columns):

| id | name | description (WHAT, one line) | level | depends_on | spec | source | status |
|----|------|------------------------------|-------|------------|------|--------|--------|
| `CLS-userService` | UserService | Owns the user lifecycle use cases | class | `CLS-userRepo, SHR-passwordHasher, ENT-user` | `.sdd/specs/MOD-domain/classes/CLS-userService.spec.md` | `src/domain/UserService.ts` | approved |
| `ENT-user` | User | The user aggregate | entity | — | `.sdd/specs/MOD-domain/model/ENT-user.spec.md` | `src/domain/User.ts` | approved |

**Per-module `MOD-shared.index.md`** (contains UI → appends `layer` + `variants` after `level`):

| id | name | description (WHAT, one line) | level | layer | variants | depends_on | spec | source | status |
|----|------|------------------------------|-------|-------|----------|------------|------|--------|--------|
| `SHR-passwordHasher` | PasswordHasher | Hashes/verifies passwords behind a stable interface | shared | — | — | — | `.sdd/specs/MOD-shared/shared/SHR-passwordHasher.spec.md` | `src/shared/PasswordHasher.ts` | reviewed |
| `COMP-button` | Button | A clickable action control | ui-component | atom | `primary, secondary, ghost, danger` | — | `.sdd/specs/MOD-shared/ui-components/COMP-button.spec.md` | `src/ui/atoms/Button.tsx` | approved |
