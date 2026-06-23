---
description: Run the whole SDD flow end-to-end — plan → specs → implement → test, automatic gates, human out of the loop; escalate only on budget overflow or an unstated stack.
argument-hint: "<free-text requirement or feature description>"
---

# /sdd-auto — SDD orchestrator

Run the full flow for `$ARGUMENTS`, one vertical slice at a time, with automatic gates and feedback loops.
Steps run in order 1 → 11; steps 7–10 repeat once per slice.

```
DATAFLOW
  requirement_text
    └▶ 1 stack_decision ─▶ 2 REQUIREMENT.md ─▶ 3 PLAN.md + target.md ─▶ 4 verdict_record
        └▶ 5 topo_order ─▶ 6 slice_list
            └▶ ┌── per slice ──────────────────────────────────────────────────┐
               │ 7 spec_paths ─▶ 8 src_paths ─▶ 9 test_paths ─▶ (gates) ─▶ 10 next │
               └──────────────────────────────────────────────────────────────────┘
                └▶ 11 whole-suite verdict_record ─▶ done
```

```
ROLE   you (main session) = orchestrator; subagents do the work via Task.
        you read verdicts from .sdd/state.md, set index `status` yourself, loop on REJECT.
LAW    Markdown spec = source of truth (authority). Reuse over repetition (DRY).
        A red test never makes code authoritative — fix the wrong spec first, then regenerate code.
RULES  obey .claude/sdd/conventions.md: ids §2 · front-matter §3 · index rows §4 · status/duties §5
        · verdict format §6 · budgets/routing §7 · change policy §8 · roster §9 · topo/slices §12.
HUMAN  touched twice only: (a) an unstated stack, (b) an escalation.
```

## Preconditions
```
REQUIRE  requirement_text present in $ARGUMENTS (raw text ok; step 2 refines it).
REQUIRE  resolvable stack:
           IF .sdd/target.md missing AND stack not inferable from requirement_text
           THEN ask the human the single stack question now, capture answer, then run unattended.
           (this is the only prompt besides escalation.)
HAVE     read-only contracts shipped with the tool: conventions.md, scot.md, ui-schema.md.
```

## Data types   *(the typed tokens used in every step's IN / OUT)*
```
requirement_text  : string                — raw $ARGUMENTS
stack_decision    : { lang, framework, … } — resolved target stack (held in memory until step 3)
REQUIREMENT.md    : file  { raw, refined, req_ids: REQ-* }
PLAN.md           : file  — plan of indexes/specs to create or modify
target.md         : file  — stack + canonical install/build/test commands
topo_order        : [ feature_id | module_id ]            — dependency-sorted
slice             : { slice_id, member_ids[], depends_on_closure[] }
slice_list        : [ slice ]                              — execution order
index_rows        : rows in level indexes, each with `status: draft|reviewed|approved`
spec_paths        : [ specs/**/*.spec.md ]  (4 levels + MOD-build)
REUSE-REPORT.md   : file  { promoted: SHR-*|COMP-*, demote_ids[] }
src_paths         : [ path ]   — only the spec's declared `source:` paths
impl_note         : .sdd/impl-notes/<id>.md
install_result    : { ok: bool, log }
test_paths        : [ tests/** ]
REPORT.md         : file  { failures[], per-test status }   — tests/REPORT.md
verdict_record    : .sdd/state.md row { phase, scope, verdict: PASS|REJECT, reasons[], routing, iteration: n/budget }
```

> Each step lists its **IN** (tokens consumed) and **OUT** (tokens produced). Gate steps emit a
> `verdict_record`; only **you** act on it via the PASS / REJECT / OVERFLOW branches.

---

### Step 1 — Resolve the stack
```
IN   requirement_text ; target.md (if present)
DO   take stack from target.md → else infer from requirement_text → else ask the human once (Preconditions).
OUT  stack_decision
```

### Step 2 — Capture the requirement   *(YOU write it — not plan-architect)*
```
IN   requirement_text
DO   write & refine; assign a stable REQ-001, REQ-002, … per atomic, testable item.
OUT  REQUIREMENT.md { raw, refined, req_ids }
```

### Step 3 — INVOKE plan-architect
```
IN   REQUIREMENT.md ; stack_decision
DO   architect derives the stack file and the plan of indexes/specs to create or modify.
OUT  target.md ; PLAN.md
```

### Step 4 — GATE plan-gatekeeper
```
IN   PLAN.md ; REQUIREMENT.md ; conventions.md
OUT  verdict_record { phase: analysis, scope: PLAN, iteration: n/3 }
      PASS                              → step 5.
      REJECT                            → re-INVOKE plan-architect (step 3) with reasons[]; loop step 4.
      OVERFLOW(>3) OR routing: escalate → ESCALATE; stop.   (escalate = unresolved <…> placeholder)
```

### Step 5 — Compute topological order
```
IN   PLAN.md ; index_rows.depends_on
DO   topo-sort features/modules (§12); break any cycle interface-first.
OUT  topo_order
```

### Step 6 — Build the slice list
```
IN   topo_order ; depends_on closures
DO   one slice = a feature/module + its depends_on_closure;
     order slices so each slice's deps live in an already-approved earlier slice.
OUT  slice_list
```

```
╔══════════════════════ LOOP: for each slice in slice_list, in order — steps 7 → 10 ══════════════════════╗
║ AFTER EVERY GATE: read the latest verdict_record for scope == slice.slice_id,                            ║
║                   then advance the affected index_rows.status yourself (§5).                             ║
╚══════════════════════════════════════════════════════════════════════════════════════════════════════════╝
```

### Step 7 — Specify   `[analysis budget 3]`   `status: draft → reviewed`
```
7a demote (feature-evolution only)
     IN  index_rows where in-scope entity status ∈ {reviewed, approved}
     OUT same rows → status: draft (§5)
7b INVOKE spec-writer
     IN  slice ; PLAN.md ; target.md ; conventions.md
     OUT index_rows + spec_paths (4 levels + MOD-build), all status: draft
7c INVOKE reuse-analyst
     IN  spec_paths (this slice) ; existing SHR-* / COMP-* specs
     OUT promoted SHR-*/COMP-* specs ; REUSE-REPORT.md
     THEN for each id in REUSE-REPORT.md.demote_ids → set index_rows.status: draft
7d GATE analysis-gatekeeper   (the only spec-phase blocker)
     IN  spec_paths ; REUSE-REPORT.md ; REQUIREMENT.md ; conventions.md
     OUT verdict_record { phase: analysis, scope: slice_id, iteration: n/3 }
      PASS         → YOU set slice spec index_rows.status: draft → reviewed; step 8.
      REJECT       → route by routing: spec defect → spec-writer (7b) · duplication → reuse-analyst (7c);
                     re-run reuse-analyst if spec_paths changed; loop step 7d.
      OVERFLOW(>3) → ESCALATE; stop slice.
```

### Step 8 — Implement   `[code budget 3]`   *(writes src/ + impl-notes only)*
```
8a INVOKE code-implementer   (per spec, in depends_on order)
     IN  one reviewed spec ; target.md ; existing src_paths
     DO  minimal Edit by default; regenerate only by exception (§8).
     OUT src_paths (the spec's declared `source:` paths) ; impl_note
8b run canonical `install` → install_result, THEN GATE code-gatekeeper
     IN  src_paths ; spec_paths (this slice) ; impl_note ; install_result
     OUT verdict_record { phase: code, scope: slice_id, iteration: n/3 }
      PASS         → step 9.   (code PASS advances no status.)
      REJECT by routing:
        code-implementer → minimal diff (8a); loop step 8b.
        spec-writer      → re-validate spec: set status reviewed → draft, re-run
                           spec-writer → reuse-analyst (if changed) → analysis-gatekeeper
                           → re-advance to reviewed, then resume code; loop step 8b.
      OVERFLOW(>3) → ESCALATE; stop slice.
```

### Step 9 — Test   `[test budget 5]`   `status: reviewed → approved`
```
9a INVOKE test-writer   (independent oracle — NEVER reads src/ or impl-notes/)
     IN  spec_paths (this slice) ONLY
     OUT test_paths : ≥1 test per ACn and per SCoT arm; GUI → Playwright e2e per (journey) AC,
                      selecting by accessible role/name from the spec.
9b INVOKE test-runner
     IN  test_paths ; src_paths ; slice.member_ids as {scope} ; target.md install/build/test commands
     OUT REPORT.md (unit + integration + component, + e2e for GUI)
9c GATE test-gatekeeper
     IN  REPORT.md ; spec_paths (for coverage) ; conventions.md
     OUT verdict_record { phase: test, scope: slice_id, coverage, routing, iteration: n/5 }
      PASS (green + full coverage)         → YOU set slice index_rows.status: reviewed → approved; step 10.
      REJECT → route per triage (§7); each loop returns to the sub-step that re-gates the fix:
        spec bug → spec-writer      : set status reviewed → draft, loop step 7, then step 8 regenerates code.
        code bug → code-implementer : loop step 8 (re-pass the code gate before re-testing).
        test bug → test-writer      : loop step 9a.
      routing: escalate (suite never ran / app won't boot / e2e skipped) → ESCALATE immediately (NOT a budget iteration).
      OVERFLOW(>5) → ESCALATE; stop slice.
```

### Step 10 — Next slice
```
IN   slice_list ; index_rows.status
DO   compute remaining = slices in slice_list not yet approved.
OUT  next_target : if remaining ≠ ∅ → (next slice, → step 7) ; else → (→ step 11)
```

### Step 11 — Final unscoped sweep
```
IN   whole approved project ; target.md test command with NO scope
DO   INVOKE test-runner (whole suite) → INVOKE test-gatekeeper over the whole project.
OUT  REPORT.md (whole suite) ; verdict_record { phase: test, scope: PROJECT }
      regression → route per §7 to the owning slice, re-run its step 9 (bounded by test budget).
      green      → project done.
```

---

## Escalation   `the only human touch-point after stack resolution`
```
ON budget overflow OR routing: escalate:
  STOP that slice (or step 1–4) and report concisely:
    · scope            (slice_id or PLAN)
    · step + iteration vs budget
    · failing verdict_record verbatim + its reasons[] from .sdd/state.md
    · author           last routed to
NEVER silently retry past budget. NEVER advance status on an unresolved scope.
Slices already approved stay approved.
```

## Resume   `from files only`
```
Loop state is reconstructable: each verdict_record carries `iteration: n/<budget>`,
and index_rows.status shows which slices are done.
TO RESUME re-run /sdd-auto:
  skip every approved slice;
  re-enter the first non-approved slice at the step its latest verdict_record implies;
  continue that step's iteration count.
```

## Status transitions   `you make them; gatekeepers never do`
```
step 4 / 7d  analysis PASS              → slice spec index_rows.status: draft → reviewed.
step 9c      test PASS (green + full     → reviewed → approved.
             coverage, implies code PASS)
step 8b      code PASS alone            → advances nothing.
any REJECT                             → status unchanged (never regresses).
```

## Outputs
```
REQUIREMENT.md · PLAN.md · target.md
per slice: index_rows · spec_paths · src_paths · impl_note · test_paths · REPORT.md
.sdd/state.md   append-only verdict_record log
index_rows.status advanced to approved for every completed slice
```
