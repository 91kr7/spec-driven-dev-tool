<!--
  TEMPLATE — MODULE spec (kind: module, structural overview). Schema per .claude/sdd/conventions.md §3.
  A module partitions the system and names the entries (FEAT/CLS/ENT/COMP/SHR) it contains.
  It carries no SCoT and no UI schematic — those belong to the entries it contains.
  Copy to specs/modules/MOD-<kebab>.spec.md; the "## Filled example" (MOD-build) is a sample — delete it from a real spec.
  Markdown is the source of truth (authority); reuse over repetition (DRY).
-->
---
id: MOD-<kebab>                 # required — MUST match the filename: specs/modules/MOD-<kebab>.spec.md
name: <HumanModuleName>         # required — human name (e.g. "API Layer", "Build & Infrastructure")
kind: module                    # required — fixed for module specs; drives the structural-overview FORM
module: MOD-<kebab>             # required — self-reference (a module IS its own home module)
status: draft                   # draft|reviewed|approved — MIRRORS modules.index.md (the index is canonical)
depends_on: [MOD-<other>, ...]  # other MODULE ids this module depends on — topological order; [] if none
requirements: [REQ-<nnn>, ...]  # requirement id(s) this module serves (back-link); [] if purely infra
source: []                      # a module is a logical grouping, not a single file → usually []. EXCEPTION: MOD-build owns its infra FILES directly here (build manifest, config, CI, framework/e2e config, forward schema scripts) — see the filled example (conventions §2).
owns_sections: []               # modules do not co-own aggregator files → []
---

# Purpose

<!--
GUIDANCE: One short paragraph. State WHAT this module is responsible for — the
single cohesive concern it owns — never HOW it is built. A reader should be able
to decide, from this paragraph alone, whether a new entry belongs here or in a
sibling module. Keep it boundary-defining, not a feature list (the entries are
listed below).
-->

<One paragraph: the cohesive responsibility this module owns and what falls
outside it.>

# Contained entries

<!--
GUIDANCE: The authoritative list of entries that BELONG to this module. Every id
here MUST have `module: MOD-<this>` in its own spec front-matter and a matching
row in the per-level index. List ids only (one line each, with a one-line WHAT) —
do NOT restate each entry's spec (DRY: reuse over repetition). Group by level for
readability. Omit a group if the module has none of that level.
-->

| Level     | Entry id            | What it represents (one line)                 |
|-----------|---------------------|------------------------------------------------|
| Feature   | `FEAT-<nnn>`        | <use-case this module orchestrates>            |
| Entity    | `ENT-<kebab>`       | <domain entity owned here>                      |
| Class     | `CLS-<lowerCamel>`  | <service / controller / screen owned here>      |
| Component | `COMP-<lowerCamel>` | <shared UI component owned here>                |
| Shared    | `SHR-<lowerCamel>`  | <shared non-UI abstraction owned here>          |

# Boundaries & dependencies

<!--
GUIDANCE: Make the module seams explicit. Two directions:
  - DEPENDS ON: the other MOD-* ids this module consumes (MUST equal the
    `depends_on:` front-matter, and the union of the cross-module `depends_on`
    of its contained entries — no hidden coupling). Say WHAT it uses from each.
  - EXPOSES TO: what this module offers outward, and to which modules (the
    public surface other modules may reference by id). Anything not listed here
    is internal and MUST NOT be referenced from outside.
Dependencies MUST be acyclic at the module level (see conventions §12).
-->

**Depends on**

| Module        | What this module uses from it                         |
|---------------|-------------------------------------------------------|
| `MOD-<other>` | <the specific entries/contracts consumed, by id>      |

**Exposes to**

| Consumer module | Public surface offered (entry ids)                  |
|-----------------|-----------------------------------------------------|
| `MOD-<other>`   | <the entry ids other modules may reference>         |

# Conventions specific to the module

<!--
GUIDANCE: ONLY rules that are special to THIS module and not already in
.claude/sdd/conventions.md / .sdd/target.md (DRY — never restate the canonical
contracts). Examples: a naming pattern local to this module, an ordering rule,
an isolation rule, a derivation rule. If the module has no special rules, write
"None beyond .claude/sdd/conventions.md." Do NOT leave this empty.
-->

- <module-local rule 1, or "None beyond .claude/sdd/conventions.md.">

---

## Filled example

> A complete, valid instance of the template above for the **mandatory**
> `MOD-build` infra module (conventions §2: present in every project). It owns
> build files, dependency manifests, config, CI, framework/e2e config, and **DB schema
> changes DERIVED from the entity specs** — schema changes are never hand-authored
> independently. `MOD-build` is the **documented exception** to the module-overview
> `source: []` rule: it owns its infra FILES **directly** in its own `source:` (each
> forward schema script is its own append-only file traced to an `ENT-*`; the
> code-implementer materializes them from the entity field tables). The DB row below
> is the backend variant; a **GUI project** additionally lists `playwright.config.ts`
> (the e2e harness) in `source:` and sets `.sdd/target.md`'s `test-e2e` to a real command.

```markdown
---
id: MOD-build
name: Build & Infrastructure
kind: module
module: MOD-build
status: draft
depends_on: [MOD-model, ENT-user]
requirements: []
source: [package.json, tsconfig.json, .github/workflows/ci.yml, db/schema/V1__create_user.sql]
owns_sections: []
---

# Purpose

MOD-build is the mandatory infrastructure module. It owns everything required to
compile, configure, test, ship, and evolve the database schema but that is not itself
domain behavior: the build files, the dependency manifests, runtime configuration,
the CI pipeline, the framework/e2e harness config, and the database schema-change
scripts. Schema changes are **derived from the entity specs in MOD-model** — they are
never hand-authored independently, so a schema change always originates from an `ENT-*`
spec, never from invention here. MOD-build owns these infra **files directly** in its
own `source:` (the documented exception to the module-overview `source: []` rule);
domain logic, use-cases, and UI never live here — they live in their feature, class,
and component modules and are merely built and shipped by MOD-build.

# Contained entries

<!-- MOD-build owns its infra FILES directly via `source:` (exceptional for a module),
     not via separate domain sub-entries; it contains no FEAT/CLS/ENT/COMP/SHR of its own. -->

| Level | Entry id | What it represents (one line) |
|-------|----------|-------------------------------|
| —     | —        | none — MOD-build owns its infra files directly via `source:` (conventions §2 exception) |

# Boundaries & dependencies

**Depends on**

| Module      | What this module uses from it                                       |
|-------------|---------------------------------------------------------------------|
| `MOD-model` | The `ENT-*` entity specs (fields, types, relations, constraints) the forward schema scripts in `source:` derive from |

**Exposes to**

| Consumer module | Public surface offered                                          |
|-----------------|-----------------------------------------------------------------|
| `MOD-api`       | the built/runnable artifacts + applied DB schema the API runs on |
| `MOD-web`       | the build/bundle entry points and (GUI) the e2e harness          |
| _all modules_   | the one CI pipeline that installs, lints, builds, tests, packages |

# Conventions specific to the module

- **Schema changes are derived, never authored.** Each forward schema script in
  `source:` (e.g. `db/schema/V1__create_user.sql`) cites the `ENT-*` spec id and
  field/relation it realizes; a schema change starts by editing the entity spec
  (Markdown is the source of truth), then this module emits the **next** forward
  script. Shipped scripts are **append-only and immutable** — a change adds a new
  `Vn` script to `source:`, never edits a prior one.
- **Config keys are declared once** (typed config: key, type, default, source). No
  module reads an undeclared environment variable; new keys are added here first.
- **CI stage order is fixed:** install → lint → build → test (incl. e2e for a GUI
  project) → package. A module extends the one CI pipeline, never adds a parallel script.
```
