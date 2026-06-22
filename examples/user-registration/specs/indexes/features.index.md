# Features index

> **⚠️ FROZEN EXAMPLE** — part of `examples/user-registration/`. Delete before
> real use. One index PER LEVEL (`.sdd/conventions.md` §1, §4). `status` is the
> canonical lifecycle home; `source` is **derived** from each spec's `source:`.
> A purely-compositional feature has `source: []` and only drives integration
> tests. Values:
> **Markdown is the source of truth (authority); reuse over repetition (DRY).**

| id | name | description | module | depends_on | spec | source | status |
|----|------|-------------|--------|------------|------|--------|--------|
| FEAT-001 | User registration | A new user submits the registration form and, on a unique email + acceptable password, is persisted and welcomed. | MOD-api | CLS-registerScreen, CLS-regCtrl, CLS-userRepo, ENT-user, SHR-passwordHasher | specs/features/FEAT-001.spec.md | — | reviewed |

`FEAT-001` is **cross-module** (its home is `MOD-api`, but it orchestrates
`MOD-web` → `MOD-api` → `MOD-db`) and **purely compositional** (`source: []`):
no coordinator code of its own; it only drives integration tests.
