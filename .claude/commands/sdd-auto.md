---
description: Run the whole SDD flow end-to-end — plan → specs → implement → test, automatic gates, human out of the loop; escalate only on budget overflow or an unstated stack.
argument-hint: "<free-text requirement or feature description>"
---

# /sdd-auto — SDD orchestrator

Run the full flow for `$ARGUMENTS`, one vertical slice at a time, with automatic gates and feedback loops.
Steps run in order 1 → 11; steps 7–10 repeat once per slice.

```
DATAFLOW
  $ARGUMENTS
    └▶ 1 stack ─▶ 2 REQUIREMENT.md ─▶ 3 PLAN.md+target.md ─▶ 4 plan gate
        └▶ 5 topo order ─▶ 6 slice list
            └▶ ┌── per slice ──────────────────────────────────────────┐
               │ 7 specs ─▶ 8 src ─▶ 9 tests ─▶ (gates) ─▶ 10 next slice │
               └────────────────────────────────────────────────────────┘
                └▶ 11 whole-suite sweep ─▶ done
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
REQUIRE  requirement present in $ARGUMENTS (raw text ok; step 2 refines it).
REQUIRE  resolvable stack:
           IF .sdd/target.md missing AND stack not inferable from $ARGUMENTS
           THEN ask the human the single stack question now, capture answer, then run unattended.
           (this is the only prompt besides escalation.)
HAVE     read-only contracts shipped with the tool: conventions.md, scot.md, ui-schema.md.
```

> Every step lists its **IN** (data consumed) and **OUT** (data produced). Gates produce a verdict
> record in `.sdd/state.md`; only **you** act on it via the PASS / REJECT / OVERFLOW branches.

---

### Step 1 — Resolve the stack
```
IN   $ARGUMENTS ; .sdd/target.md (if present)
DO   take stack from target.md → else infer from $ARGUMENTS → else ask the human once (Preconditions).
OUT  a resolved stack decision (held for step 3).
```

### Step 2 — Capture the requirement   *(YOU write it — not plan-architect)*
```
IN   $ARGUMENTS (raw free text)
DO   write & refine the requirement; assign a stable REQ-001, REQ-002, … per atomic, testable item.
OUT  requirements/REQUIREMENT.md (raw + refined, with REQ-* ids)
```

### Step 3 — INVOKE plan-architect
```
IN   requirements/REQUIREMENT.md ; resolved stack (step 1)
DO   architect derives the stack file and the plan of indexes/specs to create or modify.
OUT  .sdd/target.md ; plan/PLAN.md
```

### Step 4 — GATE plan-gatekeeper
```
IN   plan/PLAN.md ; requirements/REQUIREMENT.md ; conventions.md
OUT  .sdd/state.md record { phase: analysis, scope: PLAN, verdict, reasons, iteration: n/3 }
      PASS                              → go to step 5.
      REJECT                            → re-INVOKE plan-architect (step 3) with the reasons; loop step 4.
      OVERFLOW(>3) OR routing: escalate → ESCALATE; stop.   (escalate = unresolved <…> placeholder)
```

### Step 5 — Compute topological order
```
IN   plan/PLAN.md ; index `depends_on` edges
DO   topo-sort features/modules (§12); break any cycle interface-first.
OUT  an ordered list of features/modules
```

### Step 6 — Build the slice list
```
IN   topo order (step 5) ; depends_on closures
DO   one vertical slice = a feature/module + its depends_on closure;
     order slices so each slice's deps live in an already-approved earlier slice.
OUT  the ordered slice list (drives the loop below)
```

```
╔══════════════════════ LOOP: for each slice, in order — steps 7 → 10 ══════════════════════╗
║ AFTER EVERY GATE: read the latest .sdd/state.md record for the slice scope,                ║
║                   then advance the affected index rows' `status` yourself (§5).            ║
╚════════════════════════════════════════════════════════════════════════════════════════════╝
```

### Step 7 — Specify   `[analysis budget 3]`   `status: draft → reviewed`
```
7a demote (feature-evolution only)
     IN  in-scope entity rows currently at reviewed/approved
     OUT those rows set back to draft (§5)
7b INVOKE spec-writer
     IN  the current slice ; plan/PLAN.md ; .sdd/target.md ; conventions.md
     OUT index rows + specs/**/*.spec.md (4 levels + MOD-build), all status: draft
7c INVOKE reuse-analyst
     IN  the slice's specs ; existing SHR-* / COMP-* specs
     OUT deduped/promoted SHR-* / COMP-* specs ; specs/REUSE-REPORT.md
     THEN read REUSE-REPORT.md → demote to draft every id under its `Demote-for-re-gate:` heading.
7d GATE analysis-gatekeeper   (the only spec-phase blocker)
     IN  the slice's specs ; specs/REUSE-REPORT.md ; requirements/REQUIREMENT.md ; conventions.md
     OUT .sdd/state.md record { phase: analysis, scope: slice, verdict, reasons, iteration: n/3 }
      PASS         → YOU advance slice spec rows draft → reviewed; go to step 8.
      REJECT       → route: spec defect → spec-writer (7b) · duplication → reuse-analyst (7c);
                     re-run reuse-analyst if specs changed; loop step 7d.
      OVERFLOW(>3) → ESCALATE; stop slice.
```

### Step 8 — Implement   `[code budget 3]`   *(writes src/ + impl-notes only)*
```
8a INVOKE code-implementer   (per spec, in depends_on order)
     IN  one reviewed spec ; .sdd/target.md ; existing src/
     DO  minimal Edit by default; regenerate only by exception (§8).
     OUT src/ at the spec's declared `source:` paths ; .sdd/impl-notes/<id>.md
8b run canonical `install` (makes the read-only compile check effective), THEN GATE code-gatekeeper
     IN  src/ ; the slice's specs ; .sdd/impl-notes/* ; install result
     OUT .sdd/state.md record { phase: code, scope: slice, verdict, reasons, iteration: n/3 }
      PASS         → go to step 9.   (code PASS advances no status.)
      REJECT:
        routing: code-implementer → minimal diff (8a); loop step 8b.
        routing: spec-writer      → re-validate spec: demote reviewed → draft, re-run
                                    spec-writer → reuse-analyst (if changed) → analysis-gatekeeper
                                    → re-advance to reviewed, then resume code; loop step 8b.
      OVERFLOW(>3) → ESCALATE; stop slice.
```

### Step 9 — Test   `[test budget 5]`   `status: reviewed → approved`
```
9a INVOKE test-writer   (independent oracle — NEVER reads src/ or impl-notes/)
     IN  the slice's specs ONLY
     OUT tests/** : ≥1 test per ACn and per SCoT arm; GUI → Playwright e2e per (journey) AC,
                    selecting by accessible role/name from the spec.
9b INVOKE test-runner
     IN  tests/** ; src/ ; slice ids as {scope} ; canonical commands from .sdd/target.md
     OUT tests/REPORT.md (structured failures; unit + integration + component, + e2e for GUI)
9c GATE test-gatekeeper
     IN  tests/REPORT.md ; the slice's specs (for coverage) ; conventions.md
     OUT .sdd/state.md record { phase: test, scope: slice, verdict, coverage, triage routing, iteration: n/5 }
      PASS (green + full coverage)         → YOU advance slice rows reviewed → approved; go to step 10.
      REJECT → route per triage (§7); each loop returns to the sub-step that re-gates the fix:
        spec bug → spec-writer      : demote reviewed → draft, loop step 7, then step 8 regenerates code.
        code bug → code-implementer : loop step 8 (re-pass the code gate before re-testing).
        test bug → test-writer      : loop step 9a.
      routing: escalate (suite never ran / app won't boot / e2e skipped) → ESCALATE immediately (NOT a budget iteration).
      OVERFLOW(>5) → ESCALATE; stop slice.
```

### Step 10 — Next slice
```
IN   remaining slice list
DO   if slices remain, return to step 7 for the next slice; else go to step 11.
OUT  (control flow only)
```

### Step 11 — Final unscoped sweep
```
IN   the whole approved project ; canonical commands with NO scope
DO   INVOKE test-runner (whole suite) → INVOKE test-gatekeeper over the whole project.
OUT  tests/REPORT.md (whole suite) ; .sdd/state.md whole-project verdict
      regression → route per §7 to the owning slice, re-run its step 9 (bounded by test budget).
      green      → project done.
```

---

## Escalation   `the only human touch-point after stack resolution`
```
ON budget overflow OR routing: escalate:
  STOP that slice (or step 1–4) and report concisely:
    · scope            (slice id or PLAN)
    · step + iteration vs budget
    · failing verdict  verbatim + its reasons from .sdd/state.md
    · author           last routed to
NEVER silently retry past budget. NEVER advance status on an unresolved scope.
Slices already approved stay approved.
```

## Resume   `from files only`
```
Loop state is reconstructable: each .sdd/state.md record carries `iteration: n/<budget>`,
and index `status` shows which slices are done.
TO RESUME re-run /sdd-auto:
  skip every approved slice;
  re-enter the first non-approved slice at the step its latest record implies;
  continue that step's iteration count.
```

## Status transitions   `you make them; gatekeepers never do`
```
step 4/7d analysis PASS                → slice spec rows draft → reviewed.
step 9c   test PASS (green + full       → reviewed → approved.
          coverage, implies code PASS)
step 8b   code PASS alone              → advances nothing.
any REJECT                            → status unchanged (never regresses).
```

## Outputs
```
requirements/REQUIREMENT.md · plan/PLAN.md · .sdd/target.md
per slice: index rows · specs/**/*.spec.md · src/** · .sdd/impl-notes/<id>.md · tests/** · tests/REPORT.md
.sdd/state.md   append-only verdict log
index `status`  advanced to approved for every completed slice
```
