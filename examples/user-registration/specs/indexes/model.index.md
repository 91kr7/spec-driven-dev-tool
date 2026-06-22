# Model index

> **⚠️ FROZEN EXAMPLE** — part of `examples/user-registration/`. Delete before
> real use. One index PER LEVEL (`.sdd/conventions.md` §1, §4). `status` is the
> canonical lifecycle home; `source` is **derived** from each spec's `source:`.
> The entity is the single source from which the TS type AND the Postgres
> migration (in MOD-build) derive. Values:
> **Markdown is the source of truth (authority); reuse over repetition (DRY).**

| id | name | description | module | depends_on | spec | source | status |
|----|------|-------------|--------|------------|------|--------|--------|
| ENT-user | User | A registered person: identity, login email, display name, and password hash. | MOD-db | — | specs/model/ENT-user.spec.md | src/db/User.ts | reviewed |
