# Spec-Driven Development (SDD) — an agentic workflow for Claude Code

A set of **subagents** (`.claude/agents/`) and **slash commands** (`.claude/commands/`)
that turn Markdown specifications into working, tested code — and keep the two in
sync — for projects of **any stack** (Java, Kotlin, C#/.NET, Node/TS, Python, Go +
React/Angular/Vue/Svelte, mono/modular/microservice/CLI/library).

> **Founding principle — Markdown is the source of truth (as AUTHORITY).**
> Specs decide WHAT the system does and how it is structured. In any conflict, the
> **spec wins and the code is corrected** — never the reverse. Code is a *derived*
> artifact. This is authority, not blindness: the code agent **may** read and edit
> source to apply minimal diffs; every change is verified against the spec by a
> gatekeeper; and the **independent, spec-derived test suite** is the proof that
> code still honors the spec (behavioral equivalence — not a textual diff).

The workflow is **spec-first**: a brand-new project, or a feature added to a
project already built with this workflow (so its specs exist). Reverse-engineering
a legacy, non-spec'd codebase is **out of scope**.

---

## The two values that bind every agent

1. **Markdown is the source of truth (authority).**
2. **Reuse over repetition (DRY)** — discover-before-create, extract shared
   abstractions, and (for the frontend) compose a **UI component library** rather
   than inlining widgets.

Both appear, verbatim, in the `MINDSET` line of every agent's ROLE prompt.

---

## Repository layout

This repository **is the tool** — the workflow that builds applications, not an
application built with it. It therefore ships only the tool's content; the
per-project artifacts (`requirements/`, `plan/`, your generated `specs/`, `src/`,
`tests/`) are created by the workflow **inside your project** when you run it.

**Ships in this repo (the tool):**

```
.claude/
  agents/*.md                        # the 10 subagents
  commands/*.md                      # the 7 slash commands
.sdd/
  conventions.md                     # the single canonical reference (ids, front-matter, status, verdicts, rosters)
  scot.md                            # canonical SCoT grammar (behavioral specs)
  ui-schema.md                       # canonical UI-schematic convention — incl. the GUARANTEED baseline UI library (§9)
  templates/*.template.md            # the forms new specs are copied from (each with a filled example)
  target.md                          # per-project stack file — ships as a TEMPLATE the AI fills at plan time
  state.md                           # append-only verdict log — ships as an empty seed
  impl-notes/_TEMPLATE.md            # impl-notes template
docs/
  TECHNICAL.md                       # the detailed technical reference
  history/                           # the original design conversation (archived)
README.md
```

**Created by the workflow at runtime, inside your project (not shipped here):**

```
requirements/REQUIREMENT.md          # the requirement (raw + refined, with REQ-* ids) — written by /sdd-plan
plan/PLAN.md                         # the plan — written by plan-architect
.sdd/target.md (filled) · .sdd/state.md (appended) · .sdd/impl-notes/<id>.md
specs/indexes/{modules,features,model,classes,ui-components}.index.md   # your per-level indexes
specs/{modules,features,model,classes,shared}/<id>.spec.md   # your generated specs
specs/ui-components/<id>.spec.md     # the UI library — baseline GUARANTEED for a GUI project (ui-schema §9), then enriched
src/   tests/                        # GENERATED code & tests (derived)
```

`.sdd/conventions.md` is the **single canonical reference** (ids, front-matter
schema, status lifecycle, verdict format, the agent/command rosters, the ROLE
template). It is the first file every agent reads, so the conventions are stated
**once** instead of duplicated across eleven prompts — DRY applied to the workflow
itself. It defers to its two siblings: `.sdd/scot.md` (behavior) and
`.sdd/ui-schema.md` (UI form).

---

## The four spec levels

Each "thing" is specified in the form that fits it (chosen by a `kind:` field):

| Level | `kind:` | Form | Yields tests |
|---|---|---|---|
| **Feature / use-case** | `use-case` | orchestration **SCoT** (cross-class calls, by id) + integration ACs | integration / acceptance |
| **Entity / data-model** | `entity` | **declarative**: fields, types, relations, invariants | constraint / validation |
| **Class / service** | `service`, `controller` | **per-method SCoT** + invariants | unit |
| **UI component / screen** | `gui` | **UI schematic** (wireframe + component tree + state/events) | UI / interaction |

Structural artifacts (`dto`, `enum`, `interface`, `config`) are **declarative**;
an `interface` lists signatures only (the SCoT lives in the implementing class),
and a **stub/mock is auto-derived from its interface** — never specced separately.

- **SCoT** (`.sdd/scot.md`) — keyword/block pseudo-code (sequence / branch / loop,
  `async`, `try`), explicit I/O, and a **stable branch id** (`[B1]`, arm `B1.then`)
  on every decision so coverage is mechanical: the test-writer must produce ≥1 test
  per branch arm and per acceptance criterion (`ACn`).
- **UI schematic** (`.sdd/ui-schema.md`) — the framework-agnostic source form of a
  UI: ASCII wireframe, a component tree that **references library components by
  id**, and state/events tables. Atomic-design layers (atom → molecule → organism
  → layout). The concrete framework is applied at implementation time.

Concretization the SCoT omits (library, API binding, idiom, edge case, bug-fix
lesson) goes into `.sdd/impl-notes/<id>.md` — **never** into the gated spec — so
the spec stays clean and **regenerable**.

---

## Indexes & traceability

- **One index per level** (not per module). Each row: `id · name · one-line WHAT ·
  module · depends_on · spec path · source · status`. Agents read the index first
  and open only the specs they need (lazy loading → fewer tokens). Shared non-UI
  abstractions (`SHR-*`) are implementation units, so they are listed in
  `classes.index.md` — there is no separate shared index.
- The **spec → source** mapping's single authoritative home is each spec's
  `source:` front-matter; the index `source` column is **derived** from it. Each
  generated file carries a header back to its spec (`// spec: CLS-… — specs/…`).
- **Traceability** (REQUIREMENT → FEATURE → CLASS → SOURCE → TEST) is **not** a
  separate file — it is reconstructed on demand by `/sdd-trace` from the indexes +
  front-matter + headers + test coverage ids.
- Per-entity **`status`** (`draft → reviewed → approved`) lives in the index row.
  Gatekeepers only **judge** (verdict → `.sdd/state.md`); the **command** advances
  the status from the verdict. So `state.md` records *why*; the index records
  *where each entity stands*.

---

## The agents (10 subagents)

| Agent | Does | Reads `src/`? |
|---|---|---|
| `plan-architect` | requirement → PLAN of indexes/specs; derives the stack → `.sdd/target.md` | no |
| `plan-gatekeeper` | judges the plan (veto) | no |
| `spec-writer` | writes indexes + specs (4 levels + `MOD-build`) | no |
| `reuse-analyst` | dedupes & promotes shared specs/components (pure author) | no |
| `analysis-gatekeeper` | the single blocker of the spec phase (veto) | no |
| `code-implementer` | specs → source: create new files **or minimal diffs**; writes impl-notes | yes (edit) |
| `code-gatekeeper` | verifies code ≡ spec & mapping (veto) | yes (review) |
| `test-writer` | specs → tests, an **independent oracle** (≥1 per AC & per branch) | **no (by role)** |
| `test-runner` | runs the suite, writes `tests/REPORT.md` | yes |
| `test-gatekeeper` | verifies coverage + triages failures (spec/code/test) | yes |

**Gatekeepers judge; authors write.** Feedback loops have an **iteration budget**
(analysis 3, code 3, test 5 — configurable in `.sdd/target.md`); on overflow they
**escalate to the human** instead of looping forever. Routing keeps MD authoritative:
a red test never makes code the source of truth — either the code is fixed to the
spec, or the spec is fixed first and the code regenerated.

> **No `orchestrator` subagent.** Claude Code subagents cannot reliably spawn other
> subagents, so **orchestration lives in the slash commands** run by the main
> session (the only level that reliably invokes subagents via `Task`). Subagents are
> single-purpose, file-in/file-out workers; the command drives each loop: invoke
> agent → read the verdict from `.sdd/state.md` → decide (proceed / re-invoke the
> routed author / escalate) → advance index `status`.

---

## Isolation matrix

Two kinds of isolation hold. **(A) Context isolation** between *all* agents: they
communicate **only through files** (specs, indexes, `.sdd/state.md`, impl-notes),
never through shared conversational memory. **(B) Test-writer independence** is
**soft by design** — no runtime hook — enforced by minimal tools (no `Bash`), an
explicit NON-GOAL, and the test-gatekeeper. The **code-implementer is not isolated**;
it is kept honest by spec-as-authority + the code-gatekeeper + the independent test
suite.

| Agent | May READ | May WRITE | Bash / Edit | Reads `src/`? |
|---|---|---|---|---|
| `plan-architect` | `requirements/`, existing `specs/`, `.sdd/` | `plan/`, `.sdd/target.md` (+ `scot`/`ui-schema` if absent) | — | no |
| `plan-gatekeeper` | `requirements/`, `plan/`, `specs/`, `.sdd/` | `.sdd/state.md` | — | no |
| `spec-writer` | `plan/`, `requirements/`, `specs/`, `.sdd/{scot,ui-schema,conventions,target}.md`, `.sdd/templates/` | `specs/` (incl. indexes) | — | no |
| `reuse-analyst` | `specs/`, `.sdd/conventions.md` | `specs/` | — | no |
| `analysis-gatekeeper` | `specs/`, `requirements/`, `.sdd/` | `.sdd/state.md` | — | no |
| `code-implementer` | `specs/`, `.sdd/{target,scot,ui-schema,conventions}.md`, `.sdd/impl-notes/`, `src/` | `src/` (declared paths), `.sdd/impl-notes/<id>.md` | **Edit** (minimal diffs) | **yes** — spec stays authority |
| `code-gatekeeper` | `specs/`, `.sdd/impl-notes/`, `src/`, `.sdd/` | `.sdd/state.md` | Bash (read-only) | yes (review) |
| `test-writer` | behavioral spec sections, `.sdd/{target,scot,ui-schema,conventions}.md` | `tests/` | — | **no — by role** (not impl-notes either) |
| `test-runner` | `tests/`, `src/`, `.sdd/target.md` | `tests/REPORT.md` | Bash (run tests) | yes |
| `test-gatekeeper` | `tests/REPORT.md`, `specs/`, `src/`, `tests/`, `.sdd/` | `.sdd/state.md` | — | yes |

Orchestration is **not** an agent row — it is the commands (main session), which
read verdicts and advance index `status`.

---

## The five modes (slash commands)

Modes 1–4 are the **manual, human-in-control** path; mode 5 is **automatic, human
out of the loop**. Two read-only helpers — `/sdd-trace` (traceability) and
`/sdd-status` (a lifecycle dashboard with the `/sdd-auto` resume point) — assist on
demand.

| Command | Mode | What it runs |
|---|---|---|
| `/sdd-plan` | 1. Plan | `plan-architect` → `plan-gatekeeper`. Derives the stack into `.sdd/target.md` (asks you if it isn't stated). **No specs or code.** |
| `/sdd-specify` | 2. Plan approval | you approve the plan → `spec-writer` → `reuse-analyst` → `analysis-gatekeeper`. Advances `draft → reviewed`. |
| `/sdd-implement` | 3. Implement | `code-implementer` → `code-gatekeeper`, in `depends_on` order (new files or minimal diffs). |
| `/sdd-test` | 4. Test run | `test-writer` → `test-runner` → `test-gatekeeper` (coverage + triage). On full green: `reviewed → approved`. |
| `/sdd-auto` | 5. Automatic | all of the above end-to-end, **one vertical slice at a time** in `depends_on` order, no manual stop; escalates only on budget overflow. |
| `/sdd-trace` | helper | prints REQ → FEAT → CLASS → SOURCE → TEST. Read-only. |
| `/sdd-status` | helper | prints a lifecycle dashboard (status + latest verdict + resume point). Read-only. |

### How the stack is chosen
The technologies come from **your prompt**. The AI reads it and writes them into
`.sdd/target.md` (language, architecture, UI framework, build tool, conventions,
canonical commands). **If the stack isn't stated, the AI asks you** before writing
it — it never assumes a default. For an existing SDD project, `.sdd/target.md`
already reflects the stack and your prompt may only extend/override it.

---

## Quick start

**Adopting the workflow:** start a project from this repo, or copy `.claude/` and
`.sdd/` into it — those are the agents/commands plus all the contracts, templates,
and the guaranteed UI-library definition (`.sdd/ui-schema.md` §9). The requirement
(with the **stack**) is the only thing you must supply; if the stack isn't stated,
the workflow asks once before writing `.sdd/target.md`.

1. `/sdd-plan <describe what to build, and the stack>` — review `plan/PLAN.md` and
   `.sdd/target.md`.
2. `/sdd-specify` — approve the plan; review the generated `specs/`.
3. `/sdd-implement` — generate/patch `src/`; the code gate checks it.
4. `/sdd-test` — generate & run tests; the test gate checks coverage and triages.

…or run `/sdd-auto <describe what to build>` to do all four automatically, slice by
slice.

---

## Reuse-first frontend (guaranteed for every GUI project)

For any project with a GUI, the workflow **guarantees** a default UI component
library — `COMP-appShell`, `COMP-header`, `COMP-body`, `COMP-footer`, `COMP-panel`,
and layout helpers (`COMP-stack`, `COMP-grid`, `COMP-section`). It is **defined** in
`.sdd/ui-schema.md` §9 and **materialized** by the `spec-writer` into the project's
`specs/ui-components/` (with its index) before any screen is specified — it is **not
shipped as files**. Screens **compose these by id**; the `reuse-analyst` progressively
promotes recurring widgets (Button, TextInput, FormField, Modal, Table, …) into the
library so nothing is copy-pasted.

---

## What I changed from the starting proposal (and why)

- **Added `.sdd/conventions.md`** — a single canonical reference for ids,
  front-matter, status, verdict format, and the rosters. The agents were each going
  to restate these; centralizing them is the workflow practicing its own DRY value
  and guarantees the eleven prompts can't drift.
- **Orchestrator is a command, not a subagent** — because nested subagent spawning
  is unreliable in Claude Code. The matrix's `orchestrator` row becomes the
  commands run by the main session.
- **No runtime hook** for test-writer isolation — it is soft (role + minimal tools
  + the test gate), matching the design's "avoid fragile blocks" direction.
- **Stable `[Bn]` branch ids and enumerated arms** in SCoT, so "every branch has a
  test" is checkable mechanically rather than by judgment.
