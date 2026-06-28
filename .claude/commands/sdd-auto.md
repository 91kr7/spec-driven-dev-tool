---
description: Run the whole SDD flow end-to-end — plan → specs → implement → test, automatic gates, human out of the loop; escalate only on budget overflow or an unstated stack.
argument-hint: "<free-text requirement or feature description>"
---

# /sdd-auto — SDD orchestrator

Run the full flow for `$ARGUMENTS`, one vertical slice at a time, with automatic gates and feedback loops.
Steps run in order 1 → 9; steps 5–8 repeat once per slice.
**The main session only orchestrates** — it never authors an artifact: it invokes agents, reads each
`verdict_record`, advances index `status`, routes on REJECT, drives the loop, and owns the human touch-points.

```
DATAFLOW
  requirement_text
    └▶ 1 stack_decision ─▶ 2 REQUIREMENT.md ─▶ 3 PLAN.md + target.md + slice_list ─▶ 4 verdict_record
        └▶ ┌── per slice (from slice_list) ─────────────────────────────────────┐
           │ 5 spec_paths ─▶ 6 src_paths ─▶ 7 test_paths ─▶ (gates) ─▶ 8 next     │
           └─────────────────────────────────────────────────────────────────────┘
            └▶ 9 whole-suite verdict_record ─▶ done
```

```
ROLE   you (main session) = orchestrator; subagents do ALL authoring/judging via Task.
        you read verdicts from .sdd/verdicts/ (one file per gate; latest = highest <nn> via Glob),
        set index `status` yourself, loop on REJECT.
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

| token | type / structure | origin / note |
|-------|------------------|---------------|
| requirement_text | string | raw $ARGUMENTS |
| current_date | ISO-8601 date | supplied by the orchestrator — the subagent has no clock |
| stack_decision | { lang, framework, … } | resolved target stack — held in memory until step 3 |
| REQUIREMENT.md | file { raw, refined, req_ids: REQ-* } | authored by requirement-analyst — refined list is a dated changelog |
| PLAN.md | file { entities, Slice plan } | plan + ordered slice list; rewritten per run — on existing-SDD a delta (only NEW/MODIFY + their slices) |
| target.md | file | stack + canonical install/build/test commands |
| slice | { slice_id, member_ids[], depends_on_closure[] } | |
| slice_list | [ slice ] | execution order, authored by plan-architect inside PLAN.md |
| index_rows | rows in level indexes, each with `status: draft\|reviewed\|approved` | |
| spec_paths | [ specs/**/*.spec.md ] | 5 levels: module/feature/entity/class/UI, incl. MOD-build/MOD-schema |
| REUSE-REPORT.md | file { promoted: SHR-*\|COMP-*, demote_ids[] } | |
| src_paths | [ path ] | only the spec's declared `source:` paths |
| impl_note | impl-notes/<level>/<id>.impl-notes.md | mirrors the spec's specs/ folder + basename |
| install_result | { ok: bool, log } | |
| test_paths | [ tests/** ] | |
| REPORT.md | file { failures[], per-test status } | tests/REPORT.md |
| verdict_record | .sdd/verdicts/<nn>-<gate-agent>-<scope>-<verdict>.md { phase, scope, verdict: PASS\|REJECT, reasons[], routing, iteration: n/budget } | one file per gate; read the latest by Glob — PASS/REJECT is in the filename, open the body only on REJECT |

## How to read a step
```
EXECUTOR MARKERS — who runs the step (always the first line of the step body)
  ▶▶ INVOKE <agent>   call an AUTHOR subagent via Task (writes requirement / specs / code / tests)
  ▶▶ GATE   <agent>   call a GATEKEEPER subagent via Task → emits a verdict_record (PASS/REJECT)
  ··  YOU             the orchestrator does this directly — NO subagent (control flow, status, human only)

The step TITLE names the ACTION only. The agent (if any) is the ▶▶ line, so it is always visible.
Every step lists IN (tokens consumed) and OUT (tokens produced). After a GATE you act on its
verdict_record via the PASS / REJECT / OVERFLOW branches.
```

---

### Step 1 — Resolve the stack
```
··  YOU   (human touch-point — only the orchestrator may prompt the human)
IN   requirement_text ; target.md (if present)
DO   from target.md → else infer from requirement_text → else ask the human once (Preconditions).
OUT  stack_decision   (plan-architect writes the actual target.md in step 3)
```

### Step 2 — Capture the requirement
```
▶▶ INVOKE requirement-analyst
IN   requirement_text ; current_date   (you hold the date; the subagent has no clock)
OUT  REQUIREMENT.md { raw, refined, req_ids: REQ-* }   — refined list is a dated changelog
```

### Step 3 — Plan the indexes, slices & stack
```
▶▶ INVOKE plan-architect
IN   REQUIREMENT.md ; stack_decision
OUT  target.md ; PLAN.md { entities, ordered Slice plan } — rewritten afresh; on existing-SDD a **delta** (only NEW/MODIFY entities + the slices that contain them)   →  the orchestrator reads slice_list from here
```

### Step 4 — Gate the plan
```
▶▶ GATE plan-gatekeeper
IN   PLAN.md ; target.md ; REQUIREMENT.md ; conventions.md ; specs/ (existing — id stability)
OUT  verdict_record { phase: analysis, scope: PLAN, iteration: n/3 }
 PASS                              → enter the per-slice loop (step 5) over slice_list.
 REJECT                            → by routing:
        plan-architect      → re-INVOKE plan-architect (step 3) with reasons[]; loop step 4.
        requirement-analyst → the requirement itself is at fault: re-INVOKE requirement-analyst (step 2),
                              then re-plan (step 3); loop step 4.
 OVERFLOW(>3) OR routing: escalate → ESCALATE; stop.   (escalate = unresolved <…> placeholder)
```

```
╔══════════════════════ LOOP: for each slice in slice_list (from PLAN.md), in order — steps 5 → 8 ══════════════════════╗
║ AFTER EVERY GATE: read the latest verdict_record for scope == slice.slice_id,                                         ║
║                   then advance the affected index_rows.status yourself (§5).                                          ║
╚══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╝
```

### Step 5 — Specify the slice   `[analysis budget 3]`   `status: draft → reviewed`
```
5a ··  YOU   demote (feature-evolution only) — ONLY the entities this change rewrites
     IN  index_rows of the slice's `MODIFY` members (per the PLAN delta) whose status ∈ {reviewed, approved}
     OUT those rows → status: draft (§5).  `NEW` members start at draft; unchanged `depends_on`-closure members stay `approved` (read-only deps — never demoted, never re-worked).

5b ▶▶ INVOKE spec-writer
     IN  slice ; PLAN.md ; target.md ; conventions.md
     OUT index_rows + spec_paths (5 levels incl. MOD-build/MOD-schema), all status: draft

5c ▶▶ INVOKE reuse-analyst
     IN  spec_paths (this slice) ; existing SHR-* / COMP-* specs
     OUT promoted SHR-*/COMP-* specs ; REUSE-REPORT.md
   ··  YOU   then: for each id in REUSE-REPORT.md.demote_ids → set index_rows.status: draft

5d ▶▶ GATE analysis-gatekeeper   (the only spec-phase blocker)
     IN  spec_paths ; REUSE-REPORT.md ; REQUIREMENT.md ; conventions.md
     OUT verdict_record { phase: analysis, scope: slice_id, iteration: n/3 }
      PASS         → YOU set slice spec index_rows.status: draft → reviewed; step 6.
      REJECT       → by routing: spec defect → spec-writer (5b) · duplication → reuse-analyst (5c);
                     re-run reuse-analyst if spec_paths changed; loop step 5d.
      OVERFLOW(>3) → ESCALATE; stop slice.
```

### Step 6 — Implement the slice   `[code budget 3]`   *(writes src/ + impl-notes only)*
```
6a ▶▶ INVOKE code-implementer   (per spec, in depends_on order)
     IN  one reviewed spec ; target.md ; existing src_paths
     DO  minimal Edit by default; regenerate only by exception (§8).
     OUT src_paths (the spec's declared `source:` paths) ; impl_note

6b ··  YOU   run canonical `install` → install_result
   ▶▶ GATE code-gatekeeper
     IN  src_paths ; spec_paths (this slice) ; impl_note ; install_result
     OUT verdict_record { phase: code, scope: slice_id, iteration: n/3 }
      PASS         → step 7.   (code PASS advances no status.)
      REJECT       → by routing:
        code-implementer → minimal diff (6a); loop step 6b.
        spec-writer      → re-validate spec: set status reviewed → draft, re-run
                           spec-writer (5b) → reuse-analyst (5c, if changed) → analysis-gatekeeper (5d)
                           → re-advance to reviewed, then resume code; loop step 6b.
      OVERFLOW(>3) → ESCALATE; stop slice.
```

### Step 7 — Test the slice   `[test budget 5]`   `status: reviewed → approved`
```
7a ▶▶ INVOKE test-writer   (independent oracle — NEVER reads src/ or impl-notes/)
     IN  spec_paths (this slice) ONLY
     OUT test_paths : ≥1 test per ACn and per SCoT arm; GUI → Playwright e2e per (journey) AC,
                      selecting by accessible role/name from the spec.

7b ▶▶ INVOKE test-runner
     IN  test_paths ; src_paths ; slice.member_ids as {scope} ; target.md install/build/test commands
     OUT REPORT.md (unit + integration + component, + e2e for GUI)

7c ▶▶ GATE test-gatekeeper
     IN  REPORT.md ; spec_paths (for coverage) ; conventions.md
     OUT verdict_record { phase: test, scope: slice_id, coverage, routing, iteration: n/5 }
      PASS (green + full coverage) → YOU set slice index_rows.status: reviewed → approved; step 8.
      REJECT → route per triage (§7); each loop returns to the sub-step that re-gates the fix:
        spec bug → spec-writer (5b)      : set status reviewed → draft, loop step 5, then step 6 regenerates code.
        code bug → code-implementer (6a) : loop step 6 (re-pass the code gate before re-testing).
        test bug → test-writer (7a)      : loop step 7a.
      routing: escalate (suite never ran / app won't boot / e2e skipped) → ESCALATE immediately (NOT a budget iteration).
      OVERFLOW(>5) → ESCALATE; stop slice.
```

### Step 8 — Next slice
```
··  YOU
IN   slice_list ; index_rows.status
DO   compute remaining = slices in slice_list not yet approved.
OUT  next_target : if remaining ≠ ∅ → (next slice, → step 5) ; else → (→ step 9)
```

### Step 9 — Final unscoped sweep
```
▶▶ INVOKE test-runner   (whole suite, NO scope)
▶▶ GATE   test-gatekeeper (whole project)
IN   whole approved project ; target.md test command with NO scope
OUT  REPORT.md (whole suite) ; verdict_record { phase: test, scope: PROJECT }
      regression → route per §7 to the owning slice, re-run its step 7 (bounded by test budget).
      green      → project done.
```

---

## Escalation   `the only human touch-point after stack resolution`
```
ON budget overflow OR routing: escalate:
  STOP that slice (or step 1–4) and report concisely:
    · scope            (slice_id or PLAN)
    · step + iteration vs budget
    · failing verdict_record verbatim + its reasons[] from its `.sdd/verdicts/` file
    · author           last routed to
NEVER silently retry past budget. NEVER advance status on an unresolved scope.
Slices already approved stay approved.
```

## Resume   `from files only`
```
Loop state is reconstructable: Glob `.sdd/verdicts/` (highest `<nn>` per scope = that scope's latest
verdict_record, carrying `iteration: n/<budget>`); index_rows.status shows which slices are done.
TO RESUME re-run /sdd-auto:
  skip every approved slice;
  re-enter the first non-approved slice at the step its latest verdict_record implies;
  continue that step's iteration count.
```

## Status transitions   `you make them; gatekeepers never do`
```
step 5d      analysis PASS             → slice spec index_rows.status: draft → reviewed.
step 7c      test PASS (green + full     → reviewed → approved.
             coverage, implies code PASS)
step 6b      code PASS alone            → advances nothing.
any REJECT                             → status unchanged (never regresses).
```

## Outputs
```
REQUIREMENT.md · PLAN.md (incl. Slice plan) · target.md
per slice: index_rows · spec_paths · src_paths · impl_note · test_paths · REPORT.md
.sdd/verdicts/  append-only verdict log — one file per gate (no whole-file rewrite)
index_rows.status advanced to approved for every completed slice
```
