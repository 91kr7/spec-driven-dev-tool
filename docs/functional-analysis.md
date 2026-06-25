# SDD — Functional Analysis (the core concept)

**Main concept.** The Markdown **spec is the single source of authority**; code and
tests are *derived* artifacts. Work flows in one direction —
requirement → spec → code → test — and any conflict is always resolved *back toward
the spec*, never the reverse. A non-authoring orchestrator drives this forward one
vertical slice at a time, and every phase transition must be licensed by an
independent **gate**.

The seven building blocks and how they chain:

## 1. Requirements
The raw request is refined into **atomic, testable** statements with stable ids
(`REQ-001`, …) in `requirements/REQUIREMENT.md`. This is the *what* the user asked
for — the root every later artifact must trace back to. Ambiguity is left as an
explicit `<…>` open question, never silently resolved.

## 2. Indexes
One **index per level** (`modules`, `features`, `model`, `classes`,
`ui-components`) — a table of
`id · name · what · module · depends_on · spec · source · status`. The index is the
**map and the canonical home of `status`** (`draft → reviewed → approved`). Agents
read the index *first*, then open only the specs they need (lazy loading). It's how
the system stays navigable without loading everything.

## 3. Specs
The actual *what*, one file per thing, in the form that fits it (chosen by a `kind:`
field):

- **feature** (use-case) → orchestration + integration ACs
- **entity** → fields, relations, invariants (declarative)
- **class/service** → per-method behavior + invariants
- **gui** → UI schematic (wireframe + component tree)

Each spec is **self-sufficient and regenerable**: it carries front-matter (`id`,
`depends_on`, `requirements:`, `source:`) and **acceptance criteria** (`ACn`,
Given/When/Then). The `source:` field is the single authoritative spec→source map.

## 4. Metalanguage (SCoT)
Behavioral specs are written in **SCoT** — a fixed, language-agnostic pseudo-code
grammar (sequence / branch / loop / try, explicit I/O). Its key trick: **every
decision carries a stable branch id** (`[B1]`, arms `B1.then` / `B1.else` …). This
makes coverage *mechanical* — "test every branch" becomes a checkable
set-membership test, not a judgment. SCoT describes **behavior only**; concrete
libraries/APIs/syntax are forbidden (those go to `impl-notes`).

## 5. Implementations
The `code-implementer` turns a **reviewed** spec into source — minimal diff by
default, regenerate by exception. Every file gets a traceability header
(`// spec: CLS-… — specs/…`), and every decision the spec omits (library, idiom,
edge case) is recorded in `.sdd/impl-notes/<id>.md`, **never** back in the spec.
Code is generated *only* from a reviewed spec; if the spec turns out wrong, the spec
is fixed first and the code regenerated.

## 6. Tests
The `test-writer` is an **independent oracle**: it derives tests *from the specs
alone* — it never reads `src/`. It produces ≥1 test per acceptance criterion and per
SCoT branch arm (+ Playwright e2e for GUI journeys), each tagged with the canonical
coverage id. The passing suite is therefore the **proof of behavioral equivalence**
between spec and code — not a textual diff, and not circular (the oracle can't just
confirm whatever the code does).

## 7. Gates
Four **gatekeepers** are the only things that license a phase transition. They
**judge only** — append a `PASS/REJECT` verdict to `.sdd/state.md`, never edit
artifacts, never advance status:

- **plan-gatekeeper** — is the plan sound and fully traced to requirements?
- **analysis-gatekeeper** — are the specs complete, testable, non-duplicative?
  (`draft → reviewed`)
- **code-gatekeeper** — does the source faithfully implement the spec?
- **test-gatekeeper** — is coverage complete *and* the suite green? + triages each
  failure as spec/code/test bug. (`reviewed → approved`)

A REJECT routes to exactly one author to fix; loops have **iteration budgets**
(analysis 3, code 3, test 5) and **escalate to the human** on overflow instead of
looping forever.

---

**The chain, in one line:**

> `REQUIREMENT → (index + spec, in SCoT) → ✓analysis gate → code + impl-notes →
> ✓code gate → spec-derived tests → ✓test gate → approved`

with the rule that **the spec always wins** — a red test fixes the code (or the
spec, then regenerates the code), never the other way around. Three actor classes
keep it honest: **authors** write artifacts, **gatekeepers** judge, the
**orchestrator** advances status — and no one does two of those jobs.
