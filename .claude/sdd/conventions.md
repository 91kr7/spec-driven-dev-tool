# SDD Conventions — single canonical reference

> **Canonical contract.** Every agent reads this first. On conflict this file
> wins — except its two siblings it defers to: `scot.md` (behavioral grammar) and
> `ui-schema.md` (UI form).

**Two cross-cutting values bind every agent:**
1. **Markdown is the source of truth (authority).** Specs decide WHAT the system does; code/tests are derived. On conflict the **spec wins and code is corrected**, never the reverse.
2. **Reuse over repetition (DRY).** Discover-before-create. Duplication above a small threshold is blocking.

---

## 1. File and folder layout

```
# THE TOOL — .claude/ (immutable; a project never edits it)
.claude/agents/*.md          # the 11 subagents
.claude/commands/sdd-auto.md # the single orchestrator command
.claude/sdd/conventions.md   # THIS FILE
.claude/sdd/scot.md          # behavioral grammar (behavioral specs)
.claude/sdd/ui-schema.md     # UI convention (gui specs) + reusable component catalog (§9)
.claude/sdd/templates/*.md   # forms new specs are copied from

# THE PROJECT — SDD process metadata: ALL under .sdd/ (only SDD agents read/write it; no compiler / bundler / test-runner / human toolchain looks here)
.sdd/target.md               # stack + canonical build/test/run commands (the env contract)
.sdd/REQUIREMENT.md          # raw + refined requirement (REQ-* ids); refined list as a dated changelog
.sdd/PLAN.md                 # plan output (a delta on existing-SDD)
.sdd/specs/modules.index.md       # GLOBAL skeleton: every MOD-* + depends_on + status (the only top-level index)
.sdd/specs/<MOD-id>/<MOD-id>.spec.md    # each module's own spec, at the root of its folder (incl. MOD-build, and MOD-schema for a DB project)
.sdd/specs/<MOD-id>/<MOD-id>.index.md   # per-module roster: every entity in this module, ALL levels (lazy — once it has ≥1 member)
.sdd/specs/<MOD-id>/<level>/<id>.spec.md  # members; <level> ∈ {features,classes,model,ui-components,shared}; each subfolder created lazily
.sdd/specs/MOD-shared/{shared,ui-components}/<id>.spec.md  # the LIBRARY: domain-agnostic primitives ONLY — generic COMP-* (design-system kit) + SHR-* (utils/types); a dependency SINK (depends_on MOD-build only). Admission by NATURE, not consumer count (§13)
.sdd/specs/REUSE-REPORT.md        # reuse-analyst output: promotions + Demote-for-re-gate list
.sdd/impl-notes/<MOD-id>/<level>/<id>.impl-notes.md  # concretization notes — EXACT mirror of .sdd/specs/ (same relative path, .spec.md → .impl-notes.md; a module's OWN note sits at <MOD-id>/<MOD-id>.impl-notes.md); NO index files; NOT the gated spec; the test-writer never reads this tree
.sdd/verdicts/<scope>/                  # per-scope folder (scope = slice_id | PLAN | PROJECT) — the ONLY new structure; holds that scope's verdicts + run report:
.sdd/verdicts/<scope>/<phase>.md        # the verdict for that phase (analysis|code|test) — { verdict: PASS|REJECT, reasons[], routing }; OVERWRITTEN each gate; read by the command by known path (§6)
.sdd/verdicts/<scope>/_test-report.md   # test-runner → test-gatekeeper run result (§14; overwritten each run)

# THE PROJECT — product: stays at the root the toolchain expects, NEVER under .sdd/ (build tools, import paths, test discovery, package/docker manifests all assume these roots)
src/                         # GENERATED source (or framework roots, e.g. backend/ frontend/)
tests/                       # GENERATED test files (unit←classes, integration←features, constraint←entities, component←gui, e2e←gui screens)
```

**Organized by module, level inside.** Every entity lives under its home module: `.sdd/specs/<module:>/<level>/<id>.spec.md`.
- The **`module:` front-matter is the folder**.
- The **id prefix is the level**: `CLS-`→`classes`, `FEAT-`→`features`, `ENT-`→`model`, `COMP-`→`ui-components`, `SHR-`→`shared`. A `MOD-` spec sits at its folder root.
- Level subfolders and the per-module `<MOD>.index.md` are created **lazily** (only when populated).
- The single global `modules.index.md` is the architectural skeleton.
- A single-module project still uses `specs/<MOD>/…` (no flattening).
- **Library placement — by nature, not count:** domain-agnostic primitives (generic `COMP-*`/`SHR-*`) live in **`MOD-shared`**; a **domain** node stays in its own module (reused by a `depends_on` **edge**). The full rule — sink · primitives-only · first-use · re-home — is [§13](#13-traceability).

---

## 2. Identifier scheme

| Prefix | Level / kind | Form | Example |
|---|---|---|---|
| `REQ-` | atomic requirement | `REQ-<nnn>` | `REQ-001` |
| `MOD-` | module | `MOD-<kebab>` | `MOD-api`, `MOD-build` |
| `FEAT-` | feature / use-case | `FEAT-<nnn>` | `FEAT-001` |
| `ENT-` | domain entity | `ENT-<kebab>` | `ENT-user` |
| `CLS-` | class / service / screen | `CLS-<lowerCamel>` | `CLS-userRepo`, `CLS-regCtrl` |
| `COMP-` | shared UI component | `COMP-<lowerCamel>` | `COMP-button` |
| `SHR-` | shared non-UI abstraction | `SHR-<lowerCamel>` | `SHR-passwordHasher` |
| `AC` | acceptance criterion | `AC<n>` | `AC1` |
| `B` | SCoT branch | `B<n>` + arm | `B1.then`, `B3.empty` |

**Terminology — "entity" (generic) vs `ENT-` (specific).**
- Unqualified, **entity** = *any planned node at any level* — one index row / one spec, whatever its prefix (`MOD-`/`FEAT-`/`ENT-`/`CLS-`/`COMP-`/`SHR-`); the generic word for "a thing in the plan/index" (e.g. "one row per entity", "every `REQ-*` covered by ≥1 entity", "process entities in `depends_on` order").
- The prefix **`ENT-`** is the *narrow* sense — a **domain entity** (the `entity` kind: field table + relations + invariants).
- For the narrow sense the text says `ENT-` or "`entity` **kind/spec**". Bare "entity" is always the generic node.

- Ids are **stable**: never renumber/rename; new entries take the next free id; deprecate rather than rename.
- A spec file is named after its id and lives under its home module (`.sdd/specs/MOD-domain/classes/CLS-userRepo.spec.md`).
- **`MOD-build` is mandatory in every project.** It owns build files, manifests, config, CI, and framework/build/entry scaffolding (e.g. `tsconfig.json`, the app entry, `playwright.config.*` for a GUI project's e2e). It carries **no `depends_on`** itself, and **every other module declares `depends_on: MOD-build`**, so it is always the **first slice** — generated code cannot compile/build without the scaffolding.
  - GUI project (Frontend ≠ `none`) → `MOD-build` owns the e2e harness (`playwright.config.*` + `webServer`) and `target.md`'s `test-e2e` must be a real command, not `n/a`.
- **`MOD-schema` is mandatory for a DB project** (`target.md` DB ≠ `none`) with any persisted `ENT-*`.
  - It owns the **DB schema changes derived from the entity specs** (never hand-authored; forward-only, append-only once shipped) and MUST declare ≥1 forward schema-change script in `source:`.
  - Its `depends_on` reaches the persistence module + the `ENT-*` it evolves, so it is ordered **after** the entities.
  - Unlike `MOD-build`, it is **not requirement-exempt**: its `requirements` = the union of the `REQ-*` of the `ENT-*` whose schema it materializes (the DB exists for those persistence requirements).
  - A non-DB project has no `MOD-schema`.
- **`MOD-shared` is the LIBRARY** — the reserved-id module owning every **domain-agnostic primitive** (the generic `COMP-*` design-system kit + `SHR-*` utils/types). Its **placement rule** — by nature not count · dependency sink · admits only primitives · re-home — is [§13](#13-traceability); here, only its id-scheme facts:
  - the id `MOD-shared` is **reserved** (like `MOD-build`/`MOD-schema`);
  - members are **materialized at first use** — each rides in the `depends_on` closure of the first slice that composes it, so the module is never empty/orphaned;
  - `requirements` = the union of its members' `REQ-*`.

---

## 3. Spec front-matter

```yaml
---
id: CLS-regCtrl                 # required — matches filename + an index row
name: RegistrationController    # required
kind: controller                # required — drives the FORM (table below)
module: MOD-api                 # the home module = the spec's FOLDER (§1); required for every entity (a MOD- spec sits at its folder root)
layer: organism                 # ui-components only: atom|molecule|organism|layout
depends_on: [CLS-userRepo, ENT-user]   # ids — topological order
requirements: [REQ-001]         # back-link id(s)
source: [src/api/RegistrationController.ts]   # AUTHORITATIVE spec→source mapping
owns_sections: []               # co-owned aggregator files: the section(s) this spec owns
variants: [primary, secondary]  # ui-components only (optional)
error_style: result             # behavioral specs only: result|raise (canonical home; see scot.md §6)
---
```

- **`status` is NOT a front-matter field — it lives ONLY in the index row** ([§5](#5-status-lifecycle-and-separation-of-duties), the canonical home): `draft → reviewed → implemented → approved`, advanced by the orchestrator. Read an entity's state from its index row, never from the spec.
- `source:` is the **single authoritative** spec→source map. The index `source` column is **derived** from it by the authoring agent — never hand-edited later. Paths: NEW entity → propose from `target.md`; EXISTING → real files; `[]` for a purely-compositional feature.
- `requirements:` = back-link to the `REQ-*` id(s) the spec realizes — **real ids or, ONLY for `MOD-build`, `—`; never a prose annotation**. Every other spec needs ≥1 real `REQ-*`.
  - `MOD-build` is the **sole** exemption: scaffolding for the *whole* app, tied to no single requirement (test: removing any one `REQ-*` never removes it).
  - Two kinds borrow their requirements from a related set — `MOD-schema` = the **union** of its `ENT-*`' `REQ-*`; a **shared/library** spec (`SHR-*`, baseline-or-promoted `COMP-*`) = a **non-empty subset** of its consumers' `REQ-*`. The full rule — the subset test, **orphan** (empty `requirements:`), and **excess** (a listed `REQ-*` no consumer carries) — lives in [§13](#13-traceability). "No requirement", or an unbacked one, always signals a real defect.
- **One file ↔ one spec by default.** A shared aggregator MAY be co-owned only if every co-owner declares it in `source:` and names `owns_sections:`. Undeclared shared ownership is blocking.
- `error_style:` lives **only** in front-matter (canonical). scot.md restates the style atop a body for readability but the front-matter field is authority.

### `kind:` → body form

| `kind:` | Category | Body form |
|---|---|---|
| `service` / `controller` / `use-case` | behavioral | **SCoT** (`scot.md`) |
| `entity` | structural | declarative — field table + relations + invariants |
| `dto` / `enum` / `config` | structural | declarative table |
| `interface` | structural | declarative — **signatures only, no body** |
| `module` | structural | overview — Purpose · Contained entries · Boundaries |
| `gui` | ui | **UI schematic** (`ui-schema.md`) |

**GUI-project trigger:** `target.md` Frontend ≠ `none` (equivalently: any `gui`-kind entry exists). Only a GUI project creates `COMP-*` — from the ui-schema [§9](ui-schema.md#9-reusable-component-catalog) catalog, and only those its screens actually compose (the catalog is candidates, not a required set).

A **stub/mock is never specced** — auto-derived from its `interface` spec.

### Required sections (per kind)
- Default: `# Purpose` · `# Public interface` (inputs/outputs/errors) · `# Invariants & rules` · the **body form** · `# Acceptance criteria` (each `ACn` Given/When/Then).
- `module`: `# Purpose` · `# Contained entries` · `# Boundaries & dependencies` (no Public-interface/Invariants).
- `COMP-*` (`kind: gui`): replaces Public-interface + Invariants with the ui-schema [§6](ui-schema.md#6-extra-sections-for-components) sections (Props · Variants · Visual states · Events · Slots/children · Accessibility).

### Acceptance-criterion altitudes (what verifies each AC)
Every `ACn` is verified at exactly one of three altitudes, **marked** so coverage is mechanical:
- **test-covered** (default, untagged) — an authored test asserts it (unit / integration / component). The `test-writer` writes ≥1 mapped test (scot.md [§7.3](scot.md#7-branch-id-rules) id); the `test-gatekeeper` REJECTs if any is uncovered.
- **`(journey)`** — a screen outcome crossing the running stack; verified **end-to-end by a Playwright test** (ui-schema [§5](ui-schema.md#5-events-table-and-journey-acs)).
- **`(pipeline)`** — the outcome **is** the success of a canonical `target.md` §3 command (install / build / run-boot / migrate); verified by that command reaching green in `.sdd/verdicts/<scope>/_test-report.md`, **not** by an authored test (a test re-asserting "the build passes" is circular; one re-asserting a manifest value against the spec is tautological — neither is an independent oracle).
  - **Allowed ONLY on an infra-module AC** (`MOD-build`, `MOD-schema`) whose assertion is literally "the build / boot / migration command succeeds".
  - A behavioral spec (`CLS-*` / `FEAT-*` / `ENT-*`) may **never** tag `(pipeline)` to dodge a real test, and a genuine boot **smoke** check (e.g. application-context-loads, which exercises runtime wiring the spec left open) stays **test-covered**, not `(pipeline)`.
  - The `test-writer` authors **no** test for a `(pipeline)` AC; the `test-gatekeeper` counts it covered from the green run result and REJECTs a `(pipeline)` tag on a non-infra spec.

---

## 4. Index rows

Indexes are **logical rosters**; agents **read them first**, then open only the specs they need (lazy loading). Exactly two kinds:

**Global `.sdd/specs/modules.index.md`** — one row per module (the skeleton):
```
| id | name | description (WHAT, one line) | depends_on | spec | source | status |
```

**Per-module `.sdd/specs/<MOD>/<MOD>.index.md`** — one row per entity living in that module, **all levels** (created lazily once the module has ≥1 member):
```
| id | name | description (WHAT, one line) | level | depends_on | spec | source | status |
```
- It **drops** `module` (redundant — it is the folder) and **adds** `level` (the entity's level = its id prefix: `feature`/`class`/`entity`/`ui-component`/`shared`).
- A module containing any `ui-component` row **appends** `layer` + `variants` after `level` (filled for the `COMP-*` rows, `—` elsewhere).

Common to both:
- `description` = what it represents, never how it is built.
- `depends_on` = comma-separated ids **without brackets** (`—` when none).
- `spec` / `source` = `—` when no file (`source: []`); `spec` is the full path `.sdd/specs/<MOD>/…`.
- `source` is **derived** from the spec's `source:` by the authoring agent.
- `status` is the **canonical** lifecycle home ([§5](#5-status-lifecycle-and-separation-of-duties)).
- A **primitive** `SHR-*`/`COMP-*` is rostered in `MOD-shared.index.md` (the library); a **domain** `SHR-*`/`COMP-*` (reused within a single module) in that module's `<MOD>.index.md` ([§1](#1-file-and-folder-layout), [§13](#13-traceability)).
- **Locate a spec by id** with `Glob .sdd/specs/**/<id>.spec.md` (you need not know its module up front; the id prefix gives the level). The matching index row gives its `status`/`depends_on`/`source`.
- The authoring agent fills **every** column for each row it writes.

---

## 5. Status lifecycle and separation of duties

Per-entity status lives in the **index row**: `draft → reviewed → implemented → approved` — one state per gate.
- `draft` — authored, not yet through the analysis gate.
- `reviewed` — passed the **analysis** gate (spec OK); code not yet written.
- `implemented` — passed the **code** gate (code matches spec, compiles); tests pending.
- `approved` — passed the **test** gate (suite green + full coverage).

**Backward transition (spec change after `reviewed`).** Code is only generated from a `reviewed` spec. So whenever a spec changes after `reviewed`/`implemented`/`approved` — a gate routing a **spec bug**, a **feature evolution**, or a **reuse-analyst promotion** that rewrites a gated spec — the sequence is: the **command demotes** it `→ draft`, `spec-writer` fixes it, then it re-flows the forward path. A **code** or **test** bug never demotes the spec **to draft** (the spec stays `reviewed`; the code-status demotion below is a separate transition).

**Status regresses to match where the fix re-enters (never below it).** A REJECT demotes the affected member(s) to the status whose forward phase the routed fix re-enters, so the resume invariant ([§7](#7-failure-routing) — `phase` = least-advanced own-member `status`) stays **exact** and an interrupted run never re-enters *past* the outstanding work:
- routed to **spec** (spec bug / feature-evo / promotion, from any gate) → **draft** (re-flows analysis → code → test);
- `test`-gate **code bug** → the flagged member(s) `implemented → reviewed` — the **spec is untouched**; only the code re-passes its gate before re-testing. This is the **sole** non-spec demotion, and the one case where `status` would otherwise outrun the real work: the test gate has cleared the code phase (`implemented`), then sends work *back* to it, so `status` must step back to `reviewed` or a resume would wrongly re-enter at the test phase;
- routed to **code** at the code gate (`status` already `reviewed`) or to **test** at the test gate (`status` `implemented`) → **no demotion**: the status already names the re-entry phase.

A step-9 regression on an `approved` slice follows the same rule — demote the flagged member(s) to the routed fix's phase (`draft`/`reviewed`/`implemented`) so step 8's remaining set re-includes the slice.

**Separation of duties (strict):**
- **Gatekeepers JUDGE only** — write one verdict **file** to `.sdd/verdicts/` ([§6](#6-verdict-records)); never edit specs/code/tests/`status`.
- **The command (main session) ADVANCES `status`** from the gate's verdict.
- **Authors WRITE artifacts** (requirements/specs/code/tests/impl-notes); never verdicts.

---

## 6. Verdict records

**One file per (scope, phase), OVERWRITTEN each gate.** Each gate writes a single file `.sdd/verdicts/<scope>/<phase>.md` holding only its latest outcome; in the loop the command reads it by **known path** (no scan, no ordinal). A re-gate of the same phase overwrites it (the prior reasons[] were already consumed by the routed author). **On resume** the command reads the slice's ≤3 known-named phase files and takes the one with the latest header timestamp ([§7](#7-failure-routing)) — still known paths: no directory scan, no counter, no accumulation.

- **Path:** `.sdd/verdicts/<scope>/<phase>.md`
  - `<scope>` = the short scope key (folder): the `slice_id` (e.g. `FEAT-login`, `MOD-build`), `PLAN`, or `PROJECT`. The full member-id list goes in the body's `scope:` field.
  - `<phase>` = `analysis` | `code` | `test` (the gate's phase; `PLAN`→`analysis`, the `PROJECT` sweep→`test`).
  - The sibling `_test-report.md` (underscore-prefixed) is the test-runner's run result ([§14](#14-test-report-format)), not a verdict.
- **One record per file:**

```
## <ISO-8601 timestamp> — <gate-agent> — <PASS|REJECT>   # = current_ts, a FRESH date+time the command obtains (it has the clock) and passes per gate; stamp it verbatim, never invent it. This timestamp ORDERS verdicts — resume reads the LATEST (§7).
- scope: <ids reviewed, e.g. FEAT-001, CLS-regCtrl>
- phase: <analysis|code|test>
- verdict: PASS | REJECT
- reasons:
  - <one terse line per check — see economy rule>
- routing: <none | requirement-analyst | plan-architect | spec-writer | reuse-analyst | code-implementer | test-writer | escalate>   # REJECT only
```

- **Verdict economy — record the *conclusion*, not the re-derivation.** Enumeration rigor lives in the gate's *reasoning* (what `effort: high` buys); the file records the *outcome* plus the smallest evidence the next reader acts on:
  - **PASS** → one terse line per check group: the conclusion + the minimal rebuilt datum where the check is a forcing-function (a traceability **consumer set**, an **orphan-scan** result). NEVER a paragraph re-narrating what held, never the whole front-matter/SCoT/AC list echoed back.
  - **REJECT** → one line per *blocking* defect: the offending `id`/path + what is wrong + the **resolution** (what the routed author consumes) + `routing:`. Do not also transcribe the checks that passed.
- `routing: escalate` = a REJECT no author can fix (missing dependency, unresolved `<…>` placeholder, e2e-setup app-won't-boot) — the command surfaces it to the human.
- **Why overwrite-per-phase:** the loop reads/writes one fixed file per (scope, phase) — O(1), no ordinal, no directory scan, no accumulation. (The earlier monolithic `status.md` forced Read-whole + Write-whole on every gate → O(N²); per-phase files plus dropping the iteration budget removed the need for any cross-file counting.) Resume orders verdicts by their **self-assigned header timestamp**, NOT a cross-file index — so "latest wins" needs no counter and no `NNN-` filenames (which would reintroduce the very scan + accumulation this avoids).

**Example — compact PASS** (full rebuilt evidence collapses to conclusions):
```
## 2026-06-26T14:03:20Z — analysis-gatekeeper — PASS
- scope: FEAT-login (MOD-auth, ENT-credential, SHR-passwordHasher, CLS-credentialRepo, CLS-loginRequest, CLS-loginResponse, CLS-authService, CLS-authController, FEAT-login)
- phase: analysis
- verdict: PASS
- reasons:
  - §3 front-matter: 9/9 valid (id↔filename↔index, error_style on the 5 behavioral specs).
  - §13 traceability: REQ-003..009 each reachable; consumers(SHR-passwordHasher)={CLS-authService}, requirements={004,005,006} ⊆ consumers', no orphan/excess.
  - §5 AC testability: every spec ≥1 AC (G/W/T); behavioral ACs map to SCoT arms.
  - §4/§6 consistency: every depends_on/CALL/collaborator resolves; index source/ids match front-matter.
  - §12 cycles: acyclic. §9 duplication: REUSE-REPORT promoted=none, ownership=clean (consumer set re-verified).
- routing: none
```
**Example — compact REJECT** (one actionable line per defect):
```
## 2026-06-26T15:41:07Z — code-gatekeeper — REJECT
- scope: MOD-build
- phase: code
- verdict: REJECT
- reasons:
  - §3 orphan (veto): backend/src/main/resources/application.properties is on disk but in no spec's source:; load-bearing boot scaffolding (SQLite datasource, AC3). Resolution: add it to MOD-build.spec.md source: + re-derive the modules.index.md source column.
- routing: spec-writer
```

---

## 7. Failure routing

**No iteration budget.** A gate loops `REJECT → fix → re-gate` until PASS. `/sdd-auto` is run **attended**: a gate↔author oscillation does NOT auto-stop — the human watching the run halts it by hand. The only automatic stop is `routing: escalate` (a block no author can fix). Resume needs no counter — the first non-approved slice's **LATEST verdict wins** (max header timestamp among its ≤3 phase files): a **REJECT** → re-invoke the author its `routing:` names, feeding its `reasons[]` (the live fix, wherever that file sits); a **PASS**/none → enter the phase the own-member `status` implies ([§5](#5-status-lifecycle-and-separation-of-duties)) FRESH, no reasons. The demote ([§5](#5-status-lifecycle-and-separation-of-duties)) keeps `status` truthful, so "skip approved" and the fresh-phase choice stay right; the latest verdict drives the pending fix.

**test-gatekeeper triage routing** (MD is authority — a red test never patches code arbitrarily):
- **spec bug** → `spec-writer` (fix spec, regenerate code).
- **code bug** → `code-implementer` (minimal diff to match the spec).
- **test bug** → `test-writer` (fix the test to assert a spec AC/branch).
- **build/setup failure** (suite never ran) → route by offending file: `src/**` → `code-implementer`, `tests/**` → `test-writer`; missing dep/tooling → **escalate**. Judge run-health *before* coverage.
- **e2e-setup failure** (app won't boot / browsers missing) → **escalate** (unless pinned to a `src/**` crash → `code-implementer`, or a `tests/**` Playwright error → `test-writer`).
- **e2e phase did not run** (in-scope `gui` spec but `e2e` absent from REPORT `suites`) → **escalate**; a green unit/component run is NOT a PASS for a GUI scope.

---

## 8. Change policy

- **Edit (default)** for bug-fix AND feature evolution: read spec + `impl-notes` + source, apply a **minimal diff** to the mapped file(s). Never rewrite a whole file for a small change.
- **Spec stays authority:** the edit brings code to the spec. If a failure reveals the spec is wrong, fix the **spec first**.
- **Back-propagate to MD:** any decision SCoT omits (library, API binding, idiom, edge case, bug-fix lesson) goes into `impl-notes`, never the spec.
- **Regenerate (exception):** whole-file rewrite only when the file is new, the spec changed substantially, or it drifted badly — one file at a time.
- **Equivalence = the tests**, not a textual diff.

---

## 9. Agent roster and isolation matrix

Twelve roles; **eleven are subagents** in `.claude/agents/`. The **orchestrator is NOT a subagent** — it is the main session running `sdd-auto`. Subagents are single-purpose: read files in, write files/verdicts out. They never spawn subagents.

| Agent | Role | May WRITE | tools | Reads `src/`? | model |
|---|---|---|---|---|---|
| `requirement-analyst` | capture raw → refined requirement + REQ ids | `.sdd/REQUIREMENT.md` | `Read, Write, Edit, Glob, Grep` | no | opus |
| `plan-architect` | requirement → plan (+ ordered slices) + target | `.sdd/PLAN.md`, `.sdd/target.md` | `Read, Write, Edit, Glob, Grep` | no | opus |
| `plan-gatekeeper` | judge the plan | `.sdd/verdicts/<scope>/<phase>.md` | `Read, Write, Glob, Grep` | no | opus |
| `spec-writer` | write indexes + specs | `.sdd/specs/` | `Read, Write, Edit, Glob, Grep` | no | opus |
| `reuse-analyst` | dedupe + promote shared specs | `.sdd/specs/` | `Read, Write, Edit, Glob, Grep` | no | opus |
| `analysis-gatekeeper` | judge specs (only spec-phase blocker) | `.sdd/verdicts/<scope>/<phase>.md` | `Read, Write, Glob, Grep` | no | opus |
| `code-implementer` | specs → source | `src/` (declared paths), `.sdd/impl-notes/` | `Read, Write, Edit, Glob, Grep` | yes (edit) | opus |
| `code-gatekeeper` | judge code ≡ spec | `.sdd/verdicts/<scope>/<phase>.md` | `Read, Write, Glob, Grep, Bash` (read-only) | yes (review) | opus |
| `test-writer` | specs → tests (independent oracle) | `tests/` | `Read, Write, Edit, Glob` | **no — by role** | sonnet |
| `test-runner` | run tests, write report | `.sdd/verdicts/<scope>/_test-report.md` | `Read, Write, Glob, Bash` | yes | sonnet |
| `test-gatekeeper` | verify coverage + triage | `.sdd/verdicts/<scope>/<phase>.md` | `Read, Write, Glob, Grep` | yes | opus |

- The `.claude/sdd/` contracts + templates ship with the tool — **read-only**, no agent edits them.
- A gatekeeper's `Write` is scoped to its single phase verdict `.sdd/verdicts/<scope>/<phase>.md` (overwritten each gate) only ([§6](#6-verdict-records)).
- **test-writer independence is SOFT**: no `Bash`, explicit NON-GOAL (never read `src/` or `.sdd/impl-notes/`), and the test-gatekeeper rejects tests asserting implementation detail.
- The **test-runner** is the only agent that **executes** the suite (canonical `target.md` commands, filling only `{scope}`); for GUI e2e those commands launch/tear down the running app.
- **Models:** Opus for under-specified authoring + high-consequence judgment; Sonnet for mechanical/checklist work a concrete contract constrains. A project MAY override any `model:`.

---

## 10. Command roster

One command. The main session runs it, driving every loop (invoke agent → read verdict → decide → advance `status`).

| Command | Drives |
|---|---|
| `/sdd-auto` | The whole flow end-to-end, **one vertical slice at a time** in `depends_on` order, human OUT of the loop. Plan → Specify → Implement → Test, automatic gates; escalate only on an unstated stack or an `escalate` routing. |

---

## 11. ROLE header

Every `.claude/agents/*.md` body begins with:

```
ROLE: <one-line identity>
MISSION: <the single outcome this agent owns>
MINDSET: <values — MUST include both cross-cutting values>
NON-GOALS: <what it must never do>
```

MINDSET MUST carry both: **"Markdown is the source of truth (authority); reuse over repetition (DRY)."**

---

## 12. Topological processing and vertical slices

- Process entities in **`depends_on` topological order** (dependencies first); the graph MUST be acyclic.
- On a **cycle**, introduce an `interface`/`contract` spec, re-point members' `depends_on` onto it (stubs derive from the interface), so the physical graph is a DAG.
- **One vertical slice at a time:** the plan + ordered slice list are produced once up front (plan-architect); then for each slice — a feature (or a module, e.g. `MOD-build`) plus its `depends_on` closure — run spec→code→test before the next.
- **Scaffolding first:** `MOD-build` has no `depends_on` and every domain module depends on it, so its slice is always **first** — no generated code can compile/build until the scaffolding exists. (`MOD-schema`, by contrast, depends on the `ENT-*` and is ordered after them.)

---

## 13. Traceability

Chain **REQUIREMENT → FEATURE → CLASS → SOURCE → TEST** is rebuilt on demand from: indexes + each spec's `requirements:`/`source:` + source-file traceability headers + test coverage ids.

**Shared/library nodes carry a subset of their consumers' requirements.** A `SHR-*`/`COMP-*` invents no `REQ-*`; it lists a **non-empty subset** of the `REQ-*` carried by the specs that `depends_on` it — each id both *consumer-backed* (some consumer carries it) **and** *genuinely realized by this node* (a leaf atom serves only some of its screen's requirements, not all) — so the chain `REQ → FEATURE/CLASS/screen → SHR/COMP` is explicit at every node, never inferred. A library primitive therefore always carries ≥1 `REQ-*`: those of its consumer(s) it actually realizes — one consumer is enough (admission is by nature, not a duplication count). Two failure modes a gate REJECTs:
- **empty** `requirements:` (no consumer, or nothing realized) → **orphan**.
- a listed `REQ-*` that **no consumer carries** → **excess** (gold-plating).

The invariant "every spec carries ≥1 real `REQ-*`" stays universal — only `MOD-build` (whole-app scaffolding) is outside the requirement graph; `MOD-schema` carries the **union** of the `REQ-*` of the `ENT-*` whose schema it materializes (it realizes them all).

**Placement of shared nodes — by NATURE, not count.** The signal for "library" is *what a node is*, never how many consume it:
- A **domain-agnostic primitive** — names/encodes no domain concept (a `Button` knows nothing of "order"; `formatMoney` nothing of "invoice") **and** depends on nothing domain (a pure sink) — lives in **`MOD-shared`** from its **first** use. One consumer is enough; count is irrelevant. This is how the design-system kit (panels, forms, header, footer, generic types/utils) gets built deliberately.
- A **domain node** — names a domain concept, or depends on a feature/domain module — **never** enters `MOD-shared`. Cross-module reuse of a domain capability is a **dependency edge**: the consumer `depends_on` it **in its home module** (the edge `B → A` is preserved, not dissolved). A domain *concept* several modules genuinely need (e.g. a `User`/`Account` entity) is **extracted into its own named domain module** — a first-class `depends_on` target, never dumped in the library.
- **`MOD-shared` admits only primitives.** It depends only on `MOD-build`; a gate REJECTs any `MOD-shared → feature-module` edge (mechanical half) **and** any member that encodes domain knowledge (nature half).
- A generic primitive discovered mis-homed in a feature module is **re-homed** to `MOD-shared` (file moves folder, id unchanged — id stability [§2](#2-identifier-scheme)); the trigger is the **nature discovery**, not a second consumer.

Every generated source file carries a header pointing back to its spec:
```
// spec: CLS-regCtrl RegistrationController — .sdd/specs/MOD-api/classes/CLS-regCtrl.spec.md
```
- Comment syntax per target language; placed at the **top but after any mandatory first-line construct** (shebang, `<?php`, `"use client"`, XML/encoding decl).
- **Comment-less formats are exempt** (pure JSON like `package.json`/`tsconfig.json`, lockfiles): the spec↔source link is the `source:` declaration alone — the gatekeeper never demands a header for them.

**Comment economy — the header is the *only* mandatory comment.** The **spec is the narrative** (Purpose, SCoT arms, invariants, ACs); the generated code is its concretization and **must not re-narrate it**. Forbidden as noise:
- a comment restating a `[Bn]` branch / AC / rule already in the spec;
- docstrings paraphrasing the Purpose;
- section-banner comments (`// ---- helpers ----`);
- "what" comments on self-evident statements.

The reader wanting *why* follows the header to the spec; a genuinely non-obvious concretization rationale goes to `impl-notes`, **not** an inline comment. Default to **comment-free bodies under the header** (plus only what the language *requires* — e.g. a mandated annotation). Over-commenting is duplication that costs tokens to write **and** re-read on every downstream pass, and silently drifts from the spec.

---

## 14. Test report format

`test-runner` writes one per scope at `.sdd/verdicts/<scope>/_test-report.md` (`<scope>` = the slice_id, or `PROJECT` for the step-9 sweep; overwritten each run); `test-gatekeeper` parses it. Fixed structure (no heuristics). **Coverage** (every test-covered `ACn`/arm has a test) is verified by the gatekeeper from the tagged test files; this report supplies the **run result**. A **`(pipeline)`** AC ([§3](#3-spec-front-matter) altitudes) is the exception: it carries no tagged test, so the gatekeeper counts it covered from this report's **green run result** (the install/build/boot/migrate command it asserts necessarily ran in reaching `phase-reached: complete`).

```
# Test Report

## Run
- timestamp: <ISO-8601>   # the run time; the test-runner has Bash → obtain it from the canonical date command, never invent it
- scope: <whole-suite (unscoped) | in-scope ids, e.g. FEAT-001, CLS-regCtrl>
- suites: <which ran, e.g. unit, integration, component, e2e>
- commands: install=<cmd> | build=<cmd> | unit=<cmd> | int=<cmd> | e2e=<cmd | n/a>
- exit-status: <0 | first non-zero phase code>
- phase-reached: <install | build | unit | integration | component | e2e-setup | e2e | complete>
- tooling: <none | note about a missing/placeholder command or the reporter pair used>

## Summary
- total: <n>
- passed: <n>
- failed: <n>
- skipped: <n>

## Failures
### <test name>
- coverage: <canonical id(s), scot.md §7.3 — e.g. CLS-regCtrl::register#B1.then and/or CLS-regCtrl#AC2 | unknown>
- message: <assertion or error message>
- excerpt: |
    <trimmed stack / output excerpt>
```

- `passed + failed + skipped = total`.
- A halted install/build/e2e-setup is reported via `phase-reached`, never hidden.
- A scoped run's verdict binds **only** that scope. `e2e` appears only for GUI projects.
- A failure whose `coverage` cannot be recovered is `unknown`, never invented.
