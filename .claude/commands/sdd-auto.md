---
description: Run the whole SDD flow end-to-end — plan → specs → implement → test, automatic gates, human out of the loop; escalate only on budget overflow or an unstated stack.
argument-hint: "<free-text requirement or feature description>"
---

# /sdd-auto — the SDD orchestrator

Run the WHOLE flow for the requirement in `$ARGUMENTS`: **Plan → Specify → Implement → Test**, one vertical slice at a time, with automatic gates and feedback loops. **You (the main session) are the orchestrator:** invoke each subagent via Task, read each verdict from `.sdd/state.md`, advance index `status` yourself, loop on REJECT. The human is touched only to resolve an **unstated stack** or an **escalation**.

Obey `.claude/sdd/conventions.md` throughout (ids §2, front-matter §3, index rows §4, status/duties §5, verdict format §6, budgets/routing §7, change policy §8, roster §9, topological/slices §12). **Markdown is the source of truth (authority); reuse over repetition (DRY)** — a red test never makes code authoritative; a wrong spec is fixed first, then code is regenerated.

## Preconditions
- A requirement in `$ARGUMENTS` (raw free text is fine; Plan refines it).
- A **resolvable stack**: if `.sdd/target.md` is missing AND the stack cannot be inferred from `$ARGUMENTS`, ask the human the **single stack question** now, capture the answer, then proceed unattended. (This is the only prompt besides escalation.)
- Contracts ship with the tool (read-only): `conventions.md`, `scot.md`, `ui-schema.md`.

---

## Phase A — Plan (analysis budget 3)
1. **Resolve the stack** (ask the human once if unresolvable — see Preconditions).
2. **Capture the requirement yourself** (`plan-architect` does NOT write `requirements/`): write/refine `requirements/REQUIREMENT.md` (raw + refined), assigning a stable `REQ-001`, `REQ-002`, … to each atomic, testable requirement.
3. Invoke **`plan-architect`** → it writes/extends `.sdd/target.md` and `plan/PLAN.md`.
4. Invoke **`plan-gatekeeper`** → read the latest `phase: analysis` record for scope `PLAN`.
   - **PASS** → Phase B.
   - **REJECT** → re-invoke `plan-architect` with the reasons; loop to step 4.
   - **Overflow (>3)** or `routing: escalate` (unresolved `<…>` placeholder) → **ESCALATE**; stop.

## Phase B — Build the slice list
5. From `plan/PLAN.md` + indexes, compute `depends_on` topological order (§12); break any cycle interface-first.
6. Form the **slice list**: one vertical slice = a feature/module + its `depends_on` closure. Order slices so each slice's dependencies are in an already-approved earlier slice.

## Phase C — Per-slice loop (for each slice, in order)
After every gate, read the **latest** record for the slice scope and advance the affected index rows' `status` yourself (§5).

### 7. Specify (analysis budget 3) — advances `draft → reviewed`
   a. **Feature-evolution only:** demote any in-scope entity already at `reviewed`/`approved` to `draft` (§5).
   b. Invoke **`spec-writer`** → index rows + specs for the slice (4 levels + `MOD-build`), `status: draft`.
   c. Invoke **`reuse-analyst`** → dedupe/promote `SHR-*`/`COMP-*`. Then read `specs/REUSE-REPORT.md`: demote to `draft` every id under its `Demote-for-re-gate:` heading.
   d. Invoke **`analysis-gatekeeper`** (the only spec-phase blocker) → read the latest verdict.
      - **PASS** → advance slice spec rows `draft → reviewed`; go to step 8.
      - **REJECT** → route (`spec-writer` for spec defects, `reuse-analyst` for duplication); re-run `reuse-analyst` if specs changed; loop to step 7d.
      - **Overflow (>3)** → ESCALATE; stop this slice.

### 8. Implement (code budget 3) — writes `src/` + impl-notes only
   a. For each spec in `depends_on` order, invoke **`code-implementer`** → minimal **Edit** by default, **regenerate** only by exception (§8); writes only declared `source:` paths + `impl-notes/<id>.md`.
   b. Once `MOD-build` exists, run the canonical `install` (so the read-only compile check is effective), then invoke **`code-gatekeeper`** (read-only) → judge code ≡ spec.
   c. Read the latest code verdict.
      - **PASS** → go to step 9. (Code PASS advances no `status`.)
      - **REJECT** → `routing: code-implementer` (minimal diff) or `routing: spec-writer` (re-validate the spec: demote `reviewed → draft`, re-run `spec-writer` → `reuse-analyst` if changed → `analysis-gatekeeper` → re-advance to `reviewed`, then resume code). Loop to step 8b.
      - **Overflow (>3)** → ESCALATE; stop this slice.

### 9. Test (test budget 5) — advances `reviewed → approved`
   a. Invoke **`test-writer`** (independent oracle — never reads `src/`/`impl-notes/`): ≥1 test per `ACn` and per SCoT arm; for a GUI project, Playwright **e2e** per `(journey)` AC, selecting by accessible role/name from the spec.
   b. Invoke **`test-runner`** with the **slice ids as scope** → runs the canonical commands filtered by `{scope}` (unit + integration + component, + e2e for GUI); writes `tests/REPORT.md`.
   c. Invoke **`test-gatekeeper`** → verify coverage (incl. e2e journeys) + triage failures.
   d. Read the latest test verdict.
      - **PASS** (green + coverage complete) → advance slice rows `reviewed → approved`; go to step 10.
      - **REJECT** → route per triage (§7), each loop returning to the sub-phase that re-gates the fix:
        - **spec bug → `spec-writer`**: demote `reviewed → draft`, loop to **step 7** (re-spec → reuse → analysis gate → `reviewed`), then step 8 regenerates code.
        - **code bug → `code-implementer`**: loop to **step 8** (re-pass the code gate before re-testing).
        - **test bug → `test-writer`**: loop to **step 9b**.
      - **`routing: escalate`** (suite never ran / app won't boot / e2e suite skipped) → **ESCALATE immediately**; not a test-budget iteration.
      - **Overflow (>5)** → ESCALATE; stop this slice.

### 10. Next slice — repeat Phase C until none remain.

## Phase D — Final unscoped arbiter run
11. Once every slice is `approved`, invoke **`test-runner`** with **no scope** (whole suite) and **`test-gatekeeper`** over the whole project to catch cross-scope regressions. On a regression, route per §7 to the owning slice and re-run its step 9 (bounded by the test budget); on green, the project is done.

---

## Escalation (the only human touch-point after stack resolution)
On any budget overflow or an `escalate` routing, STOP that slice (or Phase A) and report concisely:
- **scope** (slice id or `PLAN`), **phase** + iteration vs budget,
- the **failing verdict** verbatim + its **reasons** from `.sdd/state.md`,
- the **author** last routed to.
Never silently retry past budget; never advance `status` on an unresolved scope. Slices already `approved` stay approved.

## Resume (from files only)
Loop state is reconstructable: each `.sdd/state.md` record carries `iteration: <n>/<budget>`, and index `status` shows which slices are done. To resume, re-run `/sdd-auto`: skip every `approved` slice, re-enter the first non-`approved` slice at the sub-phase its latest record implies, continuing that phase's iteration count.

## Status transitions (you make them; gatekeepers never do)
- `analysis` PASS → slice spec rows `draft → reviewed`.
- `test` PASS (green + full coverage; implies a prior code PASS) → `reviewed → approved`.
- `code` PASS alone advances nothing. No verdict ever regresses status; REJECT leaves it unchanged.

## Outputs
- `requirements/REQUIREMENT.md`, `plan/PLAN.md`, `.sdd/target.md`.
- Per slice: index rows + `specs/**/*.spec.md`, `src/**`, `.sdd/impl-notes/<id>.md`, `tests/**`, `tests/REPORT.md`.
- `.sdd/state.md` — append-only verdict log.
- Index `status` advanced to `approved` for every completed slice.
