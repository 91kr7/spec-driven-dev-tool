---
description: Mode 5 — run plan→specs→implement→test end-to-end, automatic gates, no manual approval; escalate only on budget overflow.
argument-hint: "<free-text requirement or feature description>"
---

# /sdd-auto — Mode 5 (Automatic)

Run the WHOLE SDD flow end-to-end for the requirement in `$ARGUMENTS` —
Plan → Specify → Implement → Test — with automatic gates and feedback loops and
the human OUT of the loop. You (the main session) ARE the orchestrator: you
invoke every subagent via Task, read each verdict from `.sdd/state.md`, advance
index `status` yourself, and loop on REJECT. You never stop for manual approval.
You escalate to the human in exactly ONE case beyond an absent stack: an
iteration-budget overflow. This is mode 5 of the 5-mode flow (modes 1–4 are the
manual, human-in-control path `/sdd-plan` → `/sdd-specify` → `/sdd-implement` →
`/sdd-test`; this command fuses all four and runs them unattended).

Honor `.claude/sdd/conventions.md` as canonical throughout: ids (§2), front-matter (§3),
index rows (§4), status lifecycle and separation of duties (§5), the verdict
record format (§6), budgets and failure routing (§7), the change policy (§8), the
agent roster + isolation matrix (§9), and topological / vertical-slice processing
(§12). The two cross-cutting values bind every step:
**Markdown is the source of truth (authority); reuse over repetition (DRY).** A red
test NEVER makes code authoritative — when a spec is wrong, the spec is fixed
first and code is regenerated from it.

## Preconditions
- A requirement is provided in `$ARGUMENTS` (raw free text is acceptable; the Plan
  phase refines it into `requirements/REQUIREMENT.md`).
- A **resolvable stack**. If `.sdd/target.md` does not exist AND the stack cannot
  be inferred from `$ARGUMENTS`, this is the **one and only** place this automatic
  command MUST still ask the human — it cannot assume a default stack. Ask the
  single stack question, capture the answer, then proceed unattended. (After this,
  the human is out of the loop until an escalation.)
- The canonical contracts ship with the tool and are obeyed: `.claude/sdd/conventions.md`,
  `.claude/sdd/scot.md`, `.claude/sdd/ui-schema.md` (read-only — no agent authors them).
- No human approval gate is required to run — that is the defining property of
  mode 5.

## Steps (you, the main session, perform these)

### Phase A — Plan (budget: analysis = 3; no approval stop)
1. Resolve the stack: if `.sdd/target.md` is missing and `$ARGUMENTS` does not
   imply a stack, ask the human the stack question now (the only permitted prompt
   besides escalation); otherwise continue silently.
2. **Capture the requirement yourself first** (`plan-architect` does NOT write
   `requirements/`): write/refine `requirements/REQUIREMENT.md` from `$ARGUMENTS`
   (raw + refined), assigning a stable `REQ-001`, `REQ-002`, … id to each atomic,
   testable requirement (the back-link targets for every spec's `requirements:`).
   Then invoke `plan-architect` via Task, passing `requirements/REQUIREMENT.md` +
   the `.claude/sdd/` contracts; it writes/extends `.sdd/target.md` and writes
   `plan/PLAN.md`. (Contracts ship with the tool — `plan-architect` never authors them.)
3. Invoke `plan-gatekeeper` via Task, passing `plan/PLAN.md`, `requirements/`,
   `.claude/sdd/`, `.sdd/`. Read the **latest** plan-phase record for the scope from
   `.sdd/state.md`.
4. Decide on that verdict:
   - **PASS** → continue automatically to Phase B (DO NOT stop for approval; this
     is the key difference from `/sdd-plan` + `/sdd-specify`).
   - **REJECT** → re-invoke `plan-architect` with the verdict reasons; increment
     the analysis-iteration count for the plan scope; loop back to step 3.
   - **Budget overflow (analysis > 3)** → **ESCALATE** (see Escalation) with
     scope `PLAN`, the failing verdict, and its reasons; STOP. Do not enter
     Phase B.

### Phase B — Build the slice list
5. From the approved `plan/PLAN.md` and the indexes under `specs/indexes/`,
   compute the processing order by `depends_on` (`.claude/sdd/conventions.md` §12):
   topological, dependencies first. On a dependency **cycle**, schedule the
   `interface`/`contract` specs first and let implementations depend on the
   interface (stubs derive from the interface meanwhile).
6. Form the **slice list**: process ONE VERTICAL SLICE at a time — a feature/module
   plus its `depends_on` closure — to bound context and token cost. Order slices
   so that every slice's dependencies belong to an already-approved earlier slice.

### Phase C — Per-slice loop (for EACH slice, in order)
For the current slice, run the three sub-phases in sequence, each as a closed
feedback loop with its own budget. After every gate, read the **latest** record
for the slice scope from `.sdd/state.md` and advance the affected index rows'
`status` yourself (gatekeepers judge only; you advance — §5).

7. **Specify** (budget: analysis = 3) — advances `draft → reviewed`:
   a. Invoke `spec-writer` via Task, passing `plan/PLAN.md`, the slice ids, and the
      contracts; it writes the index rows + specs for the slice (4 levels +
      `MOD-build` where relevant) at `status: draft`. (Feature-evolution run on an
      existing project: first **demote** any in-scope entity already at
      `reviewed`/`approved` to `draft` — §5 — so its modified spec re-flows through
      the gate.)
   b. Invoke `reuse-analyst` via Task on the slice's specs to dedupe and promote
      shared `SHR-*` / `COMP-*` abstractions (DRY) before judging.
   c. Invoke `analysis-gatekeeper` via Task (the only spec-phase blocker), passing
      the slice's specs + `requirements/` + `.claude/sdd/` + `.sdd/`.
   d. Read the latest analysis verdict. **PASS** → advance the slice's spec rows
      `draft → reviewed`; go to step 8. **REJECT** → route per the verdict
      (`spec-writer` for spec defects, `reuse-analyst` for duplication/ownership
      findings), increment the analysis count, loop to step 7b/7c.
   e. **Budget overflow (analysis > 3)** → ESCALATE with this slice id, the failing
      verdict, its reasons; STOP this slice (do not implement it).

8. **Implement** (budget: code = 3) — works in `depends_on` order within the slice:
   a. For each spec in the slice, in topological order, invoke `code-implementer`
      via Task, passing the spec id(s) + `.sdd/impl-notes/` + `.sdd/target.md`. It
      applies the change policy (§8): minimal **Edit** by default, whole-file
      **regenerate** only by exception (new file / substantially changed spec /
      bad drift); it writes only declared `source:` paths and records
      concretizations in `.sdd/impl-notes/<id>.md` (never in the spec).
   b. Once the slice's build manifest (`MOD-build`) exists, run the canonical
      `install` so the compile check is effective, then invoke `code-gatekeeper` via
      Task (read-only) to judge code ≡ spec for the slice.
   c. Read the latest code verdict. **PASS** → go to step 9. **REJECT** →
      re-invoke `code-implementer` with the verdict reasons (minimal diff to match
      the spec), increment the code count, loop to step 8b.
   d. **Budget overflow (code > 3)** → ESCALATE with this slice id, the failing
      verdict, its reasons; STOP this slice.

9. **Test** (budget: test = 5) — advances `reviewed → approved`:
   a. Invoke `test-writer` via Task on the slice's behavioral/feature specs (unit
      from classes, integration from features, constraints from entities, component
      from gui specs, and — for a **GUI project** — Playwright **e2e** for each in-scope
      screen's user journeys). It is the independent oracle — it must NOT read `src/` or
      `.sdd/impl-notes/` (isolation, §9); it covers every SCoT arm id and every `ACn`, and
      selects e2e elements by accessible role/name from the spec.
   b. Invoke `test-runner` via Task, **passing the slice ids as the scope**, to run the
      suite **filtered to the slice** (`{scope}` selector — unit + integration + component,
      plus e2e for a GUI project; not the whole app) and write `tests/REPORT.md` (recording
      the `scope` and `suites`). The baked **dot** reporter keeps captured output small.
   c. Invoke `test-gatekeeper` via Task to verify coverage (incl. the GUI-project e2e
      journey requirement) and triage failures.
   d. Read the latest test verdict.
      - **PASS (full green + coverage complete)** → advance the slice's rows
        `reviewed → approved`; the slice is done — go to step 10.
      - **REJECT** → route per the triage (§7), MD-as-authority. Each route loops
        back to the sub-phase that re-gates the fix, so a regenerated artifact never
        skips its gate:
        **spec bug → `spec-writer`** (the spec is wrong, so re-validate it before
        code, §5: **demote** the entity `reviewed → draft` and loop back to **step 7**
        — re-spec → `reuse-analyst` → `analysis-gatekeeper` → re-advance to
        `reviewed` — then step 8 regenerates code from the corrected spec; a red test
        never patches code arbitrarily); **code bug → `code-implementer`**, loop back
        to **step 8** so the minimal diff re-passes the **code gate** before
        re-testing; **test bug → `test-writer`** (the test must assert a spec
        AC/branch), loop to **step 9b**. Increment the test count.
      - **Run-health / e2e-setup failure routed to escalation** (the suite never ran —
        install/tooling, or the app under test would not launch / browser binaries missing,
        per §7) → **ESCALATE immediately** (env / `MOD-build` / the `run`/`test-e2e`
        command); this is **not** a test-budget iteration and does not loop to an author.
      - **Budget overflow (test > 5)** → ESCALATE with this slice id, the failing
        verdict, its reasons; STOP this slice.

10. Move to the next slice and repeat Phase C, until no slices remain.

11. **Final unscoped arbiter run.** Once every slice is `approved`, invoke `test-runner`
    **with no scope** (whole, unscoped suite — unit + integration + component + e2e) and
    `test-gatekeeper` over the whole project, to catch **cross-scope regressions** a
    per-slice scoped run cannot see. On a regression, route per the §7 triage to the owning
    slice and re-run its step-9 test sub-phase (bounded by the test budget); on green, the
    project is done.

### Escalation (the only human touch-point after stack resolution)
On ANY iteration-budget overflow (analysis > 3, code > 3, or test > 5), STOP that
slice (or Phase A for the plan) and ESCALATE to the human with a concise summary:
- the **scope** (the slice id, e.g. `FEAT-001`, or `PLAN`);
- the **phase** (`analysis | code | test`) and the iteration count vs budget;
- the **failing verdict** verbatim and its **blocking reasons** from `.sdd/state.md`;
- which **author** was last routed to.
Do not silently retry past budget and do not advance status on an unresolved scope.
Other slices that already passed remain `approved`; only the overflowing slice halts.

### Resume (from files only)
Loop state (the current slice and the per-phase iteration counts) lives in this
session, but it is **reconstructable from files** if the run is interrupted — nothing
is lost. The latest `.sdd/state.md` record for a scope carries `iteration: <n>/<budget>`,
and the index `status` columns show which slices are already `reviewed`/`approved`. To
resume, re-run `/sdd-auto`: skip every slice already `approved`, and re-enter the first
non-`approved` slice at the sub-phase its latest `.sdd/state.md` record implies,
continuing that phase's iteration count (do not restart the budget).

## Status transitions
You (not any gatekeeper) advance the index `status` from the latest `.sdd/state.md`
verdict (§5):
- `analysis` PASS → slice's spec rows `draft → reviewed`.
- `test` PASS (full green + full coverage; implies a prior `code` PASS) →
  `reviewed → approved`.
A `code` PASS alone advances no index status (it is a precondition for the test
gate). No verdict ever regresses a status; REJECT leaves status unchanged and
triggers a routed re-invocation.

## Outputs
- `requirements/REQUIREMENT.md`, `plan/PLAN.md`.
- `.sdd/target.md` (the `.claude/sdd/` contracts ship with the tool, read-only — never authored at runtime).
- Slice-by-slice: `specs/indexes/*.index.md` rows + `specs/**/*.spec.md`,
  `src/**` (declared paths), `.sdd/impl-notes/<id>.md`, `tests/**`,
  `tests/REPORT.md`.
- `.sdd/state.md` — the append-only audit trail of every gate verdict.
- Index `status` advanced to `approved` for every completed slice.

## Slice loop (auto)
This command loops Phase C over the slice list until **all** slices are
`approved`, then runs the **final unscoped arbiter test run** (step 11) over the whole
project and reports a final summary: slices processed, their final statuses,
total iterations per phase, the final whole-suite result, and any escalations raised. There is no manual "next
command" — `/sdd-auto` is the top-level orchestrator and the entire mode-5 flow.
For a manual, human-in-control run, use the four-command path instead:
`/sdd-plan` → `/sdd-specify` → `/sdd-implement` → `/sdd-test`. `/sdd-trace` can
reconstruct REQ→FEAT→CLS→SOURCE→TEST at any point.
