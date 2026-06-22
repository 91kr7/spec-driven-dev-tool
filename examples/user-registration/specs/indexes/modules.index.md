# Modules index

> **⚠️ FROZEN EXAMPLE** — part of `examples/user-registration/`. Delete before
> real use. One index PER LEVEL (`.sdd/conventions.md` §1, §4). The index `status`
> column is the **canonical** lifecycle home; `source` is **derived** from each
> spec's `source:` front-matter. Values:
> **Markdown is the source of truth (authority); reuse over repetition (DRY).**

| id | name | description | module | depends_on | spec | source | status |
|----|------|-------------|--------|------------|------|--------|--------|
| MOD-db | Persistence | Owns the user data model and its repository (data access). | MOD-db | — | specs/modules/MOD-db.spec.md | — | reviewed |
| MOD-api | HTTP API | Owns the registration HTTP endpoint and its controller logic. | MOD-api | MOD-db | specs/modules/MOD-api.spec.md | — | reviewed |
| MOD-web | Web UI | Owns the registration screen; consumes the shared UI library. | MOD-web | MOD-api | specs/modules/MOD-web.spec.md | — | reviewed |
| MOD-build | Build/Infra | Owns build files, manifests, CI, and the users-table migration derived from ENT-user. | MOD-build | MOD-db | specs/modules/MOD-build.spec.md | — | reviewed |

Topological order (dependencies first): `MOD-db` → (`MOD-build`, `MOD-api`) →
`MOD-web`. `MOD-build` follows `MOD-db` because it derives the `users` migration
from `ENT-user` (owned by `MOD-db`).
