# Spec-Driven Development (SDD) — an agentic workflow for Claude Code

A set of **subagents** (`.claude/agents/`) and a single **slash command**
(`.claude/commands/sdd-auto.md`) that turn Markdown specifications into working, tested
code — and keep the two in sync — for projects of **any stack** (Java, Kotlin,
C#/.NET, Node/TS, Python, Go + React/Angular/Vue/Svelte, mono/modular/microservice/
CLI/library).

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
.claude/                             # ← THE WHOLE TOOL: one portable, immutable folder
  agents/*.md                        # the 11 subagents
  commands/sdd-auto.md               # the single orchestrator command (/sdd-auto)
  sdd/
    conventions.md                   # the single canonical reference (ids, front-matter, status, verdicts, rosters)
    scot.md                          # canonical SCoT grammar (behavioral specs)
    ui-schema.md                     # canonical UI-schematic convention — incl. the GUARANTEED baseline UI library (§9)
    templates/*.template.md          # the forms new specs are copied from (incl. target.template, impl-notes.template)
    workflow.mmd                     # the pipeline as a Mermaid flowchart
docs/
  TECHNICAL.md                       # the detailed technical reference
  history/                           # the original design conversation (archived)
README.md
```

**Created by the workflow at runtime, inside your project (not shipped here):**

```
requirements/REQUIREMENT.md          # the requirement (raw + refined, with REQ-* ids) — written by requirement-analyst
plan/PLAN.md                         # the plan + ordered slice list — written by plan-architect
.sdd/target.md · .sdd/state.md · .sdd/impl-notes/<id>.md   # the project's SDD state
specs/indexes/{modules,features,model,classes,ui-components}.index.md   # your per-level indexes
specs/{modules,features,model,classes,shared}/<id>.spec.md   # your generated specs
specs/ui-components/<id>.spec.md     # the UI library — baseline GUARANTEED for a GUI project (ui-schema §9), then enriched
src/   tests/                        # GENERATED code & tests (derived)
```

`.claude/sdd/conventions.md` is the **single canonical reference** (ids, front-matter
schema, status lifecycle, verdict format, the agent/command rosters, the ROLE
template). It is the first file every agent reads, so the conventions are stated
**once** instead of duplicated across eleven prompts — DRY applied to the workflow
itself. It defers to its two siblings: `.claude/sdd/scot.md` (behavior) and
`.claude/sdd/ui-schema.md` (UI form).

---

## The four spec levels

Each "thing" is specified in the form that fits it (chosen by a `kind:` field):

| Level | `kind:` | Form | Yields tests |
|---|---|---|---|
| **Feature / use-case** | `use-case` | orchestration **SCoT** (cross-class calls, by id) + integration ACs | integration / acceptance |
| **Entity / data-model** | `entity` | **declarative**: fields, types, relations, invariants | constraint / validation |
| **Class / service** | `service`, `controller` | **per-method SCoT** + invariants | unit |
| **UI component / screen** | `gui` | **UI schematic** (wireframe + component tree + state/events) | component (in-process) + **e2e** (Playwright, for `(journey)` ACs) |

Structural artifacts (`dto`, `enum`, `interface`, `config`) are **declarative**;
an `interface` lists signatures only (the SCoT lives in the implementing class),
and a **stub/mock is auto-derived from its interface** — never specced separately.

- **SCoT** (`.claude/sdd/scot.md`) — keyword/block pseudo-code (sequence / branch / loop,
  `async`, `try`), explicit I/O, and a **stable branch id** (`[B1]`, arm `B1.then`)
  on every decision so coverage is mechanical: the test-writer must produce ≥1 test
  per branch arm and per acceptance criterion (`ACn`).
- **UI schematic** (`.claude/sdd/ui-schema.md`) — the framework-agnostic source form of a
  UI: ASCII wireframe, a component tree that **references library components by
  id**, and state/events tables. Atomic-design layers (atom → molecule → organism
  → layout). The concrete framework is applied at implementation time.

Concretization the SCoT omits (library, API binding, idiom, edge case, bug-fix
lesson) goes into `.sdd/impl-notes/<id>.md` — **never** into the gated spec — so
the spec stays clean and **regenerable**.

---

## How tests work

Five test types, all **derived from the specs** (the test-writer is an independent
oracle — it never reads `src/`):

- **unit** ← class SCoT (one per branch arm, collaborators stubbed) · **integration**
  ← feature SCoT (real in-process collaboration, infrastructure mocked) · **constraint**
  ← entity invariants · **component** ← gui specs (rendered in a test renderer, the
  feature mocked) · **e2e (Playwright)** ← gui screens, **GUI projects only** — drives
  the **real running app in a real browser** for each AC tagged `(journey)` (the
  primary success journey + each rendered failure journey), selecting elements by
  accessible role/name from the spec.
- **Scoped during the workflow.** Each canonical `test-*` command in `.sdd/target.md`
  carries a `{scope}` selector the `test-runner` fills, so only the **scope under work**
  runs — never the whole app. The final whole-project run is unscoped and catches
  cross-scope regressions.
- **Compact output.** The `test-*` commands bake a **dot** console reporter (tiny
  captured output) alongside a machine-readable JUnit/JSON file the runner parses.

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
  separate file — it is reconstructed on demand from the indexes + front-matter +
  source headers + test coverage ids.
- Per-entity **`status`** (`draft → reviewed → approved`) lives in the index row.
  Gatekeepers only **judge** (verdict → `.sdd/state.md`); the **command** advances
  the status from the verdict. So `state.md` records *why*; the index records
  *where each entity stands*.

---

## The agents (11 subagents)

| Agent | Does | Reads `src/`? |
|---|---|---|
| `requirement-analyst` | raw request → `requirements/REQUIREMENT.md` (atomic, testable `REQ-*`) | no |
| `plan-architect` | requirement → PLAN of indexes/specs + ordered slices; derives the stack → `.sdd/target.md` | no |
| `plan-gatekeeper` | judges the plan (veto) | no |
| `spec-writer` | writes indexes + specs (4 levels + `MOD-build`) | no |
| `reuse-analyst` | dedupes & promotes shared specs/components (pure author) | no |
| `analysis-gatekeeper` | the single blocker of the spec phase (veto) | no |
| `code-implementer` | specs → source: create new files **or minimal diffs**; writes impl-notes | yes (edit) |
| `code-gatekeeper` | verifies code ≡ spec & mapping (veto) | yes (review) |
| `test-writer` | specs → tests, an **independent oracle** (≥1 per AC & per branch; + Playwright e2e per `(journey)` AC for GUI projects) | **no (by role)** |
| `test-runner` | runs the suite **scoped to the work** (`{scope}`), writes `tests/REPORT.md` | yes |
| `test-gatekeeper` | verifies coverage + triages failures (spec/code/test) | yes |

**Gatekeepers judge; authors write.** Feedback loops have an **iteration budget**
(analysis 3, code 3, test 5 — configurable in `.sdd/target.md`); on overflow they
**escalate to the human** instead of looping forever. Routing keeps MD authoritative:
a red test never makes code the source of truth — either the code is fixed to the
spec, or the spec is fixed first and the code regenerated.

> **No `orchestrator` subagent.** Claude Code subagents cannot reliably spawn other
> subagents, so **orchestration lives in the `/sdd-auto` slash command** run by the
> main session (the only level that reliably invokes subagents via `Task`). Subagents
> are single-purpose, file-in/file-out workers; the command drives each loop: invoke
> agent → read the verdict from `.sdd/state.md` → decide (proceed / re-invoke the
> routed author / escalate) → advance index `status`. The eleven subagents + the
> orchestrating command = the **twelve roles** of the system.

---

## Isolation matrix

Two kinds of isolation hold. **(A) Context isolation** between *all* agents: they
communicate **only through files** (specs, indexes, `.sdd/state.md`, impl-notes),
never through shared conversational memory. **(B) Test-writer independence** is
**soft by design** — no runtime hook — enforced by minimal tools (no `Bash`, no
`Grep`), an explicit NON-GOAL, and the test-gatekeeper. The **code-implementer is not
isolated**; it is kept honest by spec-as-authority + the code-gatekeeper + the
independent test suite.

| Agent | May READ | May WRITE | Bash / Edit | `model` | Reads `src/`? |
|---|---|---|---|---|---|
| `requirement-analyst` | `$ARGUMENTS`, existing `requirements/`, `.claude/sdd/conventions.md` | `requirements/` | Edit | `opus` | no |
| `plan-architect` | `requirements/`, existing `specs/`, `.claude/sdd/`, `.sdd/target.md` | `plan/`, `.sdd/target.md` | Edit | `opus` | no |
| `plan-gatekeeper` | `requirements/`, `plan/`, `specs/`, `.claude/sdd/`, `.sdd/target.md` | `.sdd/state.md` | — | `opus` | no |
| `spec-writer` | `plan/`, `requirements/`, `specs/`, `.claude/sdd/{conventions,scot,ui-schema}.md`, `.sdd/target.md`, `.claude/sdd/templates/` | `specs/` (incl. indexes) | Edit | `opus` | no |
| `reuse-analyst` | `specs/`, `.claude/sdd/{conventions,ui-schema}.md`, `.sdd/target.md` | `specs/` | Edit | `opus` | no |
| `analysis-gatekeeper` | `specs/`, `requirements/`, `.claude/sdd/`, `.sdd/` | `.sdd/state.md` | — | `opus` | no |
| `code-implementer` | `specs/`, `.claude/sdd/{conventions,scot,ui-schema}.md`, `.sdd/target.md`, `.sdd/impl-notes/`, `src/` | `src/` (declared paths), `.sdd/impl-notes/<id>.md` | **Edit** (minimal diffs) | `opus` | **yes** — spec stays authority |
| `code-gatekeeper` | `specs/`, `.sdd/impl-notes/`, `src/`, `.claude/sdd/`, `.sdd/target.md` | `.sdd/state.md` | Bash (read-only) | `opus` | yes (review) |
| `test-writer` | behavioral spec sections, `.claude/sdd/{conventions,scot,ui-schema}.md`, `.sdd/target.md` | `tests/` | Edit | `sonnet` | **no — by role** (not impl-notes either) |
| `test-runner` | `tests/`, `src/`, `.sdd/target.md` | `tests/REPORT.md` | Bash (run tests) | `sonnet` | yes |
| `test-gatekeeper` | `tests/REPORT.md`, `specs/`, `src/`, `tests/`, `.claude/sdd/`, `.sdd/` | `.sdd/state.md` | — | `opus` | yes |

Orchestration is **not** an agent row — it is the `/sdd-auto` command (main session),
which reads verdicts and advances index `status`.

---

## The command (slash command)

A **single command** runs the whole flow end-to-end, **human out of the loop**, **one
vertical slice at a time** in `depends_on` order. It escalates only on an
iteration-budget overflow (or the one stack question, if the stack is unresolvable).

| Command | What it runs |
|---|---|
| `/sdd-auto <requirement + stack>` | Plan → Specify → Implement → Test end-to-end, with the automatic gates and feedback loops, slice by slice; the main session orchestrates and never authors an artifact. |

The flow (the main session orchestrates; the `▶▶` agent does the work):

1. **Resolve the stack** — from `.sdd/target.md`, else infer from the requirement, else **ask the human once**.
2. **Capture the requirement** ▶▶ `requirement-analyst` → `requirements/REQUIREMENT.md` (`REQ-*`).
3. **Plan** ▶▶ `plan-architect` → `.sdd/target.md` + `plan/PLAN.md` (incl. the ordered slice list).
4. **Gate the plan** ▶▶ `plan-gatekeeper`.
   *Then, per slice, in order:*
5. **Specify** ▶▶ `spec-writer` → `reuse-analyst` → `analysis-gatekeeper` *(on PASS: `draft → reviewed`)*.
6. **Implement** ▶▶ `code-implementer` → `code-gatekeeper` *(writes `src/` + impl-notes)*.
7. **Test** ▶▶ `test-writer` → `test-runner` → `test-gatekeeper` *(on PASS: `reviewed → approved`)*.
8. **Next slice** — repeat 5–7 until every slice is `approved`.
9. **Final unscoped sweep** ▶▶ `test-runner` → `test-gatekeeper` over the whole project, to catch cross-slice regressions.

**The human is touched twice only:** (a) an unstated stack, (b) an escalation
(budget overflow or a REJECT no author can fix). **Resume from files:** if
interrupted, just re-run `/sdd-auto` — it reconstructs progress from `.sdd/state.md`
(`iteration: n/budget`) + index `status`, skips `approved` slices, and re-enters the
first non-`approved` slice at the implied sub-phase. See the full step contract in
`.claude/commands/sdd-auto.md` and the pipeline in `.claude/sdd/workflow.mmd`.

### How the stack is chosen
The technologies come from **your prompt**. The AI reads it and writes them into
`.sdd/target.md` (language, architecture, UI framework, build tool, conventions,
canonical commands). **If the stack isn't stated, the AI asks you** before writing
it — it never assumes a default. For an existing SDD project, `.sdd/target.md`
already reflects the stack and your prompt may only extend/override it.

---

## Quick start

**Adopting the workflow:** drop the **`.claude/`** folder into your project (copy it,
or `ln -s` it from this repo) — that single folder *is* the tool: agents, the command,
contracts, templates, and the guaranteed UI-library definition (`.claude/sdd/ui-schema.md`
§9). The requirement (with the **stack**) is the only thing you supply; if the stack
isn't stated, the workflow asks once before writing `.sdd/target.md`. Everything else
(`.sdd/`, `specs/`, `src/`, `tests/`) the workflow creates inside your project.

```
/sdd-auto <describe what to build, and the stack>
```

It runs Plan → Specify → Implement → Test automatically, **one vertical slice at a
time**, advancing each entity `draft → reviewed → approved` through the gates, and
stops only to ask the stack question (if unresolvable) or to escalate a budget
overflow. Review the generated `plan/PLAN.md`, `specs/`, `src/`, and `tests/` as it
goes; re-run the same command to resume an interrupted run.

---

## Reuse-first frontend (guaranteed for every GUI project)

For any project with a GUI, the workflow **guarantees** a default UI component
library — `COMP-appShell`, `COMP-header`, `COMP-body`, `COMP-footer`, `COMP-panel`,
and layout helpers (`COMP-stack`, `COMP-grid`, `COMP-section`). It is **defined** in
`.claude/sdd/ui-schema.md` §9 and **materialized** by the `spec-writer` into the project's
`specs/ui-components/` (with its index) before any screen is specified — it is **not
shipped as files**. Screens **compose these by id**; the `reuse-analyst` progressively
promotes recurring widgets (Button, TextInput, FormField, Modal, Table, …) into the
library so nothing is copy-pasted. A GUI project also guarantees its **e2e harness**:
`MOD-build` owns `playwright.config` and `.sdd/target.md` carries a real `test-e2e`
command, so each screen's `(journey)` ACs are validated end-to-end in a real browser.

---

## What I changed from the starting proposal (and why)

- **Added `.claude/sdd/conventions.md`** — a single canonical reference for ids,
  front-matter, status, verdict format, and the rosters. The agents were each going
  to restate these; centralizing them is the workflow practicing its own DRY value
  and guarantees the eleven prompts can't drift.
- **One automatic command (`/sdd-auto`), not five manual modes** — the flow is folded
  into a single end-to-end orchestrator that runs slice by slice with the human out of
  the loop, and a dedicated **`requirement-analyst`** captures the requirement as the
  first step. This keeps the surface minimal and the pipeline resumable from files.
- **Orchestrator is a command, not a subagent** — because nested subagent spawning
  is unreliable in Claude Code. The matrix's `orchestrator` row becomes the
  `/sdd-auto` command run by the main session.
- **No runtime hook** for test-writer isolation — it is soft (role + minimal tools
  + the test gate), matching the design's "avoid fragile blocks" direction.
- **Stable `[Bn]` branch ids and enumerated arms** in SCoT, so "every branch has a
  test" is checkable mechanically rather than by judgment.
