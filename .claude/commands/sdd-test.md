---
description: Mode 4 — generate tests from specs, run them, verify coverage, triage failures.
argument-hint: "<optional: spec/feature id or slice; default = all implemented specs>"
---

# /sdd-test — Mode 4 (Test run)

Generate independent, spec-derived tests for the scope, run them, verify every
acceptance criterion and SCoT branch arm is covered, and triage each failure to
its true source (spec / code / test). This is the **last manual, human-in-control
mode** (modes 1–4); mode 5 (`/sdd-auto`) runs the same loop without a human. On
full green it advances the scope from `reviewed` to `approved`. Core values:
**Markdown is the source of truth (authority); reuse over repetition (DRY)** — a
red test never patches code arbitrarily; the triage decides what is actually wrong.

## Preconditions
- `.sdd/target.md` exists and declares the canonical build/test/run commands.
- `.sdd/conventions.md`, `.sdd/scot.md`, `.sdd/ui-schema.md` exist (shared contracts).
- The scope's code already exists with a **code gate PASS** — i.e. the relevant
  index rows are at status `reviewed` and the latest `.sdd/state.md` record for
  the scope (`phase: code`) is `PASS`. If code is missing or the code gate has not
  passed, STOP and tell the human to run `/sdd-implement` first.
- `src/` holds the implementation the test-runner will execute.

## Scope resolution
- `$ARGUMENTS` may name a feature id (`FEAT-001`), a class id (`CLS-userRepo`), or
  a slice. If empty, the scope is **all implemented specs at status `reviewed`**.
- Resolve the scope from the indexes under `specs/indexes/`, expanding a feature to
  its `depends_on` closure so integration tests have their unit-level targets. Pass
  only the resolved **paths/ids** to each agent (hand-off is file-based).

## Steps (you, the main session, perform these)
1. **Author tests.** Invoke `test-writer` via Task, passing the resolved spec
   paths/ids for the scope. It is an **independent oracle**: it reads only the
   behavioral spec sections (and `.sdd/{target,scot,ui-schema,conventions}.md`),
   **never** `src/` or `.sdd/impl-notes/`. It must emit **≥1 test per acceptance
   criterion `ACn`** and **≥1 test per SCoT branch arm** (e.g. `B1.then`,
   `B1.else`, `B3.empty`), tagging each test with its coverage id
   `<spec-id>::<function>#<arm-id>` / `<spec-id>#ACn`. Output goes to `tests/`.
2. **Run tests.** Invoke `test-runner` via Task. It runs the **canonical test
   command from `.sdd/target.md`** (never an ad-hoc command), captures pass/fail
   and any coverage data, and writes `tests/REPORT.md`.
3. **Judge + triage.** Invoke `test-gatekeeper` via Task, passing the scope ids,
   `tests/REPORT.md`, and the spec paths. It (a) verifies **coverage** — REJECT if
   any `ACn` or any branch arm in scope is uncovered — and (b) **triages each
   failing test** to one source with routing: **spec bug → `spec-writer`**, **code
   bug → `code-implementer`**, **test bug → `test-writer`**. It writes a verdict to
   `.sdd/state.md` (`phase: test`) and edits nothing else.
4. **Read the verdict.** Read the **latest** `.sdd/state.md` record for the scope.
5. **Decide:**
   - **PASS** (coverage complete AND suite green) → **YOU** advance the index
     `status` of each in-scope entity from `reviewed` to `approved` in its index
     row. The scope is done; go to Outputs.
   - **REJECT** → for each triaged item, re-invoke the routed author with the
     verdict reasons (passing the failing coverage ids and the spec/AC/branch they
     concern), then re-run **only the affected steps**:
     - **spec bug** → the spec itself is wrong, so it must be **re-validated before
       any code** (§5): YOU demote the affected entity `reviewed → draft`, re-invoke
       `spec-writer` with the reasons, re-run `reuse-analyst` if its specs changed,
       then re-invoke `analysis-gatekeeper`; on its PASS re-advance `draft → reviewed`.
       **Only then** re-invoke `code-implementer` for the affected file(s) (minimal
       diff) and redo steps 1–3 for the affected ids. (Code is only ever generated
       from a `reviewed` spec.)
     - **code bug** → `code-implementer` applies a minimal diff to match the spec;
       redo steps 2–3 (re-run + re-judge); re-author tests (step 1) only if
       coverage was also flagged.
     - **test bug** → `test-writer` fixes the offending test (it must still assert
       a real `ACn`/branch arm); redo steps 2–3.
   - Increment the **test iteration** count for the scope and record it as
     `iteration: <n>/5` in the next verdict cycle.
6. **Budget & escalation.** The **test budget is 5** iterations. On the 6th would-be
   iteration (budget overflow) → **ESCALATE to the human**: stop the loop and print
   a concise summary (scope, remaining uncovered `ACn`/arms, persistently failing
   tests with their last triage and routing, and the canonical command that was
   run). Do **not** advance any status on overflow.

## Status transitions
- `reviewed → approved` for every in-scope entity, **only** on a test gate `PASS`
  (coverage complete and suite green). You (the main session) make this edit in the
  index row; the gatekeeper never writes `status`.
- No other transition is performed here. If a spec bug forces a spec change, the
  affected entity's status is handled by the routed re-run, not advanced here until
  the gate ultimately passes.

## Outputs
- `tests/` — the generated, spec-derived suite (unit tests from class SCoT, integration
  tests from features, constraint tests from entities), each test tagged with its
  coverage id.
- `tests/REPORT.md` — the latest run report (pass/fail, coverage).
- Appended verdict record(s) in `.sdd/state.md` (`phase: test`).
- Updated `status: approved` rows in the relevant `specs/indexes/*.index.md` on PASS.

## Next command (manual path)
- On **PASS**: the scope is fully approved — there is no further manual mode for it.
  Run `/sdd-trace <id>` to print the REQ→FEAT→CLS→SOURCE→TEST chain for the scope,
  or start the next feature/slice at `/sdd-specify`.
- On **REJECT within budget**: the loop above continues automatically; no new
  command is needed.
- On **ESCALATE (budget overflow)**: the human resolves the blocker (amend the
  spec, the code, or the test guidance) and re-runs `/sdd-test <id>` for the scope.
