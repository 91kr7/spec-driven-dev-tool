# SDD Conventions — single canonical reference

> **Canonical contract.** Every agent reads this first. On any conflict this file
> wins — except its two siblings it defers to: `scot.md` (behavioral grammar) and
> `ui-schema.md` (UI form).

**Two cross-cutting values bind every agent:**
1. **Markdown is the source of truth (authority).** Specs decide WHAT the system does. On any conflict the **spec wins and code is corrected**, never the reverse. Code/tests are derived.
2. **Reuse over repetition (DRY).** Discover-before-create. Duplication above a small threshold is blocking.

---

## 1. File & folder layout

```
# THE TOOL — .claude/ (immutable; a project never edits it)
.claude/agents/*.md          # the 10 subagents
.claude/commands/sdd-auto.md # the single orchestrator command
.claude/sdd/conventions.md   # THIS FILE
.claude/sdd/scot.md          # behavioral grammar (behavioral specs)
.claude/sdd/ui-schema.md     # UI convention (gui specs) + baseline UI library (§9)
.claude/sdd/templates/*.md   # forms new specs are copied from

# THE PROJECT — written at runtime
requirements/REQUIREMENT.md  # raw + refined requirement (REQ-* ids)
plan/PLAN.md                 # plan output
.sdd/target.md               # stack + canonical build/test/run commands
.sdd/state.md                # append-only gate verdict / audit log
.sdd/impl-notes/<id>.md      # implementer concretization notes (NOT part of the spec)
specs/indexes/{modules,features,model,classes,ui-components}.index.md  # one index per level
specs/modules/<id>.spec.md   # incl. MOD-build
specs/features/<id>.spec.md  # use-case: orchestration + integration acceptance
specs/model/<id>.spec.md     # entities: fields, relations, constraints
specs/classes/<id>.spec.md   # per-method SCoT (+ feature gui screens)
specs/ui-components/<id>.spec.md  # UI library (baseline guaranteed — ui-schema §9)
specs/shared/<id>.spec.md    # shared non-UI abstractions — indexed in classes.index.md
src/                         # GENERATED
tests/                       # GENERATED (unit←classes, integration←features, constraint←entities, component←gui, e2e←gui screens)
```

**Scaling option (decided in the plan):** for a large multi-module project the fine-grained indexes (`classes`/`model`/`ui-components`) MAY split per module under `specs/indexes/<level>/<MOD-id>.index.md`; `modules`/`features` stay global. Default = single global index per level.

---

## 2. Identifier scheme

| Prefix | Level / kind | Form | Example |
|---|---|---|---|
| `MOD-` | module | `MOD-<kebab>` | `MOD-api`, `MOD-build` |
| `FEAT-` | feature / use-case | `FEAT-<nnn>` | `FEAT-001` |
| `ENT-` | domain entity | `ENT-<kebab>` | `ENT-user` |
| `CLS-` | class / service / screen | `CLS-<lowerCamel>` | `CLS-userRepo`, `CLS-regCtrl` |
| `COMP-` | shared UI component | `COMP-<lowerCamel>` | `COMP-button` |
| `SHR-` | shared non-UI abstraction | `SHR-<lowerCamel>` | `SHR-passwordHasher` |
| `AC` | acceptance criterion | `AC<n>` | `AC1` |
| `B` | SCoT branch | `B<n>` + arm | `B1.then`, `B3.empty` |

- Ids are **stable**: never renumber/rename; new entries take the next free id; deprecate rather than rename.
- A spec file is named after its id (`specs/classes/CLS-userRepo.spec.md`).
- **`MOD-build` is mandatory in every project.** It owns build files, manifests, config, CI, framework/build/entry scaffolding (e.g. `tsconfig.json`, the app entry, `playwright.config.*` for a GUI project's e2e), and **DB schema changes derived from the entity specs** (never hand-authored; forward-only, append-only once shipped).
  - DB project (`target.md` DB ≠ `none`) + any persisted `ENT-*` → `MOD-build` MUST declare ≥1 forward schema-change script in `source:`.
  - GUI project (Frontend ≠ `none`) → `MOD-build` owns the e2e harness (`playwright.config.*` + `webServer`) and `target.md`'s `test-e2e` must be a real command, not `n/a`.

---

## 3. Spec front-matter (YAML)

```yaml
---
id: CLS-regCtrl                 # required — matches filename + an index row
name: RegistrationController    # required
kind: controller                # required — drives the FORM (table below)
module: MOD-api                 # required for class/model/feature/gui
layer: organism                 # ui-components only: atom|molecule|organism|layout
status: draft                   # draft|reviewed|approved — MIRRORS the index (index is canonical)
depends_on: [CLS-userRepo, ENT-user]   # ids — topological order
requirements: [REQ-001]         # back-link id(s)
source: [src/api/RegistrationController.ts]   # AUTHORITATIVE spec→source mapping
owns_sections: []               # co-owned aggregator files: the section(s) this spec owns
variants: [primary, secondary]  # ui-components only (optional)
error_style: result             # behavioral specs only: result|raise (canonical home; see scot.md §6)
---
```

- `source:` is the **single authoritative** spec→source map. The index `source` column is **derived** from it by the authoring agent — never hand-edited later. NEW entity → propose paths from `target.md`; EXISTING → real files; `[]` for a purely-compositional feature.
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

**GUI-project trigger:** `target.md` Frontend ≠ `none` (equivalently: any `gui`-kind entry exists). Only a GUI project materializes the baseline UI library (ui-schema §9).

A **stub/mock is never specced** — it is auto-derived from its `interface` spec.

### Required sections (per kind)
- Default: `# Purpose` · `# Public interface` (inputs/outputs/errors) · `# Invariants & rules` · the **body form** · `# Acceptance criteria` (each `ACn` Given/When/Then).
- `module`: `# Purpose` · `# Contained entries` · `# Boundaries & dependencies` (no Public-interface/Invariants).
- `COMP-*` (`kind: gui`): replaces Public-interface + Invariants with the ui-schema §6 sections (Props · Variants · Visual states · Events · Slots/children · Accessibility).

---

## 4. Index row schema

```
| id | name | description (WHAT, one line) | module | depends_on | spec | source | status |
```
- `description` = what it represents, never how it is built.
- `depends_on` = comma-separated ids **without brackets** (`—` when none).
- `spec` / `source` = `—` when no file (`source: []`).
- `source` is **derived** from the spec's `source:` by the authoring agent.
- `status` is the **canonical** lifecycle home (§5).
- `ui-components.index.md` adds `layer` + `variants` columns.
- `SHR-*` are listed in `classes.index.md` (no separate shared index).
- The authoring agent fills **every** column for each row it writes.

Agents **read the index first**, then open only the specs they need (lazy loading).

---

## 5. Status lifecycle & separation of duties

Per-entity status lives in the **index row**: `draft → reviewed → approved`.
- `draft` — authored, not yet through the analysis gate.
- `reviewed` — passed the analysis gate.
- `approved` — implemented + code gate PASS + tests green + test gate PASS.

**Backward transition (spec change after `reviewed`).** Code is only generated from a `reviewed` spec. So whenever a spec changes after `reviewed`/`approved` — a gate routing a **spec bug**, a **feature evolution**, or a **reuse-analyst promotion** that rewrites a gated spec — the **command demotes** it `→ draft`, `spec-writer` fixes it, then it re-flows the forward path. A **code** or **test** bug never demotes the spec.

**Separation of duties (strict):**
- **Gatekeepers JUDGE only** — write a verdict to `.sdd/state.md`; never edit specs/code/tests/`status`.
- **The command (main session) ADVANCES `status`** from the latest verdict.
- **Authors WRITE artifacts** (specs/code/tests/impl-notes); never write verdicts.

---

## 6. `.sdd/state.md` verdict record

Append-only. Every gate appends one record:

```
## <ISO-8601> — <gate-agent> — <PASS|REJECT>
- scope: <ids reviewed, e.g. FEAT-001, CLS-regCtrl>
- phase: <analysis|code|test>
- iteration: <n>/<budget>
- verdict: PASS | REJECT
- reasons:
  - <blocking reason, citing the spec/AC/branch/requirement id>
- routing: <none | plan-architect | spec-writer | reuse-analyst | code-implementer | test-writer | escalate>   # REJECT only
```

- `routing: escalate` = a REJECT no author can fix (missing dependency, unresolved `<…>` placeholder, e2e-setup app-won't-boot) — the command surfaces it to the human.
- **Appending safely:** gatekeepers hold `Write` (whole-file), not `Edit` — they MUST **Read** `state.md`, then **Write it back in full** with the new record appended. Never write only the new record.

---

## 7. Iteration budgets & failure routing

| Loop | Budget | On overflow |
|---|---|---|
| analysis (plan + spec) | 3 | escalate |
| code | 3 | escalate |
| test | 5 | escalate |

- Budgets are **per scope** (the `PLAN` scope and each slice scope count independently).
- A **nested re-gate** (e.g. a spec or code fix triggered inside the test loop) counts against the **loop currently driving it** (the test loop), and the gatekeeper stamps **that** loop's budget in `iteration:`.

**test-gatekeeper triage routing** (MD is authority — a red test never patches code arbitrarily):
- **spec bug** → `spec-writer` (fix spec, regenerate code).
- **code bug** → `code-implementer` (minimal diff to match the spec).
- **test bug** → `test-writer` (fix the test to assert a spec AC/branch).
- **build/setup failure** (suite never ran) → route by offending file: `src/**` → `code-implementer`, `tests/**` → `test-writer`; missing dep/tooling → **escalate**. Judge run-health *before* coverage.
- **e2e-setup failure** (app won't boot / browsers missing) → **escalate** (unless pinned to a `src/**` crash → `code-implementer`, or a `tests/**` Playwright error → `test-writer`).
- **e2e phase did not run** (in-scope `gui` spec but `e2e` absent from REPORT `suites`) → **escalate**; a green unit/component run is NOT a PASS for a GUI scope.

---

## 8. Change policy — edit by default, regenerate by exception

- **Edit (default)** for bug-fix AND feature evolution: read spec + `impl-notes` + source, apply a **minimal diff** to the mapped file(s). Never rewrite a whole file for a small change.
- **Spec stays authority:** the edit brings code to the spec. If a failure reveals the spec is wrong, fix the **spec first**.
- **Back-propagate to MD:** any decision SCoT omits (library, API binding, idiom, edge case, bug-fix lesson) goes into `impl-notes`, never the spec.
- **Regenerate (exception):** whole-file rewrite only when the file is new, the spec changed substantially, or it drifted badly — one file at a time.
- **Equivalence = the tests**, not a textual diff.

---

## 9. Agent roster & isolation matrix

Eleven roles; **ten are subagents** in `.claude/agents/`. The **orchestrator is NOT a subagent** — it is the main session running `sdd-auto`. Subagents are single-purpose: read files in, write files/verdicts out; they never spawn subagents.

| Agent | Role | May WRITE | tools | Reads `src/`? | model |
|---|---|---|---|---|---|
| `plan-architect` | requirement → plan + target | `plan/`, `.sdd/target.md` | `Read, Write, Edit, Glob, Grep` | no | opus |
| `plan-gatekeeper` | judge the plan | `.sdd/state.md` | `Read, Write, Glob, Grep` | no | opus |
| `spec-writer` | write indexes + specs | `specs/` | `Read, Write, Edit, Glob, Grep` | no | opus |
| `reuse-analyst` | dedupe + promote shared specs | `specs/` | `Read, Write, Edit, Glob, Grep` | no | opus |
| `analysis-gatekeeper` | judge specs (only spec-phase blocker) | `.sdd/state.md` | `Read, Write, Glob, Grep` | no | opus |
| `code-implementer` | specs → source | `src/` (declared paths), `.sdd/impl-notes/` | `Read, Write, Edit, Glob, Grep` | yes (edit) | opus |
| `code-gatekeeper` | judge code ≡ spec | `.sdd/state.md` | `Read, Write, Glob, Grep, Bash` (read-only) | yes (review) | opus |
| `test-writer` | specs → tests (independent oracle) | `tests/` | `Read, Write, Edit, Glob` | **no — by role** | sonnet |
| `test-runner` | run tests, write report | `tests/REPORT.md` | `Read, Write, Glob, Bash` | yes | sonnet |
| `test-gatekeeper` | verify coverage + triage | `.sdd/state.md` | `Read, Write, Glob, Grep` | yes | opus |

- The `.claude/sdd/` contracts + templates ship with the tool — **read-only**, no agent edits them.
- A gatekeeper's `Write` is scoped to **appending its verdict to `.sdd/state.md`** only.
- **test-writer independence is SOFT**: no `Bash`, explicit NON-GOAL (never read `src/` or `impl-notes/`), and the test-gatekeeper rejects tests that assert implementation detail.
- The **test-runner** is the only agent that **executes** the suite (canonical `target.md` commands, filling only `{scope}`); for GUI e2e those commands launch/tear down the running app.
- **Models:** Opus for under-specified authoring + high-consequence judgment; Sonnet for mechanical/checklist work a concrete contract constrains. A project MAY override any `model:`.

---

## 10. Command roster

One command. The main session runs it, drives every loop (invoke agent → read verdict → decide → advance `status`).

| Command | Drives |
|---|---|
| `/sdd-auto` | The whole flow end-to-end, **one vertical slice at a time** in `depends_on` order, human OUT of the loop. Plan → Specify → Implement → Test, automatic gates; escalate only on budget overflow (or an unstated stack / `escalate` routing). |

---

## 11. ROLE header (mandatory in every agent file)

Every `.claude/agents/*.md` body begins with:

```
ROLE: <one-line identity>
MISSION: <the single outcome this agent owns>
MINDSET: <values — MUST include both cross-cutting values>
NON-GOALS: <what it must never do>
```

The MINDSET MUST carry both: **"Markdown is the source of truth (authority); reuse over repetition (DRY)."**

---

## 12. Topological processing & vertical slices

- Process entities in **`depends_on` topological order** (dependencies first); the graph MUST be acyclic.
- On a **cycle**, introduce an `interface`/`contract` spec, re-point members' `depends_on` onto it (stubs derive from the interface), so the physical graph is a DAG.
- **One vertical slice at a time:** take a feature, pull its `depends_on` closure, run plan→spec→code→test for that slice before the next.

---

## 13. Traceability (reconstructable, not a file)

Chain **REQUIREMENT → FEATURE → CLASS → SOURCE → TEST** is rebuilt on demand from: indexes + each spec's `requirements:`/`source:` + source-file traceability headers + test coverage ids.

Every generated source file carries a header pointing back to its spec:
```
// spec: CLS-regCtrl RegistrationController — specs/classes/CLS-regCtrl.spec.md
```
- Comment syntax per target language; placed at the **top but after any mandatory first-line construct** (shebang, `<?php`, `"use client"`, XML/encoding decl).
- **Comment-less formats are exempt** (pure JSON like `package.json`/`tsconfig.json`, lockfiles): the spec↔source link is the `source:` declaration alone — the gatekeeper never demands a header for them.

---

## 14. `tests/REPORT.md` format (test-runner → test-gatekeeper contract)

`test-runner` writes it (overwritten each run); `test-gatekeeper` parses it. Fixed structure (no heuristics). **Coverage** (every `ACn`/arm has a test) is verified by the gatekeeper from the tagged test files; this report supplies the **run result**.

```
# Test Report

## Run
- timestamp: <ISO-8601>
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
