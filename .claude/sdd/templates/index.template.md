<!--
<instructions>
TEMPLATE: index files. Two kinds:
1. GLOBAL: `.sdd/specs/modules.index.md` (one row per MODULE).
2. PER-MODULE: `.sdd/specs/<MOD>/<MOD>.index.md` (one row per entity in that module).
Authority: conventions §1 (layout), §4 (schema), §2 (ids), §3 (front-matter), §5 (status).
DRY. Delete the `<example>` blocks before saving.
</instructions>
-->

# Global module index — `.sdd/specs/modules.index.md`
<instruction>The architectural skeleton: every `MOD-*` + its `depends_on` + `status`.</instruction>

**Columns:** `id` (`MOD-*`) · `name` · `description` (WHAT, one line) · `depends_on` (comma-separated `MOD-*`, deps first, `—` if none) · `spec` (`.sdd/specs/<MOD>/<MOD>.spec.md`) · `source` (derived, `—` when `source: []`) · `status` (draft|reviewed|implemented|approved).

| id | name | description (WHAT, one line) | depends_on | spec | source | status |
|----|------|------------------------------|------------|------|--------|--------|
| `<MOD-id>` | `<Name>` | `<one-line WHAT>` | `<MOD-dep, … \| —>` | `.sdd/specs/<MOD-id>/<MOD-id>.spec.md` | `<src/… \| —>` | `<draft\|reviewed\|implemented\|approved>` |

---

# Per-module index — `.sdd/specs/<MOD>/<MOD>.index.md`
<instruction>Every entity in this module. Drops `module` column, adds `level`. If module has `ui-component`, appends `layer` + `variants` after `level`. `status` is canonical; `source` is derived.</instruction>

**Columns:** `id` (§2 prefix) · `name` · `description` (WHAT, one line) · `level` (feature|class|entity|ui-component|shared) · `depends_on` (ids, deps first, `—` if none) · `spec` (path) · `source` (derived, `—` when `source: []`) · `status`.

| id | name | description (WHAT, one line) | level | depends_on | spec | source | status |
|----|------|------------------------------|-------|------------|------|--------|--------|
| `<PREFIX-id>` | `<Name>` | `<one-line WHAT>` | `<level>` | `<dep-id, … \| —>` | `.sdd/specs/<MOD>/<level>/<id>.spec.md` | `<src/… \| —>` | `<draft\|reviewed\|implemented\|approved>` |

---
<example name="modules.index.md">
| id | name | description (WHAT, one line) | depends_on | spec | source | status |
|----|------|------------------------------|------------|------|--------|--------|
| `MOD-build` | Build & scaffolding | Build files, manifests, config, app entry | — | `.sdd/specs/MOD-build/MOD-build.spec.md` | `package.json, tsconfig.json` | approved |
| `MOD-shared` | Shared | Cross-cutting abstractions used by ≥2 modules | `MOD-build` | `.sdd/specs/MOD-shared/MOD-shared.spec.md` | — | reviewed |
| `MOD-domain` | Domain | User lifecycle + persistence | `MOD-build` | `.sdd/specs/MOD-domain/MOD-domain.spec.md` | — | approved |
</example>

<example name="MOD-domain.index.md (no UI)">
| id | name | description (WHAT, one line) | level | depends_on | spec | source | status |
|----|------|------------------------------|-------|------------|------|--------|--------|
| `CLS-userService` | UserService | Owns the user lifecycle use cases | class | `CLS-userRepo, SHR-passwordHasher, ENT-user` | `.sdd/specs/MOD-domain/classes/CLS-userService.spec.md` | `src/domain/UserService.ts` | approved |
| `ENT-user` | User | The user aggregate | entity | — | `.sdd/specs/MOD-domain/model/ENT-user.spec.md` | `src/domain/User.ts` | approved |
</example>

<example name="MOD-shared.index.md (contains UI)">
| id | name | description (WHAT, one line) | level | layer | variants | depends_on | spec | source | status |
|----|------|------------------------------|-------|-------|----------|------------|------|--------|--------|
| `SHR-passwordHasher` | PasswordHasher | Hashes/verifies passwords behind a stable interface | shared | — | — | — | `.sdd/specs/MOD-shared/shared/SHR-passwordHasher.spec.md` | `src/shared/PasswordHasher.ts` | reviewed |
| `COMP-button` | Button | A clickable action control | ui-component | atom | `primary, secondary, ghost, danger` | — | `.sdd/specs/MOD-shared/ui-components/COMP-button.spec.md` | `src/ui/atoms/Button.tsx` | approved |
</example>
