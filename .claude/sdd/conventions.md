# SDD Conventions — the single canonical reference

> **Status: canonical contract.** Every agent reads this first. It pins the
> shared vocabulary so hand-offs (which happen **only through files**) are
> unambiguous. If any other file disagrees with this one, this one wins — except
> for the two sibling contracts it defers to: `.claude/sdd/scot.md` (behavioral grammar)
> and `.claude/sdd/ui-schema.md` (UI form).

Cross-cutting values that bind **every** agent:

1. **Markdown is the source of truth (authority).** Specs/indexes decide WHAT the
   system does and how it is structured. In any conflict, the **spec wins and the
   code is corrected** — never the reverse. Code is a derived artifact.
2. **Reuse over repetition (DRY).** Prefer extracting a shared abstraction over
   duplicating logic or markup. Discover-before-create. Duplication above a small
   threshold is a blocking issue.

---

## 1. File & folder layout (canonical)

```
# THE TOOL — .claude/ (immutable; installed once; a project never edits it)
.claude/agents/*.md                  # the 10 subagents
.claude/commands/*.md                # the 7 slash commands
.claude/sdd/conventions.md           # THIS FILE — ids, front-matter, status, verdicts, rosters
.claude/sdd/scot.md                  # canonical SCoT grammar (behavioral specs)
.claude/sdd/ui-schema.md             # canonical UI-schematic convention (gui specs) + the GUARANTEED baseline UI library (§9)
.claude/sdd/templates/*.template.md  # the forms new specs are copied from (incl. target.template, impl-notes.template)

# THE PROJECT — created/written at runtime by the workflow (the project's own files)
requirements/REQUIREMENT.md          # raw + refined user requirement (with REQ-* ids)
plan/PLAN.md                         # output of the Plan mode
.sdd/target.md                       # stack/architecture + canonical build/test/run commands (AI-written, per project)
.sdd/state.md                        # append-only gatekeeper verdict / audit log
.sdd/impl-notes/<spec-id>.md         # implementer-owned concretization notes (NOT part of the gated spec)
specs/indexes/{modules,features,model,classes,ui-components}.index.md   # one index PER LEVEL
specs/modules/<id>.spec.md           # incl. MOD-build (build/config/CI/migrations)
specs/features/<id>.spec.md          # use-case: orchestration + integration acceptance
specs/model/<id>.spec.md             # domain entities: fields, types, relations, constraints
specs/classes/<id>.spec.md           # per-method SCoT (and feature-specific gui screens)
specs/ui-components/<id>.spec.md     # UI library (GUI projects: baseline GUARANTEED & materialized — ui-schema §9)
specs/shared/<id>.spec.md            # shared non-UI abstractions — indexed in classes.index.md
src/                                 # GENERATED (derived)
tests/                               # GENERATED (unit←classes, integration←features, constraints←entities)
```

**Scaling option (decided in the plan):** for a large multi-module project the
fine-grained indexes (`classes`, `model`, `ui-components`) MAY be split per module
under `specs/indexes/<level>/<MOD-id>.index.md`, keeping `modules.index` and
`features.index` global. Default is **single global index per level**.

---

## 2. Identifier scheme

| Prefix   | Level / kind                | Form              | Example                    |
|----------|-----------------------------|-------------------|----------------------------|
| `MOD-`   | module                      | `MOD-<kebab>`     | `MOD-api`, `MOD-build`     |
| `FEAT-`  | feature / use-case          | `FEAT-<nnn>`      | `FEAT-001`                 |
| `ENT-`   | domain entity (data-model)  | `ENT-<kebab>`     | `ENT-user`                 |
| `CLS-`   | class / service / screen    | `CLS-<lowerCamel>`| `CLS-userRepo`, `CLS-regCtrl` |
| `COMP-`  | shared UI component         | `COMP-<lowerCamel>`| `COMP-button`, `COMP-header` |
| `SHR-`   | shared non-UI abstraction   | `SHR-<lowerCamel>`| `SHR-passwordHasher`       |
| `AC`     | acceptance criterion (in-spec)| `AC<n>`         | `AC1`, `AC2`               |
| `B`      | SCoT branch (in-spec)       | `B<n>` + arm      | `B1.then`, `B3.empty`      |

- Ids are **stable**: never renumber/rename an existing id; new entries take the
  next free id. Renames are handled by adding a new id and deprecating the old.
- A spec file is named after its id: `specs/classes/CLS-userRepo.spec.md`.
- `MOD-build` is a **mandatory** infra module in every project: it owns build
  files, dependency manifests, config, CI, and **DB migrations derived from the
  entity specs** (migrations are never hand-authored independently).

---

## 3. Spec front-matter schema (YAML)

Every `*.spec.md` starts with YAML front-matter. Fields by applicability:

```yaml
---
id: CLS-regCtrl                 # required — matches the filename and an index row
name: RegistrationController    # required — human name
kind: controller                # required — drives the FORM (see §4)
module: MOD-api                 # required for class/model/feature/gui entries
layer: organism                 # ui-components only: atom|molecule|organism|layout
status: draft                   # draft|reviewed|approved — MIRRORS the index (index is canonical)
depends_on: [CLS-userRepo, SHR-passwordHasher, ENT-user]   # ids — topological order
requirements: [REQ-001]         # requirement id(s) this spec serves (back-link)
source: [src/api/RegistrationController.ts]   # AUTHORITATIVE spec→source mapping
owns_sections: []               # for co-owned aggregator files: the delimited section(s) this spec owns
variants: [primary, secondary]  # ui-components only (optional)
error_style: result             # behavioral specs only: result|raise (see scot.md §6)
---
```

- `source:` is the **single authoritative home** of the spec→source mapping. The
  index `source` column is **derived** from it (a command fills it; never
  hand-edited). A **new** entity proposes paths from `.sdd/target.md` conventions;
  an **existing** one points at the real files.
- **Default ownership: one file ↔ one spec.** A shared aggregator file (barrel,
  shared `types`) MAY be co-owned, but only if every co-owner declares it in
  `source:` and names its `owns_sections:`. Undeclared shared ownership is a
  blocking finding (reuse-analyst flags it; analysis-gatekeeper blocks it).
- A purely-compositional feature (no coordinator code) has `source: []` and only
  drives integration tests.

### `kind:` → form (which body the spec carries)

| `kind:`        | Category    | Body form                                   |
|----------------|-------------|---------------------------------------------|
| `service`      | behavioral  | **SCoT** (`.claude/sdd/scot.md`)                   |
| `controller`   | behavioral  | **SCoT**                                    |
| `use-case`     | behavioral  | **SCoT** (orchestration: cross-class calls) |
| `entity`       | structural  | declarative — field table, relations, invariants |
| `dto`          | structural  | declarative — field/shape table             |
| `enum`         | structural  | declarative — value table                   |
| `interface`    | structural  | declarative — signatures only, **no body**  |
| `config`       | structural  | declarative — key/type/default table        |
| `module`       | structural  | declarative — module overview (purpose, contained entries, boundaries & dependencies) |
| `gui`          | ui          | **UI schematic** (`.claude/sdd/ui-schema.md`)      |

A **stub/mock is never specced separately** — it is auto-derived from its
`interface` spec (empty implementation returning defaults), for not-yet-ready
dependencies or for tests.

### Required spec sections (all kinds)

`# Purpose` · `# Public interface` (inputs / outputs / errors) · `# Invariants & rules` ·
the **body form** for the `kind` · `# Acceptance criteria` (each `ACn` in
Given/When/Then). Behavioral specs add the SCoT block; entities add the field
table; `gui` **screens** (`CLS-*`) add the five UI-schema sections.

Two kinds shape their sections differently: a **`module`** spec uses `# Purpose` ·
`# Contained entries` · `# Boundaries & dependencies` (no Public-interface/Invariants
pair); a **shared UI component** (`COMP-*`, `kind: gui`) replaces `# Public interface`
and `# Invariants & rules` with the `.claude/sdd/ui-schema.md` §6 sections (Props · Variants ·
Visual states · Events · Accessibility), which carry its interface and rules.

---

## 4. Index row schema

Each index lists **all** entries of one level. Columns (min):

```
| id | name | description (WHAT, one line) | module | depends_on | spec | source | status |
```

- `description` says **what it represents**, never how it is built.
- `module` shows the partitioning (the level's home module).
- `source` is **derived** from each spec's `source:` front-matter.
- `status` is the **canonical** lifecycle home for the entry (see §5).
- `ui-components.index.md` adds a `layer` and a `variants` column.
- Shared non-UI abstractions (`SHR-*`, in `specs/shared/`) are listed in
  `classes.index.md` (they are implementation units); there is no separate shared
  index.
- Cell formats: `depends_on` is a comma-separated id list **without brackets**
  (`—` when none); `spec` / `source` show `—` when there is no file (`source: []`).

Agents **read the index first** and open only the specs they need (lazy loading,
token saving).

---

## 5. Status lifecycle & who writes what

Per-entity status lives in the **index row**: `draft → reviewed → approved`.

- `draft` — spec authored, not yet passed analysis gate.
- `reviewed` — passed the analysis gate (spec is internally sound).
- `approved` — implemented + code gate PASS + tests green + test gate PASS.

**Backward transitions (a spec change after `reviewed`).** Status moves forward by
default, but **code is only ever generated from a `reviewed` spec**. So whenever a
spec changes after it reached `reviewed`/`approved` — a gate routing a **spec bug**,
or a deliberate **feature evolution** — it must **re-pass the analysis gate** before
code is (re)generated: the
**command** demotes the affected entity `reviewed → draft` (or `approved → draft` if
it was already approved), `spec-writer` fixes it, then it re-advances the normal
forward path `draft → reviewed` (analysis gate) → `reviewed → approved` (test gate).
The command performs every status edit; gatekeepers only judge. A demotion never
happens for a **code** or **test** bug — those leave the spec and its status untouched
(only the code or the test is fixed).

**Separation of duties (strict):**

- **Gatekeepers JUDGE only.** They write a structured verdict to `.sdd/state.md`
  and never edit specs, code, tests, or index `status`.
- **The slash command (main session) ADVANCES `status`** in the index from the
  latest verdict in `.sdd/state.md`. So `state.md` records *why*; the index
  records *where each entity stands*.
- **Authors WRITE artifacts** (specs/code/tests/impl-notes) and never write
  verdicts.

---

## 6. `.sdd/state.md` verdict record format

`state.md` is **append-only**. Every gate appends one record:

```
## <ISO-8601 timestamp> — <gate-agent> — <PASS|REJECT>
- scope: <ids reviewed, e.g. FEAT-001, CLS-regCtrl>
- phase: <analysis|code|test>
- iteration: <n>/<budget>
- verdict: PASS | REJECT
- reasons:
  - <blocking reason 1 (with the spec id / AC id / branch id it concerns)>
  - <blocking reason 2>
- routing: <none | spec-writer | reuse-analyst | code-implementer | test-writer>   # REJECT only
```

The driving command reads the **latest** record for a scope to decide: advance
status (PASS) / re-invoke the routed author (REJECT) / escalate (budget overflow).

---

## 7. Iteration budgets & routing (feedback loops)

Configurable defaults (a project may override them in `.sdd/target.md`):

| Loop      | Budget | On overflow            |
|-----------|--------|------------------------|
| analysis  | 3      | escalate to the human  |
| code      | 3      | escalate to the human  |
| test      | 5      | escalate to the human  |

**Failure routing (test-gatekeeper triage)** — *MD is the source of truth, so a
red test never patches code arbitrarily*:

- **spec bug** → `spec-writer` (fix spec first, then regenerate code).
- **code bug** → `code-implementer` (minimal diff to match the spec).
- **test bug** → `test-writer` (fix the test; it must assert a spec AC/branch).
- **build / setup failure** (the suite never ran — non-zero `install`/`build` phase in `tests/REPORT.md`) → `code-implementer` for a compile/build error, or **escalate** for a missing dependency / tooling (env / `MOD-build`). Judge run-health *before* coverage; a non-running suite is never a `test-writer` coverage gap.

---

## 8. Change policy — edit by default, regenerate by exception

- **Edit (default), for bug-fix AND feature evolution:** the code-implementer
  reads spec + `.sdd/impl-notes/<id>.md` + existing source and applies a **minimal
  diff** to the file(s) the changed spec maps to. It **never rewrites a whole
  file** for a small change.
- **Spec stays authority:** the edit brings code to the spec, never the reverse.
  If a failure reveals the spec is wrong, fix the **spec first**.
- **Back-propagate to MD:** any decision SCoT omits (library, API binding, idiom,
  edge case, bug-fix lesson) goes into `.sdd/impl-notes/<id>.md` — never the spec.
- **Regenerate (exception):** whole-file rewrite only when the file is **new**,
  the spec changed substantially, or the file drifted badly. Driven by spec +
  impl-notes; still one file at a time.
- **Equivalence = the tests**, not a textual diff. A regenerated file must pass
  the same independent, spec-derived suite.

---

## 9. Agent roster & isolation matrix (single source)

Eleven roles; **ten are subagents** in `.claude/agents/`. The **orchestrator is
NOT a subagent** — orchestration lives in the slash commands run by the main
session (the only level that reliably invokes subagents via Task). Subagents are
single-purpose workers: read inputs from files, write outputs/verdicts to files.

| Agent | Role | May READ | May WRITE | tools frontmatter | Reads `src/`? |
|---|---|---|---|---|---|
| `plan-architect` | requirement → plan of indexes/specs | `requirements/`, `specs/` (existing), `.claude/sdd/`, `.sdd/target.md` | `plan/`, `.sdd/target.md` | `Read, Write, Glob, Grep` | no |
| `plan-gatekeeper` | judge the plan | `requirements/`, `plan/`, `specs/` (existing), `.claude/sdd/`, `.sdd/target.md` | `.sdd/state.md` | `Read, Glob, Grep` | no |
| `spec-writer` | write indexes + specs (4 levels + MOD-build) | `plan/`, `requirements/`, `specs/`, `.claude/sdd/{conventions,scot,ui-schema}.md`, `.sdd/target.md`, `.claude/sdd/templates/` | `specs/` (incl. indexes) | `Read, Write, Glob, Grep` | no |
| `reuse-analyst` | dedupe + promote shared specs (pure author) | `specs/`, `.claude/sdd/{conventions,ui-schema}.md` | `specs/` | `Read, Write, Glob, Grep` | no |
| `analysis-gatekeeper` | judge specs (the only spec-phase blocker) | `specs/`, `requirements/`, `.claude/sdd/`, `.sdd/` | `.sdd/state.md` | `Read, Glob, Grep` | no |
| `code-implementer` | specs → source (create / minimal diff) | `specs/`, `.claude/sdd/{conventions,scot,ui-schema}.md`, `.sdd/target.md`, `.sdd/impl-notes/`, `src/` | `src/` (declared paths), `.sdd/impl-notes/<id>.md` | `Read, Write, Edit, Glob, Grep` | yes (edit) |
| `code-gatekeeper` | judge code ≡ spec | `specs/`, `.sdd/impl-notes/`, `src/`, `.claude/sdd/`, `.sdd/target.md` | `.sdd/state.md` | `Read, Glob, Grep, Bash` (read-only) | yes (review) |
| `test-writer` | specs → tests (independent oracle) | behavioral spec sections, `.claude/sdd/{conventions,scot,ui-schema}.md`, `.sdd/target.md` | `tests/` | `Read, Write, Glob` | **no — by role** |
| `test-runner` | run tests, write report | `tests/`, `src/`, `.claude/sdd/conventions.md`, `.sdd/target.md` | `tests/REPORT.md` | `Read, Write, Glob, Bash` | yes |
| `test-gatekeeper` | verify coverage + triage failures | `tests/REPORT.md`, `specs/`, `src/`, `tests/`, `.claude/sdd/`, `.sdd/` | `.sdd/state.md` | `Read, Glob, Grep` | yes |

The `.claude/sdd/` contracts (`conventions`/`scot`/`ui-schema`) and the templates ship
with the **tool** and are **read-only** — no agent authors or edits them. `plan-architect`
only writes/extends the per-project `.sdd/target.md` from the user prompt (asking the
human if the stack is unstated).

**test-writer independence is SOFT** (by design, no runtime hook): minimal tools
(no `Bash`) + an explicit NON-GOAL ("never read `src/` or `.sdd/impl-notes/`") +
the test-gatekeeper rejecting tests that assert implementation detail. The
**code-implementer is NOT isolated**; it is kept honest by spec-as-authority + the
code-gatekeeper + the independent test suite.

**Model assignment (Opus ↔ Sonnet).** Each agent's frontmatter pins a `model:`
balanced to its job — **Opus** for under-specified authoring and high-consequence
judgment, **Sonnet** for mechanical / checklist work a concrete contract (or a
downstream gate) already constrains. A project MAY override any agent's `model:`.

| Agent | `model` | Why |
|---|---|---|
| `plan-architect` | `opus` | architecture & decomposition — many valid structures, high leverage |
| `spec-writer` | `opus` | authors SCoT logic, invariants, testable ACs — the creative core |
| `reuse-analyst` | `opus` | designs the *smallest justified* set of shared abstractions |
| `code-implementer` | `opus` | faithful SCoT→code, minimal diffs, concretization decisions |
| `analysis-gatekeeper` | `opus` | the only spec-phase blocker; judges self-sufficiency & testability |
| `test-gatekeeper` | `opus` | triages spec/code/test bug — a mis-route costs a whole iteration |
| `plan-gatekeeper` | `sonnet` | structural checklist on the plan; backstopped by the analysis gate |
| `code-gatekeeper` | `sonnet` | verifies code vs a concrete spec; backstopped by the test suite |
| `test-writer` | `sonnet` | mechanical coverage — one test per `ACn`/arm from concrete specs |
| `test-runner` | `sonnet` | runs canonical commands, parses, emits the fixed REPORT format |

---

## 10. Command roster & the loop each drives

Seven commands in `.claude/commands/` (five flow modes + two read-only helpers).
Modes 1–4 are the manual, human-in-control path; mode 5 is automatic, human out of
the loop. Each command runs in the main
session and drives its loop by: invoke agent → read verdict from `.sdd/state.md`
→ decide (proceed / re-invoke routed author / escalate) → advance index `status`.

| Command | Mode | Drives |
|---|---|---|
| `/sdd-plan` | 1. Plan | `plan-architect` → `plan-gatekeeper` (loop ≤3). Derives the stack → `.sdd/target.md`. No specs/code. (The `.claude/sdd/` contracts ship with the tool, read-only.) |
| `/sdd-specify` | 2. Plan approval | capture human approval → `spec-writer` → `reuse-analyst` → `analysis-gatekeeper` (loop ≤3). Advances status to `reviewed`. |
| `/sdd-implement` | 3. Implement | `code-implementer` → `code-gatekeeper` (loop ≤3), in `depends_on` order. |
| `/sdd-test` | 4. Test run | `test-writer` → `test-runner` → `test-gatekeeper` (loop ≤5); triage routes back. On full green → status `approved`. |
| `/sdd-auto` | 5. Automatic | all of the above end-to-end, **one vertical slice at a time** in `depends_on` order, human NOT in the loop; escalate only on budget overflow. |
| `/sdd-trace` | helper | reconstruct & print REQ→FEAT→CLS→SOURCE→TEST from indexes + front-matter. Read-only. |
| `/sdd-status` | helper | print a per-entity lifecycle dashboard (status + latest verdict + resume point) from indexes + `state.md`. Read-only. |

---

## 11. The ROLE prompt (mandatory header of every agent file)

Every `.claude/agents/*.md` body begins with this block, before any instructions:

```
ROLE: <one-line identity, e.g. "You are the Spec Writer.">
MISSION: <the single outcome this agent owns>
MINDSET: <values it optimizes for — MUST include the two cross-cutting values>
NON-GOALS: <what it must never do>
```

The MINDSET line must explicitly carry both cross-cutting values:
**"Markdown is the source of truth (authority); reuse over repetition (DRY)."**

---

## 12. Topological processing & vertical slices

- Process entities in **`depends_on` topological order** (dependencies first).
- On a dependency **cycle**, generate the `interface`/`contract` specs first and
  let implementations depend on the interface (breaks the cycle); stubs are
  derived from the interface meanwhile.
- In **auto mode**, work **one vertical slice (feature/module) at a time** to
  bound context and token cost: take a feature, pull its `depends_on` closure,
  and run plan→spec→code→test for just that slice before moving on.

---

## 13. Traceability (reconstructable, not a separate file)

The chain **REQUIREMENT → FEATURE → CLASS → SOURCE → TEST** is rebuilt on demand
from: the indexes (feature→module, class→feature, the derived `source` column) +
each spec's `requirements:` and `source:` front-matter + the source files'
traceability headers + the tests' coverage ids. `/sdd-trace` prints it.

Every generated source file carries a header pointing back to its spec, e.g.:

```
// spec: CLS-regCtrl RegistrationController — specs/classes/CLS-regCtrl.spec.md
```

(comment syntax per the target language).

---

## 14. `tests/REPORT.md` format (test-runner → test-gatekeeper contract)

`test-runner` writes this file (overwritten each run); `test-gatekeeper` parses it.
It MUST follow this fixed structure so the gatekeeper can read it without heuristics.
Note the division of labour: **coverage** of the spec (every `ACn` and every SCoT
branch arm having a test) is verified by the gatekeeper from the tagged test files in
`tests/**`; this report supplies the **run result** — what ran, pass/fail counts, and
each failure with the canonical coverage id (`.claude/sdd/scot.md` §7.3) it asserts.

```
# Test Report

## Run
- timestamp: <ISO-8601>
- commands: install=<cmd> | build=<cmd> | unit=<cmd> | int=<cmd>
- exit-status: <0 | first non-zero phase code>
- phase-reached: <install | build | unit | integration | complete>
- tooling: <none | note about a missing/placeholder command or reporter>

## Summary
- total: <n>
- passed: <n>
- failed: <n>
- skipped: <n>

## Failures
### <test name>
- coverage: <canonical coverage id(s), .claude/sdd/scot.md §7.3 — e.g. CLS-regCtrl::register#B1.then and/or CLS-regCtrl#AC2 | unknown>
- message: <assertion or error message>
- excerpt: |
    <trimmed stack / output excerpt>
```

`exit-status` + `commands` let the gatekeeper confirm the canonical `.sdd/target.md`
command actually ran; a halted install/build is reported via `phase-reached`, never
hidden. A failure whose `coverage` is `unknown` is flagged, never invented.
