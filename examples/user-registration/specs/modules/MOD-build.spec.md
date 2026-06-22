---
id: MOD-build
name: Build/Infra
kind: module
module: MOD-build
status: reviewed
depends_on: [MOD-db]
requirements: [REQ-001]
source: [package.json, pnpm-workspace.yaml, tsconfig.json, vitest.config.ts, .github/workflows/ci.yml, migrations/0001_create_users.ts]
---

> **⚠️ FROZEN EXAMPLE** — part of `examples/user-registration/`. Delete before
> real use. Conventions: `.sdd/conventions.md` (§2 — `MOD-build` is mandatory and
> owns DB migrations DERIVED from entity specs). Commands/stack come from
> `.sdd/target.md`. Values:
> **Markdown is the source of truth (authority); reuse over repetition (DRY).**

# Purpose

`MOD-build` is the **mandatory Build/Infra** module. It owns build files,
dependency manifests, project config, CI, and the **database migration for the
users table, derived from `ENT-user`** (migrations are never hand-authored
independently — `.sdd/conventions.md` §2).

# Public interface

- **Owns:** the build/config files and the `users` migration listed in `source:`.
- **Inputs:** the stack + canonical commands declared in `.sdd/target.md`; the
  field/invariant set of `ENT-user`.
- **Outputs:** a buildable, testable, runnable project (`pnpm` scripts) and a
  schema that satisfies the `ENT-user` invariants.
- **Depends on:** `MOD-db` (`depends_on: [MOD-db]`) — it derives the `users`
  migration from `ENT-user`, which `MOD-db` owns; so it is processed **after**
  `MOD-db` in topological order.

# Invariants & rules

- The `users` migration is **derived** from `ENT-user` — every entity field maps
  to a column, every entity invariant maps to a constraint. If `ENT-user`
  changes, the migration is regenerated; the spec stays authority.
- Canonical scripts in `package.json` MUST match `.sdd/target.md`: `install`,
  `build`, `test:unit`, `test:int`, `test`, `lint`, `dev`, `migrate`.
- CI runs `pnpm install`, `pnpm lint`, `pnpm build`, then `pnpm test`.

# Build & migration map

| Concern        | Artifact (derived path)              | Source of truth         |
|----------------|--------------------------------------|-------------------------|
| Package + scripts | `package.json`                     | `.sdd/target.md`        |
| Workspace      | `pnpm-workspace.yaml`                | `.sdd/target.md`        |
| TS config      | `tsconfig.json`                      | `.sdd/target.md`        |
| Test config    | `vitest.config.ts`                   | `.sdd/target.md`        |
| CI pipeline    | `.github/workflows/ci.yml`           | `.sdd/target.md`        |
| Users migration| `migrations/0001_create_users.ts`    | **`ENT-user`** spec     |

## Derived `users` table (from `ENT-user`)

| Column          | Type / rule (from entity)                         | Entity field   |
|-----------------|---------------------------------------------------|----------------|
| `id`            | `uuid` PRIMARY KEY                                 | `id`           |
| `email`         | `text` NOT NULL, UNIQUE on `lower(email)`          | `email`        |
| `name`          | `text` NOT NULL, CHECK length 1..100              | `name`         |
| `password_hash` | `text` NOT NULL                                    | `passwordHash` |
| `created_at`    | `timestamptz` NOT NULL DEFAULT `now()`            | `createdAt`    |

# Acceptance criteria

- **AC1** — Given `.sdd/target.md`, When `package.json` is generated, Then it
  defines scripts whose commands equal the canonical commands verbatim.
- **AC2** — Given `ENT-user`, When the `users` migration is generated, Then it
  creates the table above with a **case-insensitive unique** email constraint and
  a `name` length CHECK of 1..100.
