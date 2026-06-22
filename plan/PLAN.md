# Plan

> Project-seed template. Output of **Plan mode** (`/sdd-plan`), authored/owned by
> `plan-architect` per `.sdd/conventions.md` §9–§10. Copy this file, fill every
> section, keep it 100% English and valid Markdown. The plan decides WHAT indexes
> and specs will exist and in what order — it contains **no code and no specs**
> (those are written later by `spec-writer`). It is judged by `plan-gatekeeper`
> (verdict → `.sdd/state.md`, `.sdd/conventions.md` §6).
>
> Cross-cutting values (verbatim, binding):
> **Markdown is the source of truth (authority); reuse over repetition (DRY).**
>
> Conventions referenced (do not restate): layout `.sdd/conventions.md` §1, ids
> `.sdd/conventions.md` §2, front-matter `.sdd/conventions.md` §3, index splitting
> `.sdd/conventions.md` §1 (scaling option), topological order + vertical slices
> `.sdd/conventions.md` §12, `MOD-build` mandate `.sdd/conventions.md` §2. Entity
> ids here are **stable** and become the spec filenames downstream.

---

## Project classification (new vs existing-SDD)

> Pick exactly one. A **new** project: `plan-architect` authors `.sdd/scot.md`
> and `.sdd/ui-schema.md` once if absent, and writes/extends `.sdd/target.md` from
> the user prompt. An **existing-SDD** project: those canonical files are left
> untouched; `source:` paths point at real files, not proposed ones
> (`.sdd/conventions.md` §3, §9).

- **Classification:** `new` | `existing-SDD`
- **Rationale:** <one line: e.g. "greenfield, no `specs/` present" or "extending an existing SDD repo at `<path>`">

## Target stack summary -> `.sdd/target.md`

> A short prose summary of the chosen stack/architecture and the canonical
> build/test/run commands. The authoritative, full version lives in
> `.sdd/target.md`; this section records the decision and links to it. If the
> stack was unstated in the prompt, note the human's answer.

- **Language / runtime:** <e.g. TypeScript on Node>
- **Frameworks / libraries:** <e.g. HTTP framework, ORM, test runner>
- **Architecture:** <e.g. layered controller→service→repository>
- **Build / test / run commands:** see `.sdd/target.md` (canonical)
- **Budget overrides (if any):** <none | the §7 loop budgets this project overrides>

## Index granularity decision

> Default is one **global** index per level (`.sdd/conventions.md` §1). For a large
> multi-module project, the fine-grained indexes (`classes`, `model`,
> `ui-components`) MAY be split per module under
> `specs/indexes/<level>/<MOD-id>.index.md`; `modules` and `features` stay global.
> State the choice and the reason.

- **Decision:** `single global index per level` | `per-module split for {levels}`
- **Rationale:** <module/spec count and why the default does or does not scale>

## Entities to create/modify

> Every index entry the `spec-writer` must produce, in one table. `level` is one of
> `module | feature | model | class | ui-component | shared`. `id` follows
> `.sdd/conventions.md` §2 and becomes the spec filename. `depends_on` must form a
> DAG (cycles handled per §12 by introducing an `interface` spec first). `source`
> is **proposed** for NEW entities (from `.sdd/target.md` conventions) and the
> **real path** for MODIFY entities. `requirement` back-links to `REQ-` ids.

| id | level | module | depends_on | source (proposed/real) | requirement | NEW/MODIFY |
|----|-------|--------|------------|------------------------|-------------|------------|
| <MOD-x> | module | — | [] | <path or —> | [REQ-001] | NEW |
| <ENT-x> | model | <MOD-x> | [] | <path> | [REQ-001] | NEW |
| <CLS-x> | class | <MOD-x> | [<ids>] | <path> | [REQ-001] | NEW |

## Vertical slices (topological order)

> Group the entities into **vertical slices** (one feature/module each) and order
> the slices so dependencies come first (`.sdd/conventions.md` §12). In auto mode
> each slice runs plan→spec→code→test before the next. List each slice's `depends_on`
> closure so the slice is self-contained.

1. **Slice 1 — <feature/module name>**: <ids in this slice> — depends on: <prior slice ids or none>
2. **Slice 2 — <feature/module name>**: <ids in this slice> — depends on: <Slice 1 ids>

## Shared / cross-cutting candidates (for the reuse-analyst)

> Abstractions that look reusable across slices and should be promoted to
> `specs/shared/<SHR-…>.spec.md` (non-UI) or `specs/ui-components/<COMP-…>.spec.md`
> (UI) rather than duplicated. The `reuse-analyst` confirms or rejects each
> (`.sdd/conventions.md` §9). Flag these early so DRY is honored at spec time.

| candidate id | kind | why shared (where reused) | proposed home |
|--------------|------|---------------------------|---------------|
| <SHR-x> | shared | <used by CLS-a, CLS-b> | specs/shared/<SHR-x>.spec.md |

---

## Example (filled) — user registration

> A complete worked instance of the template above, consistent with the
> user-registration requirement (`requirements/REQUIREMENT.md`, REQ-001..REQ-005).
> Delete this section in a real project (or keep it as the seed example).

### Project classification (new vs existing-SDD)

- **Classification:** `new`
- **Rationale:** Greenfield repo; no `specs/` or `src/` present, so the canonical
  `.sdd/scot.md` and `.sdd/ui-schema.md` are authored once and `.sdd/target.md` is
  written from the prompt.

### Target stack summary -> `.sdd/target.md`

- **Language / runtime:** TypeScript on Node.
- **Frameworks / libraries:** HTTP framework for the controller layer, an ORM for
  persistence and migrations, a structured logger, and a unit/integration test
  runner. Concrete picks are recorded in `.sdd/target.md`.
- **Architecture:** Layered `controller → service → repository`, with domain
  entities and shared utilities; `MOD-build` owns config, dependency manifest, CI,
  and the DB migration derived from `ENT-user`.
- **Build / test / run commands:** see `.sdd/target.md` (canonical).
- **Budget overrides (if any):** none — the §7 defaults (analysis 3, code 3, test 5) apply.

### Index granularity decision

- **Decision:** single global index per level.
- **Rationale:** One feature slice and ~7 entities across two modules — well below
  the scale where per-module index splitting pays off. The default keeps lookups
  simple.

### Entities to create/modify

| id | level | module | depends_on | source (proposed/real) | requirement | NEW/MODIFY |
|----|-------|--------|------------|------------------------|-------------|------------|
| MOD-build      | module  | —       | []                                           | build/, package.json, ci config, migrations/ | [REQ-004] | NEW |
| MOD-api        | module  | —       | [MOD-build]                                  | src/api/                                     | [REQ-001] | NEW |
| ENT-user       | model   | MOD-api | []                                           | src/model/User.ts                            | [REQ-001, REQ-004] | NEW |
| SHR-passwordHasher | shared | MOD-api | [SHR-passwordHasher.iface]                | src/shared/passwordHasher.ts                 | [REQ-004] | NEW |
| SHR-passwordPolicy | shared | MOD-api | []                                        | src/shared/passwordPolicy.ts                 | [REQ-003] | NEW |
| CLS-userRepo   | class   | MOD-api | [ENT-user]                                   | src/api/UserRepository.ts                    | [REQ-002, REQ-004] | NEW |
| CLS-regCtrl    | class   | MOD-api | [CLS-userRepo, SHR-passwordHasher, SHR-passwordPolicy, ENT-user] | src/api/RegistrationController.ts | [REQ-001, REQ-002, REQ-003, REQ-005] | NEW |
| FEAT-001       | feature | MOD-api | [CLS-regCtrl]                                | []  (compositional — integration tests only) | [REQ-001, REQ-002, REQ-003, REQ-004, REQ-005] | NEW |

### Vertical slices (topological order)

1. **Slice 0 — Build/infra**: `MOD-build` — depends on: none. Establishes config,
   manifest, CI, and the migration scaffold so later slices have a place to land.
2. **Slice 1 — Register user (FEAT-001)**: `MOD-api`, `ENT-user`,
   `SHR-passwordPolicy`, `SHR-passwordHasher`, `CLS-userRepo`, `CLS-regCtrl`,
   `FEAT-001` — depends on: Slice 0. The full `depends_on` closure of `FEAT-001`,
   so the slice runs plan→spec→code→test end-to-end on its own.

### Shared / cross-cutting candidates (for the reuse-analyst)

| candidate id | kind | why shared (where reused) | proposed home |
|--------------|------|---------------------------|---------------|
| SHR-passwordHasher | shared | Salted hashing reused by registration now and by any future password-change / reset flow; isolating it keeps the hashing idiom in one place (DRY). | specs/shared/SHR-passwordHasher.spec.md |
| SHR-passwordPolicy | shared | Strength rules reused by registration and any future reset flow; a single policy avoids divergent rules across screens. | specs/shared/SHR-passwordPolicy.spec.md |
