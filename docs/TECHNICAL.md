# SDD Workflow — Technical Reference

> Production reference for the Spec-Driven Development (SDD) agentic workflow:
> how the **commands** orchestrate, how the **agents** work, and how every artifact
> flows through the pipeline. This document describes the system **as implemented**.
> The machine-read contract is `.claude/sdd/conventions.md` (plus `.claude/sdd/scot.md` and
> `.claude/sdd/ui-schema.md`); this file is the human-read companion. For a short overview
> see `README.md`.

**Audience:** engineers operating, supervising, or extending the workflow.
**Conventions in this doc:** `§N` refers to a section of the named `.claude/sdd/*.md`
contract; `<id>` placeholders follow the id scheme in §[Identifiers](#32-identifier-scheme).

---

## Table of contents

1. [Principles](#1-principles)
2. [Architecture](#2-architecture)
3. [The artifact model](#3-the-artifact-model)
4. [The agents](#4-the-agents)
5. [The commands](#5-the-commands)
6. [Gates, feedback loops & routing](#6-gates-feedback-loops--routing)
7. [Isolation & trust model](#7-isolation--trust-model)
8. [End-to-end walkthroughs](#8-end-to-end-walkthroughs)
9. [Supervision & traceability](#9-supervision--traceability)
10. [Operating the workflow](#10-operating-the-workflow)
11. [Extending the workflow](#11-extending-the-workflow)
12. [Design rationale](#12-design-rationale)
13. [Glossary](#13-glossary)

---

## 1. Principles

Five principles govern every part of the system.

1. **Markdown is the source of truth (authority).** The Markdown specs and indexes
   define *what* the system does and *how it is structured*. Source code and tests
   are **derived** artifacts. In any conflict, **the spec wins and the code is
   corrected** — never the reverse. This is *authority*, not *blindness*: agents may
   read code; they may never let code redefine the spec.

2. **Reuse over repetition (DRY).** Prefer one shared abstraction over duplicated
   logic or markup. *Discover-before-create*: read the indexes before authoring
   anything new. Duplication above a small threshold is a blocking defect, especially
   in the frontend (a mandatory UI component library exists from the start).

3. **Spec-first scope.** The workflow targets work where specs exist before code: a
   brand-new project, or a feature added to a project already built with SDD.
   Reverse-engineering legacy non-spec'd code is out of scope.

4. **File-only, stateless agents.** Every agent runs in its own fresh context and
   communicates with the others **only through files** (specs, indexes,
   `.sdd/state.md`, impl-notes, `tests/REPORT.md`). No agent relies on another
   agent's conversational memory. Any agent can therefore be (re)invoked cold with
   only the file paths it needs. *Consequence:* the whole pipeline can run from a
   project of only Markdown files, with no chat history — the one unavoidable human
   touch-point is supplying the **stack** if the requirement does not state it.

5. **Commands orchestrate; agents are single-purpose workers.** Claude Code subagents
   cannot reliably spawn other subagents, so **orchestration lives in the slash
   commands** run by the main session (the only level that reliably invokes subagents
   via the `Task` tool). There is **no orchestrator subagent**. Each command: invokes
   an agent → reads the verdict from `.sdd/state.md` → decides (proceed / re-invoke
   the routed author / escalate) → advances the index `status`.

---

## 2. Architecture

### 2.1 Layered model

```
 ┌──────────────────────────────────────────────────────────────────────┐
 │ CONTRACTS (.claude/sdd/) — authored once, read by every agent                │
 │   conventions.md · scot.md · ui-schema.md · target.md(*)              │
 └──────────────────────────────────────────────────────────────────────┘
                                  │ govern
                                  ▼
 ┌──────────────────────────────────────────────────────────────────────┐
 │ SPECS (specs/) — the source of truth                                  │
 │   indexes/*.index.md  +  modules/ features/ model/ classes/           │
 │   ui-components/ shared/  (copy-from forms live in .claude/sdd/templates/)   │
 └──────────────────────────────────────────────────────────────────────┘
                                  │ derived into                       ▲
                                  ▼                                    │ verified against
 ┌───────────────────────────────────────┐   ┌──────────────────────────────────┐
 │ CODE (src/) — derived                  │   │ TESTS (tests/) — derived,        │
 │   one file per spec source: mapping    │   │ INDEPENDENT oracle from specs    │
 └───────────────────────────────────────┘   └──────────────────────────────────┘
                                  │ run                                 │
                                  ▼                                     │
                         tests/REPORT.md  ──────── judged by ──────────┘
```

`(*) target.md` is per-project (written by the AI at plan time); the other three
contracts ship with the tool.

### 2.2 Orchestration model

- The **main session** runs a slash command. The command is a procedure the main
  session executes itself; it is the **orchestrator**.
- The command invokes **subagents** via the `Task` tool, passing only file paths/ids.
- Each subagent reads its inputs from files, does one job, writes its outputs to
  files (an author writes artifacts; a gatekeeper appends a verdict), and returns.
- The command reads the result **from the file it produced** (e.g. the latest
  `.sdd/state.md` verdict), never from the agent's return text as state, and decides
  the next move.
- **The command — not any agent — edits the index `status`.** Gatekeepers judge;
  authors write artifacts; the command advances/demotes lifecycle status.

### 2.3 The eleven roles

Ten are subagents in `.claude/agents/`; the eleventh ("orchestrator") is the
commands themselves.

| # | Role | Kind | Phase |
|---|------|------|-------|
| 1 | `plan-architect` | author | plan |
| 2 | `plan-gatekeeper` | gate | plan |
| 3 | `spec-writer` | author | spec |
| 4 | `reuse-analyst` | author | spec |
| 5 | `analysis-gatekeeper` | gate | spec |
| 6 | `code-implementer` | author | code |
| 7 | `code-gatekeeper` | gate | code |
| 8 | `test-writer` | author | test |
| 9 | `test-runner` | runner | test |
| 10 | `test-gatekeeper` | gate + triage | test |
| — | *orchestrator* | the 7 commands (main session) | all |

---

## 3. The artifact model

### 3.1 Folder layout

```
# THE TOOL — .claude/ (immutable; one portable folder; a project never edits it)
.claude/agents/*.md                  the 10 subagents
.claude/commands/*.md                the 7 slash commands
.claude/sdd/
  conventions.md                     THE canonical reference (ids, front-matter, status, verdicts, rosters)
  scot.md                            canonical SCoT grammar (behavioral specs)
  ui-schema.md                       canonical UI-schematic convention (gui specs) + the GUARANTEED baseline UI library (§9)
  templates/*.template.md            the forms new specs are copied from (incl. target.template, impl-notes.template)
docs/TECHNICAL.md                    this document

# THE PROJECT — created/written at runtime by the workflow
requirements/REQUIREMENT.md          raw + refined requirement, with REQ-* ids
plan/PLAN.md                         output of the Plan mode
.sdd/
  target.md                          stack + source-path conventions + canonical commands (AI-written, per project)
  state.md                           append-only verdict / audit log
  impl-notes/<spec-id>.md            implementer-owned concretization (NOT part of the gated spec)
specs/
  indexes/{modules,features,model,classes,ui-components}.index.md   one index PER LEVEL
  modules/<id>.spec.md               incl. MOD-build (build/config/CI/schema changes)
  features/<id>.spec.md              use-cases (orchestration + integration acceptance)
  model/<id>.spec.md                 domain entities (fields, types, relations, constraints)
  classes/<id>.spec.md               per-method SCoT services/controllers AND feature gui screens
  ui-components/<id>.spec.md         UI component library — baseline GUARANTEED per GUI project (ui-schema §9), then enriched
  shared/<id>.spec.md                shared non-UI abstractions (SHR-*) — indexed in classes.index.md
src/                                 GENERATED (derived)
tests/                               GENERATED (unit←classes, integration←features, constraints←entities)
```

### 3.2 Identifier scheme

Stable, never renumbered/renamed (a rename = a new id + deprecation of the old).

| Prefix | Level / kind | Form | Example |
|---|---|---|---|
| `MOD-` | module | `MOD-<kebab>` | `MOD-api`, `MOD-build` |
| `FEAT-` | feature / use-case | `FEAT-<nnn>` | `FEAT-001` |
| `ENT-` | domain entity | `ENT-<kebab>` | `ENT-user` |
| `CLS-` | class / service / screen | `CLS-<lowerCamel>` | `CLS-regCtrl` |
| `COMP-` | shared UI component | `COMP-<lowerCamel>` | `COMP-button` |
| `SHR-` | shared non-UI abstraction | `SHR-<lowerCamel>` | `SHR-passwordHasher` |
| `REQ-` | requirement (in `REQUIREMENT.md`) | `REQ-<nnn>` | `REQ-001` |
| `AC` | acceptance criterion (in-spec) | `AC<n>` | `AC1` |
| `B` | SCoT branch (in-spec) | `B<n>` + arm | `B1.then`, `B3.empty` |

`MOD-build` is **mandatory in every project**: it owns build files, dependency
manifests, config, CI, and the **DB schema changes derived from the entity specs**
(schema changes are never hand-authored independently).

### 3.3 Spec levels and the `kind:` → form mapping

A spec's `kind:` front-matter field decides which **body form** it carries.

| `kind:` | Category | Body form | Lives in |
|---|---|---|---|
| `service`, `controller` | behavioral | **SCoT** (per-method) | `specs/classes/` |
| `use-case` | behavioral | **SCoT** (cross-class orchestration) | `specs/features/` |
| `entity` | structural | declarative: field table + relations + invariants | `specs/model/` |
| `dto`, `enum`, `interface`, `config` | structural | declarative tables (interface = signatures only, no body) | `specs/shared/` or `specs/classes/` |
| `module` | structural | overview: Purpose · Contained entries · Boundaries & dependencies | `specs/modules/` |
| `gui` (screen) | ui | UI schematic (5 sections) | `specs/classes/` |
| `gui` (component, `COMP-*`) | ui | UI schematic + §6 component sections | `specs/ui-components/` |

The **four spec levels** map onto these kinds:
- **Feature** (`use-case`) — the scenario, the collaborators referenced **by id**, a
  high-level orchestration SCoT, and integration acceptance criteria. If it needs real
  coordinator code it declares a `source:` file; otherwise `source: []` (purely
  compositional — it only drives integration tests).
- **Entity** (`entity`) — the single source from which both the code entity **and** the
  DB schema change derive. Declarative, no SCoT.
- **Class/service** (`service`/`controller`) — the detailed per-method SCoT + invariants.
- **UI** (`gui`) — the schematic form: wireframe + component tree (by id) + state/events.

A **stub/mock is never specced separately** — it is auto-derived from an `interface`
spec (an empty implementation returning declared defaults), used for not-yet-ready
dependencies and by tests.

### 3.4 Required spec sections (and the two exceptions)

Most kinds carry: `# Purpose` · `# Public interface` (inputs/outputs/errors) ·
`# Invariants & rules` · the **body form** · `# Acceptance criteria` (each `ACn` in
Given/When/Then). **Two exceptions** (conventions §3):
- a **`module`** spec uses `# Purpose` · `# Contained entries` · `# Boundaries &
  dependencies` (no Public-interface/Invariants pair);
- a shared **`COMP-*`** component replaces `# Public interface` + `# Invariants & rules`
  with the `ui-schema §6` sections (Props · Variants · Visual states · Events ·
  Accessibility), which carry its interface and rules.

Every spec must be **self-sufficient to regenerate its code from scratch** without
reading existing source — this keeps full regeneration possible as the safety net.

### 3.5 Front-matter schema

```yaml
---
id: CLS-regCtrl                 # required — matches the filename and an index row
name: RegistrationController    # required
kind: controller                # required — drives the body form (§3.3)
module: MOD-api                 # required for class/model/feature/gui
layer: organism                 # ui-components only: atom|molecule|organism|layout
status: draft                   # MIRRORS the index row (the index is canonical)
depends_on: [CLS-userRepo, SHR-passwordHasher, ENT-user]   # ids, topological
requirements: [REQ-001]         # back-link to requirement id(s)
source: [src/api/RegistrationController.ts]   # AUTHORITATIVE spec→source mapping
owns_sections: []               # for a co-owned aggregator file: the section(s) this spec owns
variants: [primary, secondary]  # ui-components only (optional)
error_style: result             # behavioral only: result|raise (matches the SCoT error-style)
---
```

### 3.6 Indexes

**One index per level** (not per module). Each row:

```
| id | name | description (WHAT, one line) | module | depends_on | spec | source | status |
```

- `source` is **derived** from each spec's authoritative `source:` front-matter
  (a command fills it; it is never hand-edited).
- `ui-components.index.md` adds `layer` and `variants` columns (after `module`).
- **`SHR-*` rows live in `classes.index.md`** (they are implementation units); there
  is no separate shared index.
- Cell formats: `depends_on` is a comma-separated id list **without brackets** (`—`
  when none); `spec`/`source` show `—` when there is no file (`source: []`).
- Agents **read the index first** and open only the specs they need (lazy loading →
  fewer tokens).
- **Scaling option:** a large multi-module project may split the fine-grained indexes
  (`classes`, `model`, `ui-components`) per module under
  `specs/indexes/<level>/<MOD-id>.index.md`, keeping `modules` and `features` global.
  This is decided in the plan.

### 3.7 Status lifecycle

Per-entity status lives in the **index row** and moves `draft → reviewed → approved`:

| status | meaning |
|---|---|
| `draft` | spec authored, analysis gate not yet passed |
| `reviewed` | passed the analysis gate (internally sound) |
| `approved` | implemented + code-gate PASS + tests green + test-gate PASS |

**Backward transitions.** Status normally moves forward, but **code is only ever
generated from a `reviewed` spec**. So whenever a spec changes after it reached
`reviewed`/`approved` — a gate routing a **spec bug**, or a deliberate **feature
evolution** — the spec must re-pass the analysis gate before code is (re)generated:
the **command** demotes the entity `reviewed → draft` (or `approved → draft`), the
`spec-writer` fixes it, and it re-advances the normal forward path. A demotion never
happens for a *code* or *test* bug — those leave the spec untouched.

**Who writes what:** gatekeepers JUDGE (verdict → `.sdd/state.md`) and never touch
`status`; authors WRITE artifacts; the **command** advances/demotes `status` from the
latest verdict.

### 3.8 `.sdd/state.md` — verdict log (append-only)

Every gate appends one record. The driving command reads the **latest** record for a
scope to decide.

```
## <ISO-8601 timestamp> — <gate-agent> — <PASS|REJECT>
- scope: <ids reviewed, e.g. FEAT-001, CLS-regCtrl>
- phase: <analysis|code|test>
- iteration: <n>/<budget>
- verdict: PASS | REJECT
- reasons:
  - <blocking reason, citing the spec id / ACn / branch arm it concerns>
- routing: <none | spec-writer | reuse-analyst | code-implementer | test-writer>
```

`state.md` records **why**; the index records **where each entity stands**. (The plan
gate also uses `phase: analysis`, distinguished from the spec gate by its `scope`.)

### 3.9 `tests/REPORT.md` — run report (the test-runner → test-gatekeeper contract)

`test-runner` writes it (overwritten each run); `test-gatekeeper` parses it. Fixed
structure (conventions §14): `## Run` (timestamp, commands, exit-status,
phase-reached, tooling) · `## Summary` (total/passed/failed/skipped) · `## Failures`
(per failing test: `coverage`, `message`, `excerpt`). **Coverage** of the spec is
verified by the gatekeeper from the tagged test files in `tests/**`; the report
supplies the run result and the coverage id of each failure.

### 3.10 Coverage-id notation (the join key)

One canonical notation links test-writer → test-runner → test-gatekeeper
(`scot.md §7.3`). **Use it verbatim everywhere:**
- **branch arm:** `<spec-id>::<function>#<arm-id>` — e.g. `CLS-regCtrl::register#B1.else`
- **acceptance criterion:** `<spec-id>#ACn` — e.g. `CLS-regCtrl#AC2`

The test-writer tags each test with this id; the test-runner echoes it verbatim into
`tests/REPORT.md`; the test-gatekeeper joins on it to verify coverage and to triage.

### 3.11 SCoT in one paragraph (full grammar: `scot.md`)

SCoT (Structured Chain-of-Thought) is the only pseudo-code allowed in behavioral
specs. Constructs: **sequence**, **branch** (`IF`/`SWITCH`), **loop**
(`FOR`/`WHILE`/`REPEAT`), plus `ASYNC`/`AWAIT` and `TRY`/`CATCH`. Every branching
construct carries a stable `[Bn]` id whose **arms are enumerated** (`B1.then`,
`B1.else`, `B2.case:<label>`, `B3.body`/`B3.empty`, `B4.body`/`B4.skip`,
`B6.ok`/`B6.catch:<Err>`). Functions declare explicit `INPUT`/`OUTPUT` and an
`error-style` (`result` or `raise`). SCoT contains **no** library names, language
syntax, or SQL — those are concretizations recorded in impl-notes. The complete arm
set of a function is its **coverage set**, which the test-writer must cover.

### 3.12 UI schematic in one paragraph (full convention: `ui-schema.md`)

A `gui` spec is a framework-agnostic schematic: an **ASCII wireframe**, a **component
tree** that composes library `COMP-*` **by id** (never re-describing them), a **State**
table, and an **Events** table (a non-trivial handler attaches a small SCoT snippet
with branch ids). Shared `COMP-*` specs add Props/Variants/Visual-states/Events/
Accessibility. Atomic-design layers: atom → molecule → organism → layout. No framework
code, no CSS/colors — only design-token *names* resolved from `target.md`.

### 3.13 Spec ↔ source mapping & traceability

Bidirectional, single authoritative home = the spec's `source:` front-matter:
- **spec → source:** the spec declares the file(s) it produces; the index `source`
  column is derived from it.
- **source → spec:** each generated file carries a header, e.g.
  `// spec: CLS-regCtrl RegistrationController — specs/classes/CLS-regCtrl.spec.md`
  (comment syntax per the target language).
- Default **one file ↔ one spec**; a shared aggregator file may be co-owned only if
  every co-owner declares it in `source:` and names its `owns_sections:`.

The full chain **REQUIREMENT → FEATURE → CLASS/ENTITY/COMPONENT → SOURCE → TEST** is
never a separate file — it is reconstructed on demand by `/sdd-trace` from indexes +
front-matter + headers + test coverage ids.

### 3.14 Tool content vs runtime artifacts

This repository is the **tool**. It ships only `.claude/` (agents, commands, and
`.claude/sdd/` = the `conventions`/`scot`/`ui-schema` contracts + the `templates/`
forms, including `target.template` and `impl-notes.template`) and `docs/`. The
per-project `.sdd/` (`target.md` / `state.md` / `impl-notes/`) is created at runtime.
In particular the **UI
library is not shipped as files**: for a GUI project the `spec-writer` materializes the
guaranteed baseline (`.claude/sdd/ui-schema.md` §9) into `specs/ui-components/`. The
layout in §3.1 is the **runtime** layout of a *project that uses the tool*:
`requirements/`, `plan/`, `src/`, `tests/`, and the populated `specs/` subfolders are
**created by the workflow inside that project**, not shipped in the tool repo.

Two runtime files are written directly from command/agent instructions (no template
file is read), so their format is documented here rather than shipped as a placeholder:

**`requirements/REQUIREMENT.md`** — written by `/sdd-plan` (the main session):

```
# Requirement — <title>

## Raw
<the user's prompt, verbatim>

## Refined
- REQ-001: <atomic, testable requirement>
- REQ-002: <…>

## Scope
- In:  <…>
- Out: <…>

## Acceptance at a glance
- REQ-001 → <observable signal that it is met>
```

**`plan/PLAN.md`** — written by `plan-architect`:

```
# Plan — <project or feature>

## Classification
new app | feature on an existing SDD project

## Target stack
<one-line summary → written in full to .sdd/target.md>

## Index granularity
single global index per level | per-module split (with rationale)

## Entities to create / modify
| id | level | module | depends_on | source | requirement | NEW/MODIFY |
| … | … | … | … | … | … | … |

## Vertical slices (depends_on topological order)
1. <slice id> — <entity ids>

## Shared / cross-cutting candidates (for the reuse-analyst)
- <pattern> → <proposed SHR-* / COMP-* abstraction>
```

---

## 4. The agents

Each agent below: **Mission**, **When invoked**, **Reads** / **Writes** (files only),
**Tools**, **Procedure**, **Veto criteria** (gatekeepers) or **Definition of done**
(authors), **Guardrails**. Every agent file opens with a `ROLE / MISSION / MINDSET /
NON-GOALS` block, and the MINDSET always carries the two cross-cutting values verbatim.

### 4.1 `plan-architect` (author · plan)
- **Mission:** turn the requirement into a PLAN of which indexes/specs to create or
  modify — no specs, no code.
- **When:** first step of `/sdd-plan` and Phase A of `/sdd-auto`.
- **Reads:** `requirements/REQUIREMENT.md`, existing `specs/` + indexes, `.claude/sdd/` + `.sdd/`.
- **Writes:** `plan/PLAN.md`, `.sdd/target.md` only (the `.claude/sdd/` contracts are read-only). It does **not** write `requirements/`.
- **Tools:** `Read, Write, Glob, Grep`.
- **Procedure:** classify NEW-project vs existing-SDD; derive the stack into
  `target.md` (**STOP and ask the human if the stack is unstated** — never assume a
  default); read the `REQ-*` ids the command assigned in `REQUIREMENT.md`; enumerate
  every entity (`id`, level, `module`, `depends_on`, `source`, requirement, NEW/MODIFY);
  decide index granularity (global default vs per-module split); order the work into
  vertical slices in `depends_on` topological order (cycles → interface-first); flag
  shared/cross-cutting candidates for the reuse-analyst (discover-before-create against
  the UI library).
- **Definition of done:** every entity carries all fields and a back-link to an
  existing `REQ-*`; slices topologically ordered; `target.md` complete; no spec/code.

### 4.2 `plan-gatekeeper` (gate · plan)
- **Mission:** judge `plan/PLAN.md`; PASS or REJECT with blocking reasons.
- **When:** after `plan-architect` (in `/sdd-plan` and `/sdd-auto` Phase A).
- **Reads:** `requirements/`, `plan/`, `specs/` (existing), `.claude/sdd/` + `.sdd/`.
- **Writes:** one verdict to `.sdd/state.md`.
- **Tools:** `Read, Glob, Grep`.
- **Veto criteria (REJECT if):** `target.md` missing or **not fully resolved** (any
  `<…>` left in §1 stack / §2 source-path conventions / §3 canonical commands); any
  entity missing `id`/`level`/`module`/`depends_on`/`source`/`requirement` or with a
  malformed id; a `depends_on` cycle without an interface-first break; a slice order
  that violates topology; a requirement covered by no entity; an entity pointing at a
  non-existent requirement; unflagged shared/cross-cutting duplication; a large project
  with no index-granularity decision; `MOD-build` absent.
- **Routing on REJECT:** always `plan-architect`.

### 4.3 `spec-writer` (author · spec)
- **Mission:** write the indexes and the specs at all levels (+ `MOD-build`), each in
  the form its `kind` requires.
- **When:** `/sdd-specify` step 2; `/sdd-auto` step 7a; re-invoked on a spec-bug route.
- **Reads:** `plan/`, `requirements/`, `specs/` (+ `ui-components.index` for
  discover-before-create), `.claude/sdd/{conventions,scot,ui-schema}.md`, `.sdd/target.md`, `.claude/sdd/templates/`.
- **Writes:** `specs/**` (specs + index rows), all at `status: draft`.
- **Tools:** `Read, Write, Glob, Grep`.
- **Procedure:** work entities in `depends_on` order; copy the matching template; fill
  front-matter; pick the body by kind (behavioral → SCoT; entity → field table +
  invariants; structural → declarative; **module → overview**; gui → schematic
  composing `COMP-*` by id — for a GUI project, FIRST materialize the guaranteed
  **baseline UI library** (ui-schema §9) from the template); create `MOD-build` (schema changes derived from entity specs;
  `MOD-build.depends_on` the entity-owning modules); write `ACn` in Given/When/Then;
  ensure each spec is regenerable; derive each index `source` column from `source:`
  front-matter. **For feature evolution, the command demotes an already-passed entity
  to `draft` before the spec-writer rewrites it.**
- **Definition of done:** every planned entity has an index row + a spec in the right
  folder; required sections present (honoring the `module`/`COMP-*` exceptions); SCoT
  branches have ids; gui references components by id; `SHR-*` rows go in
  `classes.index`.

### 4.4 `reuse-analyst` (author · spec)
- **Mission:** maximize reuse / eliminate duplication across the **specs** (Markdown
  only), before any code exists. A **pure author** — it edits, it does not judge.
- **When:** after `spec-writer`, before the analysis gate; re-run on any spec change.
- **Reads:** `specs/`, `.claude/sdd/conventions.md`, `.claude/sdd/ui-schema.md`.
- **Writes:** `specs/**` (promoted/edited specs, updated index rows), `specs/REUSE-REPORT.md`.
- **Tools:** `Read, Write, Glob, Grep`.
- **Procedure:** detect recurring logic / near-duplicate SCoT / repeated widgets /
  repeated types; promote shared widgets into `specs/ui-components/` (`COMP-*`, correct
  `layer`) and shared non-UI abstractions into `specs/shared/` (`SHR-*`); rewrite
  consumers to reference the promoted id; enforce discover-before-create; write the
  reuse report. Never promotes a pattern used in only one place.

### 4.5 `analysis-gatekeeper` (gate · spec)
- **Mission:** the single authority that blocks the spec phase.
- **When:** end of `/sdd-specify`; `/sdd-auto` step 7c; on a spec-bug route.
- **Reads:** all `specs/` + indexes, `requirements/`, `.claude/sdd/` + `.sdd/`.
- **Writes:** one verdict to `.sdd/state.md`.
- **Tools:** `Read, Glob, Grep`.
- **Veto criteria (REJECT if):** a spec is not self-sufficient to regenerate;
  front-matter missing/invalid; required sections missing **(honoring the
  `module`/`COMP-*` exceptions — do not reject a COMP for "missing Public
  interface")**; `ACn` missing / not testable / not Given-When-Then; a SCoT branch
  lacks an id or its arms are not enumerable; a gui screen re-describes a library
  component; the source mapping is malformed (paths violate `target.md`; undeclared
  shared ownership; index `source` ≠ spec `source:`); a requirement is untraceable;
  unjustified duplication above threshold (→ route `reuse-analyst`); a dependency cycle
  without an interface-first break; `MOD-build` missing or schema changes not derived from
  entities.
- **Routing on REJECT:** `spec-writer` (default) or `reuse-analyst` (duplication).

### 4.6 `code-implementer` (author · code)
- **Mission:** turn approved specs into source — create new files or apply **minimal
  diffs** — emitting the language/framework from `target.md`; record concretizations
  in impl-notes; never edit the gated spec.
- **When:** `/sdd-implement` step 3; `/sdd-auto` step 8a; on a code-bug/spec-bug
  re-route.
- **Reads:** the spec(s), `.claude/sdd/{conventions,scot,ui-schema}.md`, `.sdd/target.md`,
  `.sdd/impl-notes/<id>.md`, and the existing `src/` files it touches.
- **Writes:** `src/**` (declared `source:` paths), `.sdd/impl-notes/<id>.md`.
- **Tools:** `Read, Write, Edit, Glob, Grep`.
- **Procedure:** work in `depends_on` order (cycles → interfaces first, derive stubs);
  **new file** → generate from spec (+ impl-notes), stamp the traceability header;
  **existing file** → read it and apply the smallest diff that brings the touched
  entity to the spec (never a whole-file rewrite for a small change); map SCoT
  faithfully (sequence/branch/loop, error-style, invariants); emit shared `SHR-*`/
  `COMP-*` once and import everywhere; record every concretization (library/version,
  API binding, idiom, edge case, bug-fix lesson) into impl-notes. Regenerate a whole
  file only as an exception (new / substantially-changed spec / bad drift).
- **Definition of done:** every declared `source:` file exists with its header and
  matches the SCoT/interface/invariants; impl-notes updated; no spec edited.

### 4.7 `code-gatekeeper` (gate · code)
- **Mission:** verify code faithfully implements the spec and the on-disk spec↔source
  mapping is real, complete, and traceable.
- **When:** `/sdd-implement` step 4; `/sdd-auto` step 8b; after any in-loop code fix.
- **Reads:** specs, impl-notes, `src/`, `.claude/sdd/` + `.sdd/`.
- **Writes:** one verdict to `.sdd/state.md`.
- **Tools:** `Read, Glob, Grep, Bash` (**read-only** — compile/lint/inspect only;
  never mutate, install, format, or run tests-as-fixes).
- **Veto criteria (REJECT if):** code diverges from the SCoT / interface / invariants;
  a `source:` file is missing, or an orphan source file has no spec; a file lacks its
  traceability header; a concretization contradicts the spec; the gated spec was edited
  by the implementer; shared logic is duplicated instead of imported; (optional) a
  read-only build/lint fails. Per-kind it checks behavioral (SCoT realized), structural
  (tables match), **gui** (composes `COMP-*` by id, realizes State/Events incl. handler
  SCoT arms, Props/variants, accessibility intent), and notes that a `source: []` spec
  (module overview / compositional feature) has **no code of its own** to verify.
- **Routing on REJECT:** `code-implementer` (or `spec-writer` if the spec is at fault).

### 4.8 `test-writer` (author · test — the independent oracle)
- **Mission:** write tests that are an INDEPENDENT oracle derived only from the
  **behavioral** part of the specs — ≥1 test per `ACn` and per SCoT branch arm.
- **When:** `/sdd-test` step 1; `/sdd-auto` step 9a; re-invoked on a test-bug route.
- **Reads:** behavioral spec sections (public interface, ACs, SCoT), entity
  constraints, gui events; `.claude/sdd/{conventions,scot,ui-schema}.md`, `.sdd/target.md`. **Never
  reads `src/` or `.sdd/impl-notes/`.**
- **Writes:** `tests/**` (+ interface-derived stub helpers).
- **Tools:** `Read, Write, Glob` (**no `Bash` by design**).
- **Procedure:** unit tests from class specs (one per branch arm + per `ACn`),
  integration tests from feature specs, constraint tests from entity specs, gui tests
  from Events + Accessibility ACs; tag each test with its canonical coverage id
  (§3.10); stub not-yet-ready dependencies from their interface specs only.
- **Definition of done:** every in-scope `ACn` and SCoT arm has ≥1 test asserting a
  spec outcome (never an implementation detail); assertion style matches the spec's
  declared `error_style`.

### 4.9 `test-runner` (runner · test)
- **Mission:** run the spec-derived suite via the canonical commands from `target.md`
  and write a parseable `tests/REPORT.md`. Fixes nothing.
- **When:** `/sdd-test` step 2; `/sdd-auto` step 9b.
- **Reads:** `tests/`, `src/`, `.sdd/target.md`.
- **Writes:** `tests/REPORT.md`.
- **Tools:** `Read, Write, Glob, Bash`.
- **Procedure:** run `install` → `build` → `test-unit`/`test-int` (or `test-all`) from
  `target.md §3` (a missing/placeholder command → record it and exit non-zero); parse
  per-test pass/fail and recover each test's **canonical** coverage id, echoing it
  verbatim; capture failures with message + trimmed excerpt; write the §3.9 structure
  (`coverage: unknown` rather than inventing one when unrecoverable).

### 4.10 `test-gatekeeper` (gate + triage · test)
- **Mission:** verify coverage (every `ACn` and SCoT arm has a test, else REJECT) and
  triage each failure as a spec / code / test bug.
- **When:** `/sdd-test` step 3; `/sdd-auto` step 9c.
- **Reads:** `tests/REPORT.md`, specs, `tests/`, `src/` (read-only, for triage), `.claude/sdd/` + `.sdd/`.
- **Writes:** one verdict to `.sdd/state.md` with per-failure routing.
- **Tools:** `Read, Glob, Grep`.
- **Veto criteria (REJECT if):** any `ACn` or SCoT arm in scope is uncovered (→
  `test-writer`); a test asserts an implementation detail instead of a spec AC/branch
  (→ `test-writer`); any in-scope test is failing. **Triage** each failure: **spec bug**
  → `spec-writer` (fix spec, then regenerate code); **code bug** → `code-implementer`
  (minimal diff); **test bug** → `test-writer`. PASS only when coverage is complete
  **and** the suite is green.

---

## 5. The commands

Seven commands: five flow **modes** (1–4 manual/human-in-control, 5 automatic/human-
out-of-the-loop) and two read-only **helpers**. Each command runs in the main session
and is the orchestrator (§2.2).

| Command | Mode | Drives | Status effect |
|---|---|---|---|
| `/sdd-plan` | 1 Plan | `plan-architect` → `plan-gatekeeper` | none (no specs yet) |
| `/sdd-specify` | 2 Plan approval | `spec-writer` → `reuse-analyst` → `analysis-gatekeeper` | `draft → reviewed` |
| `/sdd-implement` | 3 Implement | `code-implementer` → `code-gatekeeper` | none (waits for tests) |
| `/sdd-test` | 4 Test run | `test-writer` → `test-runner` → `test-gatekeeper` | `reviewed → approved` |
| `/sdd-auto` | 5 Automatic | all of the above, slice by slice | `draft → reviewed → approved` |
| `/sdd-trace` | helper | reconstruct REQ→…→TEST + coverage gaps | none (read-only) |
| `/sdd-status` | helper | lifecycle dashboard + resume point | none (read-only) |

### 5.1 `/sdd-plan` — Mode 1 (Plan)
Turn the requirement into a plan; **no specs, no code**. Steps: capture/refine
`requirements/REQUIREMENT.md` from `$ARGUMENTS`, **assigning a stable `REQ-*` id to each
atomic requirement** (these are the back-link targets); invoke `plan-architect` (it
derives `target.md`); if the
stack is unstated, relay its question to the human and stop until answered; invoke
`plan-gatekeeper`; on PASS tell the human the plan is ready for `/sdd-specify`; on
REJECT re-invoke `plan-architect` with the reasons (analysis budget 3); on overflow
escalate. **Stops at an approved plan** — it does not write specs.

### 5.2 `/sdd-specify` — Mode 2 (Plan approval → write specs)
Represents the **human's approval** of the plan, then writes the spec corpus and drives
it through the analysis gate. Steps: confirm approval (do not proceed without it);
**(feature evolution: demote any in-scope `reviewed`/`approved` entity to `draft`
first)**; invoke `spec-writer` (rows at `draft`); invoke `reuse-analyst`; invoke
`analysis-gatekeeper`; on PASS advance each in-scope row `draft → reviewed`; on REJECT
re-invoke the routed author (`spec-writer` or `reuse-analyst`) and re-judge (analysis
budget 3); on overflow escalate. Output: the `specs/` corpus, ready for human review,
then `/sdd-implement`.

### 5.3 `/sdd-implement` — Mode 3 (Implement)
Turn `reviewed` specs into source and drive the code gate. Steps: resolve the work set
(`$ARGUMENTS` or all `reviewed` specs) and order it topologically; for each entity
invoke `code-implementer` then `code-gatekeeper`; on PASS proceed; on REJECT route by
the verdict (code budget 3):
- **code bug** → re-invoke `code-implementer` (minimal diff) → re-judge.
- **spec bug** → demote `reviewed → draft`, re-run **inline** `spec-writer` →
  `reuse-analyst` → `analysis-gatekeeper` → re-advance to `reviewed`, then resume
  `code-implementer`. (Do **not** re-run the whole `/sdd-specify` — its human-approval
  step does not belong mid-implementation.) Spec rework does not consume the code
  budget.
On overflow escalate. **Does not set `approved`** — `reviewed → approved` requires
green tests (Mode 4). It records the code-gate PASS in `state.md` for `/sdd-test`.

### 5.4 `/sdd-test` — Mode 4 (Test run)
Generate spec-derived tests, run them, verify coverage, triage failures; on full green
advance `reviewed → approved`. Steps: invoke `test-writer` (independent oracle), then
`test-runner` (→ `tests/REPORT.md`), then `test-gatekeeper` (coverage + triage). On
PASS advance `reviewed → approved`; on REJECT route per triage (test budget 5):
- **test bug** → `test-writer` → redo run + judge.
- **code bug** → re-run the implement step (`code-implementer` → `code-gatekeeper`,
  iterating until the **code gate** PASSes, bounded by the test budget) → redo run +
  judge.
- **spec bug** → demote `reviewed → draft`, re-validate via `spec-writer` →
  `reuse-analyst` → `analysis-gatekeeper` → re-advance to `reviewed`, then re-run the
  implement step (with the **code gate**) → re-author + run + judge tests.
On overflow escalate. Every route loops back to the sub-phase that **re-gates the fix**.

### 5.5 `/sdd-auto` — Mode 5 (Automatic)
Run Plan → Specify → Implement → Test end-to-end, **human out of the loop**, escalating
only on budget overflow (and the one stack question if the stack is unresolvable).
Phase A = Plan (no approval stop). Phase B = build the **slice list** in `depends_on`
order (one vertical slice = a feature/module + its closure). Phase C, per slice in
order: **Specify** (7, `draft→reviewed`) → **Implement** (8, code gate) → **Test** (9,
`reviewed→approved`). REJECT routing re-gates: **spec bug → step 7**, **code bug → step
8**, **test bug → step 9b**; budget overflow → escalate (and stop that slice; other
approved slices stand). **Resume from files:** if interrupted, re-run `/sdd-auto` — it
reconstructs progress from `state.md` (`iteration: n/budget`) + index `status`, skips
`approved` slices, and re-enters the first non-`approved` slice at the implied
sub-phase.

### 5.6 `/sdd-trace` — Helper (read-only)
Reconstruct and print **REQUIREMENT → FEATURE → CLASS/ENTITY/COMPONENT → SOURCE →
TEST** for a focus id (or the whole project), from indexes + spec front-matter +
source headers + test coverage ids. Highlights AC/branch coverage gaps, orphan nodes
(no requirement back-link), dangling/headerless `source:` paths, and index↔spec
`source` drift. Writes nothing.

### 5.7 `/sdd-status` — Helper (read-only)
Print a **lifecycle dashboard**: per-entity `status` + last phase/verdict + iter/budget
+ `blocked-on`, a summary (counts per status, blocked entities, escalations), and the
**`/sdd-auto` resume point**. Surfaces (does not fix) status↔verdict inconsistencies
(e.g. `approved` but the latest test verdict is REJECT). Complements `/sdd-trace`:
`/sdd-status` answers *"where does each thing stand and what's next?"*.

---

## 6. Gates, feedback loops & routing

### 6.1 The gate model
Three gates, each owned by a gatekeeper that **judges only**: analysis
(`analysis-gatekeeper`), code (`code-gatekeeper`), test (`test-gatekeeper`). A
gatekeeper emits a structured `PASS`/`REJECT` verdict to `.sdd/state.md`; the
**command** acts on it. (The plan gate, `plan-gatekeeper`, guards the plan before the
spec phase.)

### 6.2 Iteration budgets
Defaults (configurable in `target.md §4`): **analysis 3 · code 3 · test 5**. Each
REJECT → re-author → re-judge cycle counts as one iteration **against that scope**. On
overflow the command **escalates to the human** with the scope, phase, failing verdict
+ reasons, and the last routed author; it does not loop forever and does not advance
status on an unresolved scope.

### 6.3 Routing (MD-as-authority)
A red test never patches code arbitrarily. The `test-gatekeeper` triages each failure:

| Cause | Meaning | Route | Then |
|---|---|---|---|
| **spec bug** | the spec is wrong/incomplete | `spec-writer` | demote → re-analysis-gate → re-implement (code gate) → re-test |
| **code bug** | code ≠ a correct spec | `code-implementer` | re-implement → code gate → re-test |
| **test bug** | the test asserts the wrong thing | `test-writer` | re-author the test → re-run |

### 6.4 Re-gate consistency (invariant)
**Every regenerated artifact re-passes its own gate before the next phase.** A changed
spec re-passes the analysis gate; regenerated code re-passes the code gate; a changed
test is re-run and re-judged. This holds identically in `/sdd-implement`, `/sdd-test`,
and `/sdd-auto`.

### 6.5 Status demotion
A spec-bug route (any phase) or a deliberate feature evolution demotes the entity
`reviewed → draft` (or `approved → draft`) — performed by the command — so it re-earns
`reviewed` (and then `approved`) through the gates. Code/test bugs never demote.

---

## 7. Isolation & trust model

Two kinds of isolation, both real but enforced differently.

**A) Context isolation (all agents).** Agents communicate **only through files**. No
agent depends on another agent's conversational memory. This makes hand-offs
auditable, token-cheap, and resumable.

**B) Test-writer independence (soft, by design — no runtime hook).** Tests must be an
independent oracle, so the `test-writer` must not see the implementation. Enforced by
three soft layers: (1) **minimal tools** — `Read, Write, Glob`, no `Bash`; (2) an
explicit **NON-GOAL** — never read `src/` or `.sdd/impl-notes/`; (3) the
**test-gatekeeper** rejects tests that assert implementation detail. (An earlier
`PreToolUse` hook + lock file was deliberately dropped as fragile.)

**The `code-implementer` is NOT isolated** — it reads/edits `src/`. It is kept honest
by **spec-as-authority + the code-gatekeeper + the independent test suite**, not by
read-blindness. Even a prompt-injection in a spec is caught by the gatekeeper + the
independent tests.

### 7.1 Isolation matrix

| Agent | May READ | May WRITE | Bash/Edit | `model` | Reads `src/`? |
|---|---|---|---|---|---|
| `plan-architect` | `requirements/`, existing `specs/`, `.claude/sdd/`, `.sdd/target.md` | `plan/`, `.sdd/target.md` | — | `opus` | no |
| `plan-gatekeeper` | `requirements/`, `plan/`, `specs/`, `.claude/sdd/`, `.sdd/target.md` | `.sdd/state.md` | — | `opus` | no |
| `spec-writer` | `plan/`, `requirements/`, `specs/`, `.claude/sdd/{conventions,scot,ui-schema}.md`, `.sdd/target.md`, `.claude/sdd/templates/` | `specs/` (incl. indexes) | — | `opus` | no |
| `reuse-analyst` | `specs/`, `.claude/sdd/{conventions,ui-schema}.md` | `specs/` | — | `opus` | no |
| `analysis-gatekeeper` | `specs/`, `requirements/`, `.claude/sdd/`, `.sdd/` | `.sdd/state.md` | — | `opus` | no |
| `code-implementer` | `specs/`, `.claude/sdd/{conventions,scot,ui-schema}.md`, `.sdd/target.md`, `.sdd/impl-notes/`, `src/` | `src/` (declared paths), `.sdd/impl-notes/<id>.md` | **Edit** | `opus` | **yes** — spec stays authority |
| `code-gatekeeper` | `specs/`, `.sdd/impl-notes/`, `src/`, `.claude/sdd/`, `.sdd/target.md` | `.sdd/state.md` | Bash (read-only) | `opus` | yes (review) |
| `test-writer` | behavioral spec sections, `.claude/sdd/{conventions,scot,ui-schema}.md`, `.sdd/target.md` | `tests/` | — | `sonnet` | **no — by role** |
| `test-runner` | `tests/`, `src/`, `.claude/sdd/conventions.md`, `.sdd/target.md` | `tests/REPORT.md` | Bash (run) | `sonnet` | yes |
| `test-gatekeeper` | `tests/REPORT.md`, `specs/`, `src/`, `tests/`, `.claude/sdd/`, `.sdd/` | `.sdd/state.md` | — | `opus` | yes |
| *commands* (main session) | everything | index `status`, drives all | Task | *(session)* | no (coordinates) |

**Model split.** `opus` for under-specified **authoring** (`plan-architect`,
`spec-writer`, `reuse-analyst`, `code-implementer`) and the **verification gates**
(`plan-gatekeeper` graph/DAG + coverage validation, `analysis-gatekeeper` the spec
blocker, `code-gatekeeper` the semantic code↔spec equivalence check, `test-gatekeeper`
the spec/code/test triage); `sonnet` for the two **purely mechanical** roles
(`test-writer` coverage enumeration, `test-runner` execution + reporting). The
pipeline is reasoning-heavy, so the split is opus-leaning by design; a project may
override any agent's `model:`. Canonical in
`.claude/sdd/conventions.md` §9; a project may override any agent's `model:`.

---

## 8. End-to-end walkthroughs

### 8.1 New feature, manual path

Requirement: *"Register a user with a unique email and a strong password"* (stack: TS,
modular, Express + React, pnpm, Postgres).

1. **`/sdd-plan "...stack..."`** → the command writes `requirements/REQUIREMENT.md`
   with `REQ-001`; `plan-architect` writes `.sdd/target.md` and `plan/PLAN.md`
   (modules `MOD-db`/`MOD-api`/`MOD-web`/`MOD-build`; entity `ENT-user`; classes
   `CLS-userRepo`/`CLS-regCtrl`/`CLS-registerScreen`; shared `SHR-passwordHasher`);
   `plan-gatekeeper` PASS → ready for approval.
2. **`/sdd-specify`** → human approves; `spec-writer` writes the indexes + specs at
   `draft`; `reuse-analyst` promotes the recurring widgets into `specs/ui-components/`
   (`COMP-textInput`, `COMP-formField`, `COMP-button`) and rewires the screen to
   reference them by id; `analysis-gatekeeper` PASS → the command advances every row to
   `reviewed`.
3. **`/sdd-implement`** → `code-implementer` generates, in `depends_on` order,
   `ENT-user` → `CLS-userRepo` (interface) → `SHR-passwordHasher` → `CLS-regCtrl` →
   `CLS-registerScreen`, plus `MOD-build`'s users-table schema change; each file carries
   its `// spec:` header; concretizations (argon2, the ORM call, the Result mapping)
   go into `.sdd/impl-notes/`; `code-gatekeeper` PASS.
4. **`/sdd-test`** → `test-writer` derives the suite from the behavioral specs
   (unit per branch arm + per `ACn`, integration from `FEAT-001`, constraints from
   `ENT-user`); `test-runner` runs it → `tests/REPORT.md`; `test-gatekeeper` confirms
   full coverage + green → the command advances every row `reviewed → approved`.

`/sdd-status` at any point shows the dashboard; `/sdd-trace FEAT-001` shows the chain.

### 8.2 Automatic path
**`/sdd-auto "...requirement + stack..."`** runs the same four phases unattended, **one
vertical slice at a time** in `depends_on` order, advancing status from each verdict
and escalating only on a budget overflow (or the stack question if unresolvable). If it
is interrupted, re-running it resumes from `state.md` + index `status`.

### 8.3 A test-phase failure (triage in action)
`test-runner` reports `CLS-regCtrl::register#B1.then` failing (a duplicate email returns
500, not `EmailAlreadyTaken`). `test-gatekeeper` triages: the spec's `AC2` and SCoT arm
are correct, the code diverges → **code bug** → `code-implementer` applies a minimal
diff → code gate re-passes → tests re-run green → `approved`. Had the spec's `AC2` been
wrong, it would route **spec bug** → demote `reviewed → draft` → re-analysis-gate →
re-implement (code gate) → re-test.

### 8.4 Feature evolution (already `approved`)
To change `ENT-user` (add a `displayName`): `/sdd-specify ENT-user` demotes the touched
entities `approved → draft`, the `spec-writer` edits the entity spec, the analysis gate
re-passes (`reviewed`), `/sdd-implement` applies a minimal diff to the entity + the
`MOD-build` schema change (code gate), `/sdd-test` re-derives constraint tests and
re-approves. The blast radius is the touched entity, not the project.

---

## 9. Supervision & traceability

- **`/sdd-status [scope]`** — the live dashboard: every entity's `status`, last
  verdict, what is blocked and on whom, and the `/sdd-auto` resume point. Use it to
  decide the next manual step, to unblock, or to resume an interrupted auto run.
- **`/sdd-trace [id]`** — the traceability chain REQ→FEAT→CLS/ENT/COMP→SOURCE→TEST,
  with AC/branch coverage gaps, orphan nodes, and source-mapping drift.
- Both are **read-only** and invoke no subagent — safe to run anytime.

---

## 10. Operating the workflow

### 10.1 Adopting the tool
Drop the **`.claude/`** folder into your project (copy it, or `ln -s` it from this
repo) — that one folder *is* the tool: agents, commands, contracts, templates, and the
guaranteed UI-library definition (`.claude/sdd/ui-schema.md` §9). The only thing you
supply is the requirement (with the stack); everything else (`.sdd/`, `specs/`, `src/`,
`tests/`) the workflow creates inside your project.

### 10.2 The stack question
The stack comes from your prompt; the AI writes it into `.sdd/target.md`. **If the
stack is unstated, the AI asks once** before writing `target.md` — it never assumes a
default. `/sdd-auto` is the one automatic command that still asks this one question if
it cannot infer the stack.

### 10.3 Multi-module scaling
For a large multi-module project, the plan may split the fine-grained indexes
(`classes`, `model`, `ui-components`) per module under
`specs/indexes/<level>/<MOD-id>.index.md`, keeping `modules` and `features` global.

### 10.4 Configuring budgets
Override the analysis/code/test budgets in `.sdd/target.md §4`. Overflow always
escalates to the human.

---

## 11. Extending the workflow

- **Add a UI component** — the **baseline is guaranteed automatically** (the
  `spec-writer` materializes `.claude/sdd/ui-schema.md` §9); for anything beyond it let the
  `reuse-analyst` promote it (the normal path), or author a `COMP-*` spec in
  `specs/ui-components/` (per `ui-schema §6`, with a `layer`) and register it in
  `ui-components.index.md`. Screens then reference it by id.
- **Add a spec kind** — add the row to `conventions.md §3` (`kind:`→form), define its
  required sections, add a template under `.claude/sdd/templates/`, and teach the
  `spec-writer` (body form) and `analysis-gatekeeper` (sections) about it. (This is
  exactly how `module` and the `COMP-*` exception were added.)
- **Non-functional requirements** — today NFRs enter as testable `ACn` (so the test
  gate enforces them) or as `target.md §5` context notes. A dedicated NFR convention
  (ids + a gate) is a candidate extension.
- **Add an agent** — only if a genuinely new responsibility appears (e.g.
  reverse-engineering legacy code into specs — currently out of scope). Give it a
  `ROLE/MISSION/MINDSET/NON-GOALS` block, the minimum tools, a file-only contract, and
  a row in the isolation matrix.

When extending, keep the invariants: file-only hand-offs, gatekeepers judge / authors
write / commands advance status, and the two cross-cutting values in every role.

---

## 12. Design rationale

- **Multi-level indexes** → lazy loading of specs → fewer tokens.
- **SCoT for behavior, declarative for entities/structural, schematic for UI** → each
  thing in the right form, faithfully translatable to any stack.
- **Impl-notes outside the spec** → the gated spec stays clean and regenerable;
  concretization has its own home.
- **Edit by default, regenerate by exception** → minimal blast radius; full
  regeneration remains the safety net.
- **Tests as the equivalence oracle (enforced coverage)** → "code ≡ spec" is verified
  behaviorally, never by a non-deterministic textual diff.
- **Author/judge separation + iteration budgets + escalation** → robust loops, no
  infinite loops, a human backstop.
- **Commands orchestrate (no orchestrator subagent)** → matches Claude Code's real
  capability (no reliable nested subagent spawning).
- **One canonical coverage-id notation** → mechanical, unambiguous coverage matching.

---

## 13. Glossary

- **SDD** — Spec-Driven Development: you start from the specs; the code is derived.
- **SCoT** — Structured Chain-of-Thought: the structured pseudo-code (sequence/branch/
  loop) used in behavioral specs.
- **Gatekeeper** — a reviewer agent with veto power over a phase; judges only.
- **Author** — an agent that writes artifacts (specs/code/tests/impl-notes).
- **Impl-notes** — concretization (library, API, idiom, edge case) the SCoT omits;
  owned by the implementer; never read by the test-writer.
- **Coverage id** — the canonical join key `<spec-id>::<function>#<arm-id>` /
  `<spec-id>#ACn` linking a test to the spec unit it covers.
- **Vertical slice** — a feature/module plus its `depends_on` closure, processed
  end-to-end to bound context.
- **Resume point** — the first non-`approved` slice (and its sub-phase) where
  `/sdd-auto` would continue, reconstructable from `state.md` + index `status`.
- **Stub** — an empty implementation auto-derived from an `interface` spec for a
  not-yet-ready dependency or for tests.
```
