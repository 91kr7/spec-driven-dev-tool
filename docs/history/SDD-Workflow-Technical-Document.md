# Spec-Driven Development (SDD) — Workflow Technical Document

This document explains **how the workflow works** — the agentic Spec-Driven Development system that the prompt (`Prompt-SDD-per-ClaudeCode.md`) asks Claude Code to build. It is meant to be read by a person: principles, file structure, agents, modes, and feedback loops, with the rationale behind the choices.

---

## 1. The idea in one sentence

You write **specifications in Markdown**; a set of **agents** analyze them, write them, translate them into code, write and run the tests, and correct each other through **gatekeepers**. Markdown is the **source of truth**: if the code doesn't match the spec, it's the code that is wrong.

---

## 2. The founding principle: "Markdown is the source of truth"

The specs decide **what** the system does and **how it is structured**. Code is a **derived** artifact.

Note an important evolution made during design: the principle holds as **authority**, not as "blindness". That is:

- the agent that writes the code **may read and edit the source** (it has to, because the pseudo-code does not deal with libraries, syntax, or APIs);
- but in any conflict the **spec wins**: the code is corrected toward the spec, never the other way round;
- every change is **verified against the spec** by a gatekeeper;
- implementation details the spec doesn't capture go into a separate file (`impl-notes`), so the behavioral spec stays clean and a **regeneration** of the code from the specs always remains possible;
- the **proof** that the code still honors the spec is the **tests** (behavioral equivalence), not a textual comparison.

Scope: the workflow is **spec-first**, for new projects or for features added to a project already built with SDD. Reverse-engineering legacy, non-spec'd code is out of scope.

---

## 3. The file structure (the single authoritative source)

### Indexes and specs, in plain terms
Think of a **ring binder** with two kinds of pages:

- **Indexes = the tables of contents.** One row per thing, just to know "what exists". There is one per type: one lists the modules, one the features, one the entities, one the classes. They let you avoid opening everything: read the index, open only the spec you need (token saving).
- **Specs = the detail pages.** One spec per single thing, with all the details inside.

**Example with real names.** A user-management app, feature **"User registration"**, which requires two parts: a **Database** (where you store the user) and an **API** (the endpoint that receives the registration).

```
specs/
├── indexes/                ← the TABLES OF CONTENTS (one per type)
│   ├── modules.index.md     rows: Database, API
│   ├── features.index.md    row:  User registration
│   ├── model.index.md       row:  User
│   └── classes.index.md     rows: UserRepository (Database), RegistrationController (API)
│
├── modules/                ← the module SPECS
│   ├── Database.spec.md
│   └── API.spec.md
├── features/
│   └── UserRegistration.spec.md   → says: needs UserRepository + RegistrationController
├── model/
│   └── User.spec.md                → fields: name, email (unique), password…
└── classes/
    ├── UserRepository.spec.md       → logic to store the user (module: Database)
    └── RegistrationController.spec.md→ endpoint logic (module: API)
```

Three rules and that's it: **one table of contents per type** (not per module); **one spec per thing**, in the folder of its type; each class spec states **which module it belongs to** (just a label). Note: in the real system every entry also has a **short id** (e.g. `FEAT-001`, `CLS-...`) used for references; here I used names for readability.

### Full folder layout

```
requirements/REQUIREMENT.md     user requirement (raw + refined)
plan/PLAN.md                    output of Plan mode
.sdd/target.md                  chosen stack + build/test/run commands
.sdd/scot.md                    canonical pseudo-code grammar (SCoT)
.sdd/ui-schema.md               canonical convention for schematic UI
.sdd/state.md                   append-only log of gatekeeper verdicts
.sdd/impl-notes/<id>.md         implementation notes (owned by the implementer)
specs/indexes/*.index.md        indexes (modules, features, classes, ui-components, model) + status
specs/modules/<id>.spec.md      modules (incl. MOD-build for build/CI/migrations)
specs/features/<id>.spec.md     use-cases (orchestration + integration acceptance)
specs/model/<id>.spec.md        domain entities (User, Order…): fields, types, relations, constraints
specs/classes/<id>.spec.md      detailed per-method logic (SCoT)
specs/ui-components/<id>.spec.md shared UI library
specs/shared/<id>.spec.md       shared non-UI abstractions
src/                            CODE (derived)
tests/                          TESTS (derived from the specs)
```

### Indexes: why they exist
Indexes are **lightweight maps**: one row per entry with `id`, name, a **one-line description of what it represents**, a `module` column, dependencies, the spec path, the associated source file, and `status`. They **save tokens**: an agent reads the index first and opens **only** the specs it needs, instead of loading everything. Indexes are also the **home of each entry's status** (`draft → reviewed → approved`).

Indexes are **per level (type), not per module**: one `modules.index`, one `features.index`, one `classes.index`, one `model.index`, one `ui-components.index`, each listing **all** entries of that level with the `module` column showing the partitioning. Adding a feature that spans two modules (e.g. Database + API) means adding rows to the existing per-level indexes and creating one spec per entry in the level folder — you do not create a per-module index. For large projects the fine-grained indexes can be **split per module** (a choice taken in the plan).

---

## 4. The four spec levels

Each thing takes the form that fits its level:

1. **Feature (use-case)** — the flow, which classes/components collaborate (referenced by id), a **high-level orchestration** and the **integration acceptance criteria**. If the orchestration is real code, the feature owns a coordinator/controller file; otherwise it is pure composition and drives the integration tests.
2. **Entity / data-model** — a "thing" the app stores (User, Order, Product…): **fields, types, relations, constraints** (e.g. unique email, age ≥ 0). It is the **single source** from which both the code entity and the DB table/migration are derived. It is **declarative**, no pseudo-code.
3. **Class / service** — the **detailed per-method logic** in SCoT, plus invariants. The real behavior lives here.
4. **UI component / screen** — the **schematic form** of the UI (wireframe + component tree + state/events table).

Tests follow the levels: **unit** from classes, **integration** from features, **validation/constraints** from entities.

### SCoT: structured pseudo-code (behavior only)
For **logic** we use **SCoT** (Structured Chain-of-Thought): pseudo-code with only the *sequence / branch / loop* constructs (plus async where needed) and explicit input/output, so it translates faithfully into any language. The grammar is fixed **once** in `.sdd/scot.md`, with stable branch ids (used to measure test coverage).

Anything that is **not behavior** does not use SCoT: a `kind:` field in each spec decides the form. **DTOs, value objects, enums, entities, config** are **declarative** (a field/type/constraint table). An **interface/contract** only declares the signatures (method, inputs, outputs, errors), with no body: the SCoT lives in the class that implements it. A **stub/mock** is not specced separately: it is derived automatically from the contract.

### Implementation Notes: concretization
SCoT does not talk about libraries, API signatures, or language idioms. Everything SCoT omits but that's needed to reproduce the behavior (chosen library and version, exact API usage, edge-case handling, lessons from bug-fixes) goes into `.sdd/impl-notes/<id>.md`, written by the implementer. Spec (behavior) + impl-notes (concretization) together **determine the behavior** of the code — enough to regenerate it and pass the same tests.

---

## 5. The agents

| Agent | What it does | Can it read the code? |
|---|---|---|
| **plan-architect** | From the requirement, produces the plan of indexes and specs | no |
| **plan-gatekeeper** | Validates the plan (can block) | no |
| **spec-writer** | Writes indexes and specs (4 levels) | no |
| **reuse-analyst** | Removes duplication, promotes shared components/abstractions (pure author) | no |
| **analysis-gatekeeper** | Validates the specs: completeness, testability, traceability, duplication (veto) | no |
| **code-implementer** | Translates specs into code; for existing code applies **minimal diffs** | yes (to edit) |
| **code-gatekeeper** | Verifies the code honors the spec (veto) | yes (to review) |
| **test-writer** | Writes tests from the specs (independent oracle) | no (by design) |
| **test-runner** | Runs the tests and produces a report | yes |
| **test-gatekeeper** | Verifies coverage and triages failures | yes |
| **orchestrator** | Coordinates everything, in dependency order, by vertical slices | no |

Role principle: **gatekeepers judge** and don't edit; **authors** write. The status in the indexes is advanced by the orchestrator/command from the verdicts — so gatekeepers stay pure.

---

## 6. The five modes (slash commands)

The first four are the **manual, step-by-step path with the human in control**; the fifth is **automatic, no human in the loop**.

1. **Plan** — from the user prompt, produce the plan of how it would structure indexes and specs; derive the stack (`target.md`) and, on a new project, write `scot.md`/`ui-schema.md`. The plan-gatekeeper validates automatically.
2. **Plan approval** — the human approves the plan, then indexes and specs are **written** (spec-writer → reuse-analyst → analysis gate).
3. **Implement** — from the approved specs, produce the code (new files or minimal diffs) and run it through the code gate.
4. **Test run** — generate tests from the specs, run them, triage failures.
5. **Automatic** — run everything end-to-end, **one vertical slice** (feature/module) at a time, in dependency order, stopping only on iteration-budget overflow.

---

## 7. Gatekeepers, feedback loops, and bug handling

Each phase (analysis, code, test) has a **reviewer with veto power** that writes a structured verdict (`PASS`/`REJECT` + reasons) to `.sdd/state.md`. Loops have an **iteration budget** (suggested defaults: analysis 3, code 3, test 5); on overflow they **escalate to the human** instead of looping forever.

### How a bug is handled (change policy)
- **Edit by default** (both bug-fix and evolution): the implementer reads spec + impl-notes + source and applies a **minimal diff** to the affected file only. It **does not rewrite whole files** for a small change.
- The **spec remains the authority**: the edit brings the code in line with the spec. If the problem reveals the spec is wrong, fix the **spec first**.
- Any new implementation decision goes back into the **impl-notes**.
- **Regeneration** (exception): a full file rewrite only if it's new, if the spec changed substantially, or if the file drifted too much.
- **Equivalence guarantee = the tests**, not a textual comparison. Coverage is enforced: at least one test per acceptance criterion and per SCoT branch.

### Failure triage
When a test fails, the test-gatekeeper classifies the cause and routes:
- **spec bug** → back to the spec-writer (then regenerate);
- **code bug** → back to the implementer (minimal diff);
- **test bug** → back to the test-writer.

---

## 8. Reuse and the UI library

Reuse is a first-class objective. Gatekeepers **reject** specs or code that duplicate an already-available pattern. For the frontend a **UI component library** is mandatory: ready from the start with Header, Body, Footer, Panel and layout helpers; progressively enriched with the recurring widgets (Button, TextInput, Select, FormField, Modal, Table…), organized in **atomic-design** style. Screens **reference components by id**, they don't re-describe them.

---

## 9. Traceability

There is no separate traceability file: the chain **REQUIREMENT → FEATURE → CLASS → SOURCE FILE → TEST** is reconstructed on demand from the indexes (feature→module, class→feature, the `source` column) plus the `requirements:` and `source:` fields in the specs. The spec→source mapping has a **single authoritative home**: the spec front-matter; the indexes derive it. Each code file carries a header at the top pointing back to its spec.

---

## 10. Why it's built this way (summary of choices)

- **Multi-level indexes** → lazy loading of specs → fewer tokens.
- **SCoT for logic, declarative for entities/structural artifacts/UI** → each thing in the right form, faithfully translatable.
- **Impl-notes outside the spec** → the (gated) behavioral spec stays clean; concretization has its own place.
- **Edit by default, regenerate by exception** → no needless rewrites, minimal blast radius.
- **Tests as oracle with enforced coverage** → "code ↔ spec" equivalence is verified, not assumed.
- **Separated author/judge roles + iteration budget** → robust feedback loops, no infinite loops, human escalation.
