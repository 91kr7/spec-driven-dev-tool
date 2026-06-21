# Prompt for Claude Code — Build a Spec-Driven Development agentic workflow

> Copy the block below and paste it into Claude Code. This is the meta-prompt: it asks Claude Code to **build** the agent system, not to run it. The entire prompt is in English on purpose.

---

## GOAL
Build a **Spec-Driven Development (SDD) agentic workflow**, compatible with Claude Code (subagents in `.claude/agents/` and slash commands in `.claude/commands/`).

**Founding principle: _Markdown is the source of truth_ — as the AUTHORITY.**
Markdown indexes and specifications decide WHAT the system does and how it is structured. In any conflict between code and spec, **the spec wins and the code is corrected** — never the reverse. Code is a **derived** artifact.
This does **not** mean the code agent is forbidden from reading code. It MAY read and edit existing source to apply **minimal diffs** (see "Change policy"). It means: the spec — never the code — is the authoritative definition; every change is verified against the spec by a gatekeeper; implementation details the spec omits are recorded in a separate implementer-owned `impl-notes` file (the gated spec is never altered); and a **full regeneration from spec + impl-notes must always remain possible**, with the independent, spec-derived **test suite** proving the result still matches the spec (behavioral equivalence — not a textual diff).

**Project assumption — spec-first only (new projects AND existing SDD projects).** This workflow targets work that is **spec-first**: either a brand-new project, or a **feature added to a project that was itself built with this workflow** (so its specs already exist). In both cases specs exist before code. The code-writing agent (**code-implementer**) MAY read and edit existing source to apply **minimal diffs** during both **bug-fixing and feature evolution** — it does not blindly rewrite whole files (see "Change policy"). The **test writer** is kept **implementation-independent** (by convention + tools + the test gate, not a hard block — see isolation), because tests must be an **independent oracle**, not a mirror of the implementation. Reverse-engineering a **legacy, non-spec'd** codebase into specs is **out of scope**; if ever needed it would be a separate, explicitly-added agent.

## SCOPE (the system must be broad and technology-agnostic)
The workflow must work for applications of **different types and stacks**, without rewriting the agents:
- **Architectures**: monolithic, modular/multi-module, microservices, libraries/SDKs, CLI.
- **Backend**: Java (Spring/Jakarta), Kotlin, C#/.NET, Node/TypeScript, Python, Go, etc.
- **Frontend/GUI**: React, Angular, Vue, Svelte, plus non-web UIs.
- **Mixed**: full-stack projects with multiple modules and multiple languages in the same repo.

Design consequences you must respect:
- Agents and templates are **language- and framework-agnostic**: the target (language, build tool, UI framework, architecture) is a **parameter**, never a hard-coded assumption.
- Specs describe **behavior and structure**, not implementation choices locked to a framework; the concrete framework is applied at implementation time and is declared in a per-project config file (`.sdd/target.md`).
- For **multi-module / multi-stack** projects, indexes and specs must scale (many modules, many languages) while preserving traceability.

### How the technology stack is chosen (important)
- The technologies to use are **specified by the user prompt** when starting a **new application** or adding a **new feature**.
- It is the **AI that must read the user prompt and write the chosen stack into `.sdd/target.md`** (language, architecture, UI framework, build tool, conventions, and — for a new feature — the existing stack it must fit into).
- If the user prompt does not state the stack for a new app/feature, the AI must **ask the user** before writing `.sdd/target.md` (do not silently assume a default).
- For an **existing SDD project** (already built with this workflow, so its specs exist) where a feature is added, `.sdd/target.md` must reflect the established stack; the user prompt may only extend/override it, and the AI records that.
- Every downstream agent (especially the code-implementer) reads `.sdd/target.md` to know which language/framework to emit. `.sdd/target.md` is itself a Markdown file and therefore part of the source of truth.

## ROLE PROMPTS (a persona at the top of every agent)
Every agent file must **begin with a role prompt** — a short persona block that primes the agent before any instructions. Use a consistent shape:
```
ROLE: <one-line identity, e.g. "You are the Spec Writer">
MISSION: <the single outcome this agent owns>
MINDSET: <the values it optimizes for, e.g. "reuse over repetition, clarity over cleverness">
NON-GOALS: <what it must never do, e.g. "never let code override the spec", "never invent requirements", "never rewrite a whole file for a small change">
```
The role prompt is mandatory for all agents and must explicitly carry the two cross-cutting values: **Markdown is the source of truth (authority)** and **reuse over repetition (DRY)**.

## REUSE-FIRST DESIGN (DRY) — a primary objective, especially for the frontend
Avoiding verbose, copy-pasted code through **massive reuse** is a first-class goal of this workflow, not an afterthought. Bake it into every layer:
- Every agent's role prompt and review criteria must **prioritize reuse**: prefer extracting a shared abstraction over duplicating logic or markup.
- Gatekeepers must **reject** specs or code that duplicate a pattern already available (or that should be promoted to) a shared library. Duplication above a small threshold is a blocking issue.
- The plan and the specs must surface **cross-cutting elements** (shared services, utilities, types, and — above all — UI components) and define them **once** in dedicated shared specs that other specs reference by id.

### Frontend UI component library (mandatory, by default + progressively enriched)
For any project with a GUI, the workflow must produce and maintain a **UI component library** as its own module/spec set, instead of inlining widgets in every screen:
- **Default scaffold from the start** (layout primitives): an app shell with **Header**, **Body/Content area**, **Footer**, and reusable **Panel/Card** containers, plus layout helpers (grid/stack/section). Screens are composed from these, never hand-rolled.
- **Progressive enrichment**: as the spec writer and the reuse analyst identify **recurring cross-cutting widgets**, promote them into the library — e.g. **Button**, **TextInput**, **TextArea**, **Select/Dropdown**, **Checkbox/Radio/Toggle**, **FormField (label+control+error)**, **Modal/Dialog**, **Table/DataGrid**, **Tabs**, **Toast/Notification**, **Badge/Chip**, **Avatar**, **Icon**, **Spinner/Loader**, **Pagination**, **Breadcrumb**, **Menu/Nav**. (Extend this list as patterns emerge; the list is a starting point, not a limit.)
- Organize the library with an **atomic-design** mindset: **atoms** (Button, Input) → **molecules** (FormField, SearchBar) → **organisms** (Header, Table, Panel) → **templates/layouts** (app shell). Each component is specified **once** with its own UI-schematic spec, variants, props/inputs, states, and events.
- Feature/screen specs must **reference library components by id** and only specify composition + screen-specific behavior — never re-describe a button or input inline.
- The library is **framework-agnostic at the spec level**; the concrete implementation (React/Angular/Vue/etc.) is applied at implementation time from `.sdd/target.md`. The same component specs can target different frameworks.
- Add a dedicated index for it: `specs/indexes/ui-components.index.md`, with one-line descriptions, variants, and `status`, so agents can discover existing components **before** creating new ones (discover-before-create is mandatory to prevent duplicates).

## ARTIFACTS AND STRUCTURE (Markdown as the source of truth)
Define and use a Markdown file structure. Starting proposal (adapt it if you can improve it, and explain why):
```
requirements/REQUIREMENT.md          # raw + refined user requirement
plan/PLAN.md                         # output of plan mode
.sdd/target.md                       # stack/architecture/framework + canonical build/test/run commands (written by the AI)
.sdd/scot.md                         # canonical SCoT grammar (incl. stable branch ids for coverage) — authored ONCE, used by all agents
.sdd/ui-schema.md                    # canonical UI-schematic convention (wireframe + component tree + state/events)
.sdd/state.md                        # append-only verdict/audit log (per-entity status lives in the indexes)
.sdd/impl-notes/<spec-id>.md         # implementer-owned concretization notes (NOT part of the gated spec)
specs/indexes/{modules,features,classes,ui-components,model}.index.md   # indexes; also hold per-entity status
specs/modules/<id>.spec.md             # incl. a project/infra module (e.g. MOD-build) for build/config/CI/migrations
specs/features/<id>.spec.md            # use-case: orchestration + integration acceptance, references classes by id
specs/model/<id>.spec.md               # domain entities (User, Order, Product...): fields, types, relations, constraints
specs/classes/<id>.spec.md             # detailed per-method SCoT
specs/ui-components/<id>.spec.md        # shared UI component library (atoms/molecules/organisms/layouts)
specs/shared/<id>.spec.md               # shared non-UI abstractions (services, utils, types)
src/                                 # GENERATED (derived)
tests/                               # GENERATED (unit from class specs, integration from feature specs)
```
Traceability (REQ → FEATURE → CLASS → SOURCE → TEST) is **not** kept as a separate file: it is reconstructable on demand from the indexes (feature→module, class→feature, the `source` column) plus each spec's `requirements` and `source` front-matter. A `/sdd-trace` command may print it when needed.

### Indexes (to reduce token cost and stay efficient)
Create **multi-level indexes** — propose the best granularity among module / feature / class-component, but by default use **all three**, since they answer different questions:
- **Module index**: how the system is partitioned.
- **Feature index**: what the system does for the user (use cases), mapped to modules.
- **Class/Component index**: implementation units, mapped to features/modules — **feature-specific screens go here** (kind: gui), while **shared, reusable widgets live in the ui-components index**.
- **Model index**: domain entities (the "things" the app stores), each with a one-line description, mapped to the modules/features that use them.

Each index entry must contain at least: `id`, `name`, a **one-line description of WHAT it represents** (not how it is built), a **`module` column** (which module the entry belongs to), dependencies, the spec path, a **`source` column** listing the source file(s) this entry maps to (e.g. `src/auth/AuthService.java`, **generated from each spec's authoritative `source:` front-matter**), and `status` (`draft`→`reviewed`→`approved`). Agents read the **index first** and open **only** the specs they need (lazy loading) to minimize tokens. The `source` column also lets an agent jump from a spec straight to its implementation (and back) without scanning the whole tree.

**Indexes are per LEVEL (type), not per module.** There is one `modules.index`, one `features.index`, one `classes.index`, one `model.index`, one `ui-components.index` — each listing **all** entries of that level across the whole project, with the `module` column showing the partitioning. Adding a feature that spans, say, a `MOD-db` and a `MOD-api` module means: add their rows to `modules.index`; add the feature row to `features.index`; add each entity to `model.index` and each class to `classes.index` (tagged with its `module`); and create one spec file per entry under the matching `specs/<level>/` folder. You do **not** create a separate index per module.
**Scaling option:** for a large multi-module project a single `classes.index` can grow big and erode the token saving; the plan-architect may then **split the fine-grained indexes per module** (e.g. `specs/indexes/classes/MOD-api.index.md`), keeping `modules.index` and `features.index` global. This choice is taken in the plan.

**Worked example (illustrative)** — a "User registration" feature needing a Database module and an API module produces:
```
specs/indexes/modules.index.md    rows: MOD-db "Database", MOD-api "API"
specs/indexes/features.index.md   row:  FEAT-001 "User registration"
specs/indexes/model.index.md      row:  ENT-user "User"
specs/indexes/classes.index.md    rows: CLS-userRepo "UserRepository" (module MOD-db),
                                        CLS-regCtrl  "RegistrationController" (module MOD-api)
specs/modules/MOD-db.spec.md      specs/modules/MOD-api.spec.md
specs/features/FEAT-001.spec.md   (references CLS-userRepo + CLS-regCtrl by id)
specs/model/ENT-user.spec.md
specs/classes/CLS-userRepo.spec.md   specs/classes/CLS-regCtrl.spec.md
```
One **index per level** (each row tagged with its `module`); one **spec file per entry** under the matching `specs/<level>/` folder. No per-module index files (unless the scaling option is chosen).

**The indexes are the home of per-entity `status`.** The lifecycle status of every module/feature/class/component lives in its index row (`draft`→`reviewed`→`approved`). To keep gatekeepers pure (they only judge), **the orchestrator/slash command advances the `status` from the latest gate verdict** — the gatekeepers themselves write only to `.sdd/state.md` (the append-only verdict/audit log). So `state.md` records *why*, the index records *where each entity stands*.

### Spec ↔ source mapping (bidirectional traceability — required)
Each spec and the source files that implement it are linked in **both** directions, with a **single authoritative home: the spec's front-matter**. The index `source` column is **derived** from it (a generator/command fills it), never hand-edited.
- **Spec → source (authoritative, in the spec front-matter):** every `*.spec.md` declares the target source file(s) it produces, e.g. `source: [src/auth/AuthService.java]`. Paths/extensions follow `.sdd/target.md` conventions. For a **new** entity they are proposed from those conventions; for an **existing** one they point at the real files. The spec says where its code MUST go.
- **Source → spec:** the code-implementer stamps a **traceability header** in each file pointing back to its spec id (e.g. `// spec: CLS-001 AuthService — specs/classes/CLS-001.spec.md`).
- **Default ownership: one file ↔ one spec.** Prefer each source file owned by exactly one spec. **Shared aggregator files** (a barrel/index, a shared `types` file) MAY be co-owned by several specs, but only if the co-ownership is declared in the `source` mapping and each spec owns a clearly delimited section. The reuse analyst flags accidental (undeclared) shared ownership.
- **Gatekeeper checks:** the analysis gate verifies the mapping is well-formed (paths obey conventions; shared files are declared, not accidental) and that the derived index `source` column matches the specs. The code gatekeeper verifies on-disk reality: every declared `source` file exists and carries its header; every source file traces back to a spec; an orphan source file or a missing target file is **blocking**.
- The code-implementer reads the spec and may read the current file contents to apply a minimal diff, **but the spec remains the authority** — code is brought to the spec, never the spec bent to the code.

### Specifications
Every `*.spec.md` must be **self-sufficient enough to (re)generate its code from scratch without reading existing source** — this keeps full regeneration possible as the safety net, even though day-to-day changes are minimal diffs.

**Four spec levels (different responsibilities):**
- **Feature spec (use-case):** the scenario/flow, the classes/components that collaborate (referenced **by id**, not re-described), a **high-level orchestration SCoT** (the sequence of cross-class calls), and **integration-level acceptance criteria**. It does not duplicate class logic. If the orchestration needs real code, the feature spec **owns a coordinator/controller source file** (declared in its `source:`) whose logic is that orchestration SCoT; otherwise it is purely compositional (no `source`) and only drives integration tests. Either way, **all code still traces to a spec**.
- **Entity / data-model spec:** a domain "thing" the app stores (User, Order, Product…): its **fields, types, relations, and constraints/invariants** (e.g. email unique, age ≥ 0). It is the **single source** from which both the code entity/DTO **and** the DB schema/migrations are derived (migrations are produced via `MOD-build`, never hand-written separately). No imperative SCoT here — it's a declarative description.
- **Class/service spec:** the detailed **per-method SCoT** and invariants — the real behavior lives here.
- **UI component / screen spec:** the schematic UI form.
Tests follow the levels: **unit tests from class specs, integration/acceptance tests from feature specs, and constraint/validation tests from entity specs.**

**Each spec contains the form that fits its level:**
- **Behavior (class/feature) → SCoT pseudo-code** following the **canonical grammar in `.sdd/scot.md`** (authored once): the constructs _sequence / branch / loop_ (plus async/await where behavior is inherently asynchronous) with **explicit input/output**, so it maps **faithfully** (not necessarily 1:1, since paradigms differ) to any target language.
- **Entity (data-model) → a declarative description, NOT SCoT:** a field table (name, type, constraints), relations to other entities, and invariants. The code entity and the DB migration both derive from it.
- **Structural artifacts (DTO, value object, enum, interface/contract, config) → declarative, NOT SCoT:** a field/shape table for DTOs/enums; for an interface/contract just the **signatures** (method, inputs, outputs, errors) with **no body** — the SCoT lives in the class that *implements* the interface. These live in `specs/shared/` if cross-cutting. A **stub/mock is never specced separately**: it is auto-derived from its interface/contract spec (an empty implementation returning defaults), used when a dependency isn't ready yet or by tests.
- **GUI → schematic UI representation** following `.sdd/ui-schema.md` (the UI's source form, NOT SCoT): ASCII wireframe + component tree + state/events table. Event handlers are described schematically; use a small SCoT snippet only where a handler's logic is non-trivial. Declarative/reactive frameworks are produced from this schematic at implementation time.
- A **`kind:` front-matter field** that decides the form: behavioral kinds (`service`, `controller`, `use-case`) carry **SCoT**; structural kinds (`entity`, `dto`, `enum`, `interface`, `config`) are **declarative**; `gui` uses the **UI schematic**. Agents pick the right form from `kind`.
- Purpose, public interface (inputs/outputs/errors), invariants and rules, **testable acceptance criteria each with a stable id (AC1, AC2…) in Given/When/Then form**, a **`requirements:` front-matter list** of the requirement id(s) it serves (back-link for traceability), and a **`source:` front-matter field** (see "Spec ↔ source mapping").

**Implementation Notes live OUTSIDE the spec.** The behavioral spec is gated and the code-implementer must **never edit it**. Concretization the SCoT omits — chosen library/version, exact API usage, language idioms, edge-case handling, and lessons learned from bug-fixes (with rationale) — is written by the implementer to a **separate, implementer-owned file `.sdd/impl-notes/<spec-id>.md`**. Together, **the spec (behavior) + its impl-notes (concretization) determine the code's behavior** — enough that a regeneration is behaviorally faithful (passes the same tests) and does not re-introduce fixed bugs. The goal is **behavioral** completeness, not byte-for-byte reproduction. (The test writer never reads impl-notes — see isolation.)

## REQUIRED AGENTS (from analysis to testing, with feedback loops and strict gatekeepers)
Create a set of subagents covering the whole process. For each, define: purpose, input, output, **granted tools (minimum necessary)**, and **veto criteria**. At minimum:
1. An **analysis/planning** agent that turns the requirement into a plan.
2. A **plan gatekeeper** (can block and send back).
3. An agent that **writes indexes and specifications** (feature / entity / class / UI levels; SCoT per `.sdd/scot.md`, UI schematic per `.sdd/ui-schema.md`). It also creates a **project/infra spec** (e.g. `MOD-build`) that owns build files, dependency manifests, config, DB migrations and CI — so infra is spec-driven and traceable too. DB migrations are **derived from the entity specs**, not authored independently.
4. An **analysis/spec gatekeeper** (verifies completeness, consistency, testability, traceability, mapping well-formedness, **and unjustified duplication**; veto power). This is the single authority that *blocks* the spec phase.
5. A **code-implementer** that turns specs into source (spec + `.sdd/impl-notes/<id>.md` + `.sdd/target.md`). For **new** entities it creates files; for **existing** ones it reads the current source and applies **minimal diffs** (bug-fix and feature evolution alike). It never rewrites a whole file unless the file is new or a regeneration is explicitly requested. It writes concretization decisions to its **impl-notes file** and **never edits the gated spec**. The spec is always the authority (see "Change policy").
6. A **code gatekeeper** (verifies code faithfully implements the spec; veto power).
7. An agent that **writes tests from the specs** — **implementation-independent**, reading only the **behavioral** part of each spec (public interface, acceptance criteria, SCoT), **never** `src/` or the impl-notes. It must produce **at least one test per acceptance-criterion id and per SCoT branch id** (unit from class specs, integration from feature specs), so the suite is a real oracle.
8. An agent that **runs the tests** (using the canonical build/test commands from `.sdd/target.md`, building/installing deps as needed) and produces a structured failure report.
9. A **test gatekeeper / triage** agent that (a) **verifies coverage** — every AC and every SCoT branch has a test, else REJECT — and (b) classifies each failure as a **spec**, **code**, or **test** bug and routes the feedback to the right agent.
10. A **reuse/abstraction analyst (DRY analyst)** — see below.
11. An **orchestrator** that runs the whole flow automatically, processing entities in **`depends_on` topological order** (dependencies first; on cycles, generate interfaces/contracts first) and, in auto mode, **one vertical slice (feature/module) at a time** to bound context and token cost.

### Reuse / abstraction analyst (dedicated agent — required)
ROLE: "You are the Reuse Analyst." MISSION: maximize reuse and eliminate duplication across the specs, designing reuse **at the spec level** (before code) and **re-running whenever specs change** (including feature evolution). It runs after the spec writer and before the code-implementer. It must:
- Scan all indexes and specs to detect **recurring patterns**: repeated logic, near-duplicate SCoT blocks, repeated UI widgets, repeated types/DTOs/validation rules.
- For UI: identify cross-cutting components (buttons, inputs, panels, tables, modals, etc.) and ensure they are **promoted into `specs/ui-components/`** and registered in `ui-components.index.md`; rewrite feature/screen specs to **reference them by id** instead of inlining.
- For non-UI: promote shared services/utilities/types into `specs/shared/` and update references.
- Produce a **refactor proposal** (which abstractions to extract, what to deduplicate, estimated duplication removed) and apply it to the specs (or hand precise edits to the spec writer).
- Enforce **discover-before-create**: flag any new spec that re-implements something already in the library.
- Operate at the **spec level only** (Markdown), so reuse is designed before code exists — keeping with "MD is the source of truth". It is a **pure author**: it edits/deduplicates specs but does **not** itself gate. The *analysis gatekeeper* is what blocks residual unjustified duplication (clean author/judge separation).
The code-implementer and code gatekeeper must honor these shared specs: emit shared library code once and import it everywhere, never duplicate it.

### AGENT ISOLATION (enforce it for real — defense in depth)
Two kinds of isolation must hold.

**A) Context isolation (between ALL agents).** Each subagent runs in its own context and must communicate with the others **only through files** — specs, indexes, `.sdd/state.md`, impl-notes — never through shared conversational memory. An agent loads the relevant index first and opens only the files it needs. No agent relies on what another agent "said," only on what is written to MD. This keeps hand-offs clean, auditable, and token-cheap.

**B) Test-writer independence (soft, by design — no hard guard).** The **test writer** should not look at the implementation: tests must be an independent oracle derived from the **behavioral** part of the specs (public interface, acceptance criteria, SCoT) — never `src/` or the impl-notes — otherwise tests would risk mirroring the implementation (and its bugs) instead of checking the spec. We enforce this **without a runtime block** (no path-guard hook), using three soft layers:

1. **Minimal tools.** Grant the test writer only `Read`, `Write`, `Glob`; do **not** grant `Bash`. With no shell and a `Read` scoped by its role prompt, there is no natural path to the implementation.
2. **Explicit role prompt.** Its `NON-GOALS` state: never read `src/`, never read `.sdd/impl-notes/`; derive every test from the spec's behavior only.
3. **Test-gatekeeper check.** The test gatekeeper rejects any test that asserts implementation details rather than a spec acceptance criterion / SCoT branch — catching coupling after the fact.

(We deliberately dropped the earlier `PreToolUse` hook + lock file: it was fragile and easy to misconfigure into blocking all reads. Soft enforcement fits the "avoid blocks" direction and is enough, because tests are also checked by the gatekeeper.)

**The code-implementer is NOT isolated** — it reads and edits `src/` to apply minimal diffs. It is kept honest by: **spec-as-authority + gatekeeper + an independent test oracle**, not by read-blindness. Concretely:
- the spec always wins; the implementer brings code to the spec, never the reverse;
- any concretization the SCoT omits is written to `.sdd/impl-notes/<id>.md` (never into the gated spec), so MD stays complete enough to rebuild behavior;
- the **code gatekeeper** re-verifies `code ≡ spec` after every change;
- the **spec-derived test suite is the equivalence check**: because the tests are written independently from the specs (implementation-independent test writer), a passing suite is what proves the code still honors the specs after any edit. A **full regeneration of a file from its spec** is always possible on demand (e.g. if it drifts badly), and the regenerated code must pass the same tests. We rely on **behavioral** equivalence (tests), never on a textual diff against the old code.

This is still defense in depth: even if a prompt-injection in a spec tells an agent to do the wrong thing, the gatekeeper + the independent test suite catch it. All code-writing agents read `.sdd/target.md` to know which language/framework to emit (an MD file inside the allow-list).

### Isolation matrix (define each agent's scope explicitly)
Produce this matrix in the README and honor it in each agent's `tools` frontmatter and role prompt:

| Agent | May READ | May WRITE | Bash/Edit/Task | Reads `src/`? |
|---|---|---|---|---|
| plan-architect | `requirements/` | `plan/` | none | no |
| plan-gatekeeper | `requirements/`, `plan/` | `.sdd/state.md` | none | no |
| spec-writer | `plan/`, `requirements/`, `specs/`, `.sdd/scot.md`, `.sdd/ui-schema.md` | `specs/` (incl. indexes) | none | no |
| reuse-analyst | `specs/` | `specs/` | none | no |
| analysis-gatekeeper | `specs/`, `requirements/` | `.sdd/state.md` | none | no |
| **code-implementer** | `specs/`, `.sdd/{target,scot,ui-schema}.md`, `.sdd/impl-notes/`, `src/` (files it touches) | `src/` (declared paths), `.sdd/impl-notes/<id>.md` | Edit (minimal diffs) | yes — minimal diffs; spec stays authority |
| code-gatekeeper | `specs/`, `.sdd/impl-notes/`, `src/` | `.sdd/state.md` | Bash read-only | yes (review) |
| **test-writer** | behavioral spec sections, `.sdd/{target,scot,ui-schema}.md` | `tests/` | none | **no — by role (soft), not impl-notes either** |
| test-runner | `tests/`, `src/`, `.sdd/target.md` | `tests/REPORT.md` | Bash (run tests) | yes |
| test-gatekeeper | `tests/REPORT.md`, `specs/`, `src/`, `tests/` | `.sdd/state.md` | none | yes |
| orchestrator | `.sdd/**`, indexes | `.sdd/**`, **index `status`** | Bash, Task | no — coordinates, updates status from verdicts |

Rule of thumb: **gatekeepers judge** (verdict → `.sdd/state.md`) and never edit artifacts; the **orchestrator/command** advances index `status` from the verdicts. The **test writer** stays implementation-independent by role + tools + the test gate (soft). The **code-implementer** may read/edit `src/` but is bound by spec-as-authority, the code gatekeeper, and the independent test suite.

### Gatekeepers and feedback loops (strict)
- Each phase (analysis, code, test) has a dedicated reviewer with **veto power** that emits a **structured** verdict (`PASS`/`REJECT` + reasons) written to `.sdd/state.md`.
- Feedback loops have an **iteration budget** (suggested defaults: analysis loop 3, code loop 3, test loop 5 — make them configurable); on overflow, **escalate to the human** instead of looping forever.
- Routing rule consistent with "MD is the source of truth": a red test never patches the code arbitrarily; either the code is fixed to match the spec, or — if the spec is wrong — go back to the spec and **regenerate** the code. Code never becomes the source of truth.

### Change policy — edit by default, regenerate by exception (covers bug-fix AND feature evolution)
A change never forces a full-codebase rewrite, and **the code-implementer must not rewrite a whole file for a small change**. SCoT is intentionally abstract metacode: it specifies behavior (sequence/branch/loop, I/O, invariants) but **not** library choices, API signatures, or language syntax. Many fixes and evolutions live exactly in that concretization layer (`=` vs `==`, a wrong library call, adding a field, wiring a new dependency) and are best done by **reading the actual source and editing it surgically**.

- **Edit (the default), for both bug-fixes and feature evolution:** the code-implementer reads the spec (SCoT) + its `.sdd/impl-notes/<id>.md` and the existing source, then applies a **minimal diff** to the file(s) the changed spec maps to. Surrounding code is preserved; the blast radius is the touched entity, not the project.
- **Spec stays the authority:** every edit brings the code in line with the spec, never the reverse. The **code gatekeeper** re-verifies `code ≡ spec` after the change. If a failure or new requirement reveals the spec is wrong/incomplete, update the **spec first**, then implement.
- **Back-propagate to MD:** any implementation decision the SCoT doesn't capture (library, API binding, idiom, edge case) is written into **`.sdd/impl-notes/<id>.md`** (never the gated spec), so the MD layer stays behaviorally determinative and a future regeneration won't undo or re-break it.
- **Regenerate (the exception):** do a whole-file (re)write only when the file is **new**, when the spec changed so substantially that a clean rebuild is clearer, or when the file has drifted badly from its spec. Regeneration is driven by spec + impl-notes and, because specs/files are small and single-responsibility, still touches one file at a time.
- **Equivalence guarantee = the tests, not a textual diff.** The independent, spec-derived test suite is what proves the code still honors the specs after edits. If a file is ever regenerated from scratch, it must pass the same tests. This is what keeps MD the real source of truth despite editing — without depending on the (non-deterministic) exact text the implementer emits.

## MODES TO EXPOSE (slash commands)
Implement these five modes. **Modes 1–4 are the step-by-step, human-in-control path** — the human runs each step and decides when to proceed to the next; **mode 5 runs everything automatically with the human NOT in the loop.**
1. **Plan**: from the user prompt, the **plan-architect** produces a plan of **how it would create or modify the indexes and specs** (which modules/features/classes/components, dependencies, what would change) — **no specs or code yet** — then the **plan-gatekeeper** validates it automatically (a REJECT loops back to the planner). This is also where the AI derives the stack and writes `.sdd/target.md` (asking the user if the stack isn't stated), and — if not already present — authors the shared contracts `.sdd/scot.md` and `.sdd/ui-schema.md` once.
2. **Plan approval**: capture **human approval** of the plan, then **write the indexes and specifications** (spec-writer → reuse-analyst → analysis gate). Output: the MD specs, ready for the human to review.
3. **Implement**: from the approved specs, produce the code (code-implementer — create new files or apply minimal diffs to existing ones) and run it through the code gate.
4. **Test run**: generate tests from specs, run them, and triage failures.
5. **Automatic**: run the whole workflow end to end (plan → write specs → implement → test) with the automatic gates and feedback loops, **human not in the loop** — it does not stop for manual approval and escalates only on iteration-budget overflow. It processes **one vertical slice (feature/module) at a time in `depends_on` order** to bound context and token cost.

## OUTPUT I EXPECT FROM YOU (Claude Code)
- The subagent files in `.claude/agents/*.md` (with `name`, `description`, `tools` frontmatter), **each starting with a ROLE prompt** as defined above.
- The slash command files in `.claude/commands/*.md` for the 5 modes.
- The **templates** for index and spec files: one for logic with SCoT, one for an entity (declarative), one for GUI with UI schematic, and one for a **shared UI component** (atom/molecule/organism with variants, props, states, events), each with a filled example.
- The **`ui-components.index.md`** plus the default UI library scaffold specs (Header, Body, Footer, Panel, layout helpers) so the reuse-first frontend is in place from the start.
- The shared contracts authored once: **`.sdd/scot.md`** (canonical SCoT grammar) and **`.sdd/ui-schema.md`** (UI-schematic convention), referenced by every agent.
- The isolation matrix in the README (no runtime guard/hook — test-writer independence is by role + tools + the test gate; the code-implementer is kept honest by the gatekeeper + the independent test suite).
- A short `README.md` explaining the structure, the flow, the 4 spec levels, and how to use the modes.

## LANGUAGE
The entire deliverable — all agent prompts, templates, README, and comments — must be written **100% in English**.

## BEFORE YOU START
If anything is ambiguous (e.g., the target stack is not stated in the user prompt, index granularity, the exact SCoT format), **ask me 2-3 focused questions**; then proceed.
