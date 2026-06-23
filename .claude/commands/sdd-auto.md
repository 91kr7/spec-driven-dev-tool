---
description: Run the whole SDD flow end-to-end — plan → specs → implement → test, automatic gates, human out of the loop; escalate only on budget overflow or an unstated stack.
argument-hint: "<free-text requirement or feature description>"
---

# /sdd-auto — SDD orchestrator

Run the full flow for `$ARGUMENTS`, one vertical slice at a time, with automatic gates and feedback loops.

```
INPUT:  $ARGUMENTS (free-text requirement)
FLOW:   Phase A Plan ─▶ Phase B Slice list ─▶ Phase C [ Specify ▶ Implement ▶ Test ] ×each slice ─▶ Phase D Final sweep
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
REQUIRE  requirement present in $ARGUMENTS (raw text ok; Plan refines it).
REQUIRE  resolvable stack:
           IF .sdd/target.md missing AND stack not inferable from $ARGUMENTS
           THEN ask the human the single stack question now, capture answer, then run unattended.
           (this is the only prompt besides escalation.)
HAVE     read-only contracts shipped with the tool: conventions.md, scot.md, ui-schema.md.
```

---

## Phase A — Plan   `[analysis budget 3]`
```
A1  RESOLVE stack          → ask human once iff unresolvable (see Preconditions).
A2  CAPTURE requirement    → YOU write requirements/REQUIREMENT.md (raw + refined),
                             one stable REQ-001, REQ-002, … per atomic, testable requirement.
                             (plan-architect does NOT write requirements/.)
A3  INVOKE plan-architect  → writes/extends .sdd/target.md + plan/PLAN.md.
A4  GATE   plan-gatekeeper → read latest `phase: analysis` record, scope PLAN.
      PASS                              → Phase B.
      REJECT                            → re-invoke plan-architect with reasons; loop A4.
      OVERFLOW(>3) OR routing: escalate → ESCALATE; stop.   (escalate = unresolved <…> placeholder)
```

## Phase B — Build the slice list
```
B1  COMPUTE depends_on topological order (§12); break any cycle interface-first.
B2  FORM slice list: one vertical slice = a feature/module + its depends_on closure.
                     order slices so each slice's deps live in an already-approved earlier slice.
```

## Phase C — Per-slice loop   `for each slice, in order`
```
AFTER EVERY GATE: read the latest record for the slice scope, then advance the affected
                  index rows' `status` yourself (§5).
```

### C7 — Specify   `[analysis budget 3]`   `status: draft → reviewed`
```
a  IF feature-evolution: demote any in-scope entity at reviewed/approved → draft (§5).
b  INVOKE spec-writer    → index rows + specs for the slice (4 levels + MOD-build), status: draft.
c  INVOKE reuse-analyst  → dedupe/promote SHR-* / COMP-*.
                           then read specs/REUSE-REPORT.md: demote to draft every id
                           listed under its `Demote-for-re-gate:` heading.
d  GATE analysis-gatekeeper   (the only spec-phase blocker) → read latest verdict.
      PASS         → advance slice spec rows draft → reviewed; go C8.
      REJECT       → route: spec defect → spec-writer · duplication → reuse-analyst;
                     re-run reuse-analyst if specs changed; loop C7d.
      OVERFLOW(>3) → ESCALATE; stop slice.
```

### C8 — Implement   `[code budget 3]`   `writes src/ + impl-notes only`
```
a  FOR each spec in depends_on order: INVOKE code-implementer
        → minimal Edit by default; regenerate only by exception (§8);
          writes only declared `source:` paths + impl-notes/<id>.md.
b  ONCE MOD-build exists: run canonical `install` (makes the read-only compile check effective),
        then GATE code-gatekeeper (read-only) → judge code ≡ spec.
c  READ latest code verdict.
      PASS         → go C9.   (code PASS advances no status.)
      REJECT:
        routing: code-implementer → minimal diff; loop C8b.
        routing: spec-writer      → re-validate spec: demote reviewed → draft,
                                     re-run spec-writer → reuse-analyst (if changed) → analysis-gatekeeper
                                     → re-advance to reviewed, then resume code; loop C8b.
      OVERFLOW(>3) → ESCALATE; stop slice.
```

### C9 — Test   `[test budget 5]`   `status: reviewed → approved`
```
a  INVOKE test-writer  (independent oracle — never reads src/ or impl-notes/):
        ≥1 test per ACn and per SCoT arm; GUI project → Playwright e2e per (journey) AC,
        selecting by accessible role/name from the spec.
b  INVOKE test-runner  with slice ids as {scope}
        → run canonical commands filtered by {scope} (unit + integration + component, + e2e for GUI);
          writes tests/REPORT.md.
c  GATE test-gatekeeper → verify coverage (incl. e2e journeys) + triage each failure.
d  READ latest test verdict.
      PASS (green + full coverage) → advance slice rows reviewed → approved; go C10.
      REJECT → route per triage (§7); each loop returns to the sub-phase that re-gates the fix:
        spec bug → spec-writer      : demote reviewed → draft, loop C7 (re-spec → reuse → analysis → reviewed), then C8 regenerates code.
        code bug → code-implementer : loop C8 (re-pass the code gate before re-testing).
        test bug → test-writer      : loop C9b.
      routing: escalate (suite never ran / app won't boot / e2e skipped) → ESCALATE immediately (NOT a budget iteration).
      OVERFLOW(>5) → ESCALATE; stop slice.
```

### C10 — Next slice
```
REPEAT Phase C until no slices remain.
```

## Phase D — Final unscoped sweep
```
D11  WHEN every slice is approved:
       INVOKE test-runner with NO scope (whole suite) + test-gatekeeper over the whole project.
       regression → route per §7 to the owning slice, re-run its C9 (bounded by test budget).
       green      → project done.
```

---

## Escalation   `the only human touch-point after stack resolution`
```
ON budget overflow OR routing: escalate:
  STOP that slice (or Phase A) and report concisely:
    · scope            (slice id or PLAN)
    · phase + iteration vs budget
    · failing verdict  verbatim + its reasons from .sdd/state.md
    · author           last routed to
NEVER silently retry past budget. NEVER advance status on an unresolved scope.
Slices already approved stay approved.
```

## Resume   `from files only`
```
Loop state is reconstructable: each .sdd/state.md record carries `iteration: <n>/<budget>`,
and index `status` shows which slices are done.
TO RESUME re-run /sdd-auto:
  skip every approved slice;
  re-enter the first non-approved slice at the sub-phase its latest record implies;
  continue that phase's iteration count.
```

## Status transitions   `you make them; gatekeepers never do`
```
analysis PASS                         → slice spec rows draft → reviewed.
test PASS (green + full coverage,      → reviewed → approved.
           implies a prior code PASS)
code PASS alone                       → advances nothing.
any REJECT                            → status unchanged (never regresses).
```

## Outputs
```
requirements/REQUIREMENT.md · plan/PLAN.md · .sdd/target.md
per slice: index rows · specs/**/*.spec.md · src/** · .sdd/impl-notes/<id>.md · tests/** · tests/REPORT.md
.sdd/state.md   append-only verdict log
index `status`  advanced to approved for every completed slice
```
