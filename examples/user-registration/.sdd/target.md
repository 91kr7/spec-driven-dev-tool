# Target — stack, architecture & canonical commands

> **⚠️ FROZEN EXAMPLE.** Part of the `examples/user-registration/` teaching
> slice. Delete the example before real use. This mirrors the real
> `.sdd/target.md` an AI would write per project (see `.sdd/conventions.md` §1).
> Cross-cutting values still bind:
> **Markdown is the source of truth (authority); reuse over repetition (DRY).**

## Project

| Field            | Value                                                     |
|------------------|-----------------------------------------------------------|
| Project type     | new app                                                    |
| Architecture     | modular (per-module ownership; topological `depends_on`)  |
| Language         | TypeScript (Node 20)                                       |
| Backend          | Express                                                    |
| Frontend         | React                                                      |
| Build / package  | pnpm                                                       |
| Unit tests       | Vitest                                                     |
| Integration tests| Supertest (HTTP) over Vitest                              |
| UI tests         | Testing Library (React) over Vitest                       |
| Database         | Postgres                                                   |
| Migrations       | node-pg-migrate (migrations DERIVE from entity specs)     |
| Comment header   | `//`                                                       |

## Source path conventions

New specs propose `source:` paths from these conventions; existing specs point at
real files. The `source:` front-matter is authoritative — index `source` columns
are derived from it.

| Level                | Convention                                  | Example                                  |
|----------------------|---------------------------------------------|------------------------------------------|
| Domain entity (`ENT-`)| `src/db/<Name>.ts`                          | `src/db/User.ts`                         |
| Repository / DB class | `src/db/<Name>.ts`                          | `src/db/UserRepository.ts`               |
| API controller (`CLS-`)| `src/api/<Name>.ts`                        | `src/api/RegistrationController.ts`      |
| Web screen (`CLS-` gui)| `src/web/screens/<Name>.tsx`              | `src/web/screens/RegisterScreen.tsx`     |
| Shared non-UI (`SHR-`)| `src/shared/<name>.ts`                      | `src/shared/passwordHasher.ts`           |
| Shared UI (`COMP-`)   | `src/web/ui/<Name>.tsx`                     | `src/web/ui/Button.tsx`                  |
| Migrations (MOD-build)| `migrations/<timestamp>_<name>.ts`          | `migrations/0001_create_users.ts`        |
| Unit tests            | `tests/unit/<Name>.test.ts`                 | `tests/unit/RegistrationController.test.ts` |
| Integration tests     | `tests/int/<feature>.test.ts`               | `tests/int/registration.test.ts`         |
| UI tests              | `tests/ui/<Name>.test.tsx`                   | `tests/ui/RegisterScreen.test.tsx`       |

## Canonical commands

The single source of truth for how this project is built, tested and run. Agents
(`code-implementer`, `test-runner`, `code-gatekeeper`) use these verbatim.

| Purpose             | Command          |
|---------------------|------------------|
| Install deps        | `pnpm install`   |
| Build               | `pnpm build`     |
| Unit tests          | `pnpm test:unit` |
| Integration tests   | `pnpm test:int`  |
| All tests           | `pnpm test`      |
| Lint                | `pnpm lint`      |
| Run (dev)           | `pnpm dev`       |
| DB migrate          | `pnpm migrate`   |

## Source-file traceability header

Every generated source file carries a header pointing back to its spec
(`.sdd/conventions.md` §13), using this project's `//` comment syntax:

```
// spec: CLS-regCtrl RegistrationController — specs/classes/CLS-regCtrl.spec.md
```

## Iteration budgets

Defaults from `.sdd/conventions.md` §7 (not overridden for this example):

| Loop     | Budget | On overflow           |
|----------|--------|-----------------------|
| analysis | 3      | escalate to the human |
| code     | 3      | escalate to the human |
| test     | 5      | escalate to the human |
