---
description: Run the whole SDD flow end-to-end — plan → specs → implement → test, automatic gates, human out of the loop; escalate only on budget overflow or an unstated stack.
argument-hint: "<free-text requirement or feature description>"
---

# /sdd-auto — SDD orchestrator

Run full flow for `$ARGUMENTS`, one vertical slice at a time; automatic gates + feedback loops.
- Steps run in order 1 → 9.
- Steps 5–8 repeat once per slice.
- **Main session only orchestrates** — never authors an artifact. It only:
  - invokes agents,
  - reads each `verdict_record`,
  - advances index `status`,
  - routes on REJECT,
  - drives the loop,
  - owns human touch-points.

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
        - own the per-scope CURSOR `.sdd/verdicts/<scope>/_cursor.md` (phase/iteration/verdict_seq); pass iteration+nn to each gate.
        - read each gate's single new verdict by KNOWN path (nn = cursor verdict_seq+1) — never scan .sdd/verdicts/.
        - set index `status` yourself.
        - loop on REJECT.
LAW    - Markdown spec = source of truth (authority).
        - Reuse over repetition (DRY).
        - A red test never makes code authoritative — fix the wrong spec first, then regenerate code.
STATE  Durable state in FILES, never in this conversation: `slice_list` (PLAN.md) · `index_rows.status`
        (the indexes) · the per-scope CURSOR `.sdd/verdicts/<scope>/_cursor.md` (loop control: phase/iteration/verdict_seq).
        The `<nn>-…` verdicts in `.sdd/verdicts/<scope>/` are append-only AUDIT cronaca — the loop NEVER scans them to decide (Loop cursor §); prunable once approved.
        A slice's in-loop output — verdict bodies, agent return payloads, TEST-/REUSE-REPORT text,
        reasons[] threaded between iterations — is per-slice SCRATCH:
        - re-derive, don't recall: re-Read the cursor / re-Glob specs when a step needs a fact; never lean on conversation memory.
        - garbage-collect once the slice is `approved` (step 8) — the cursor stays (it is the resume anchor).
        Every slice boundary = cold start — same file-only reconstruction the Resume section performs.
RULES  Obey .claude/sdd/conventions.md: ids §2 · front-matter §3 · index rows §4 · status/duties §5
        · verdict format §6 · budgets/routing §7 · change policy §8 · roster §9 · topo/slices §12.
HUMAN  Touched twice only: (a) unstated stack, (b) escalation.
```

## Preconditions
```
REQUIRE  requirement_text present in $ARGUMENTS (raw text ok; step 2 refines it).
REQUIRE  resolvable stack:
           IF .sdd/target.md missing AND stack not inferable from requirement_text
           THEN ask the human the single stack question now, capture answer, then run unattended.
                (only prompt besides escalation.)
HAVE     read-only contracts shipped with the tool: conventions.md, scot.md, ui-schema.md.
```

## Data types   *(the typed tokens used in every step's IN / OUT)*

| token | type / structure | origin / note |
|-------|------------------|---------------|
| requirement_text | string | raw $ARGUMENTS |
| current_date | ISO-8601 date | orchestrator supplies to every dated artifact (requirement-analyst + each GATE); subagents have no clock — never invent it |
| stack_decision | { lang, framework, … } | resolved target stack — held in memory until step 3 |
| REQUIREMENT.md | file { raw, refined, req_ids: REQ-* } | authored by requirement-analyst — refined list is a dated changelog |
| PLAN.md | file { entities, Slice plan } | plan + ordered slice list; rewritten per run — on existing-SDD a delta (only NEW/MODIFY + their slices) |
| target.md | file | stack + canonical install/build/test commands |
| slice | { slice_id, member_ids[], depends_on_closure[] } | |
| slice_list | [ slice ] | execution order, authored by plan-architect inside PLAN.md |
| index_rows | rows in per-module `<MOD>.index.md` (+ global `modules.index.md`), each `status: draft\|reviewed\|implemented\|approved` | |
| spec_paths | [ .sdd/specs/**/*.spec.md ] | 5 levels: module/feature/entity/class/UI, incl. MOD-build/MOD-schema |
| REUSE-REPORT.md | file { promoted: SHR-*\|COMP-*, demote_ids[], re_homed[]: {id, old_path → new_path} } | |
| src_paths | [ path ] | only the spec's declared `source:` paths |
| impl_note | .sdd/impl-notes/<MOD-id>/<level>/<id>.impl-notes.md | mirrors the spec's .sdd/specs/<MOD-id>/<level> path + basename |
| install_result | { ok: bool, log } | |
| test_paths | [ tests/** ] | |
| TEST-REPORT | file { failures[], per-test status } | `.sdd/verdicts/<scope>/_test-report.md` — one per scope, overwritten each run |
| cursor | `.sdd/verdicts/<scope>/_cursor.md` { phase, iteration, verdict_seq } | orchestrator-owned loop control state; fixed-size, overwritten each gate (Loop cursor §, conventions §7) |
| verdict_record | `.sdd/verdicts/<scope>/<nn>-<gate-agent>-<scope>-<verdict>.md` { phase, scope, verdict: PASS\|REJECT, reasons[], routing, iteration: n/budget } | one file per gate (audit); read the single new one by KNOWN path (nn = cursor verdict_seq+1) — PASS/REJECT in the filename, open body only on REJECT; never scan |

## How to read a step
```
EXECUTOR MARKERS — who runs the step (always first line of the step body)
  ▶▶ INVOKE <agent>   call an AUTHOR subagent via Task (writes requirement / specs / code / tests)
  ▶▶ GATE   <agent>   call a GATEKEEPER subagent via Task → emits a verdict_record (PASS/REJECT)
  ··  YOU             orchestrator does this directly — NO subagent (control flow, status, human only)

Step TITLE names the ACTION only. The agent (if any) is the ▶▶ line, always visible.
Every step lists IN (tokens consumed) and OUT (tokens produced). After a GATE, act on its
verdict_record via the PASS / REJECT / OVERFLOW branches.
```

## Loop cursor   `the only control state the loop reads — never scan .sdd/verdicts/`
```
PER SCOPE (each slice + PLAN + PROJECT): a tiny file `.sdd/verdicts/<scope>/_cursor.md` (conventions §7):
   { phase: analysis|code|test · iteration: n · verdict_seq: n }
YOU own it (gatekeepers never touch it): fixed-size, OVERWRITTEN, never appended.

AT EVERY GATE:
  1 READ the scope cursor → iteration i, verdict_seq s, phase p (budget B = analysis 3 · code 3 · test 5).
  2 INVOKE the gate with: current_date, iteration=i, budget=B, nn=s+1 (YOU zero-pad nn → `01`,`02`,…).
  3 the gate writes .sdd/verdicts/<scope>/<nn>-<agent>-<scope>-<verdict>.md (audit, stamped `iteration: i/B`) — and nothing else.
  4 READ the outcome from THAT one file's name (nn=s+1 is known); open its body ONLY on REJECT (for reasons[]). NEVER Glob/scan the folder.
  5 WRITE the cursor: verdict_seq:=s+1 (ALWAYS; keep phase+iteration unless a rule below changes them). Then for the DRIVING gate:
       PASS                  → advance index status (§5); on entering the next driving loop set phase:=next, iteration:=1.
       REJECT (i < B)        → iteration:=i+1 (phase unchanged).
       OVERFLOW (i = B)      → ESCALATE; phase+iteration unchanged (verdict_seq already advanced above — so resume never re-issues this nn).
     A NESTED sub-gate (re-analysis/re-code run DURING a code/test fix, §7) → verdict_seq++ ONLY; phase+iteration stay on the driving loop, stamped with the driving i / B.

RESET iteration:=1 (phase as noted; KEEP verdict_seq — monotonic, never reset → new verdicts continue past prior `<nn>`, no collision) on TOP-LEVEL entry to a driving loop:
  - the PLAN gate (run start);
  - a slice's analysis loop on first entry (NEW) or an evolution demote (5a) reached from step 4/8 — NOT from a code/test fix;
  - code on analysis-PASS; test on code-PASS.
  A demote routed from inside the code/test loop (§7 spec-bug) is NESTED, not top-level → do NOT reset.
  → budget independent of `.sdd/verdicts/` size; re-evolving a finished scope starts at iteration 1, old verdicts never erode it.
MISSING cursor (first run / manual prune) → rebuild: iteration:1, verdict_seq = highest <nn> in .sdd/verdicts/<scope>/ (or 0), phase = LEAST-ADVANCED status of the scope's OWN members (exclude read-only depends_on-closure deps): draft→analysis · reviewed→code · implemented→test (ALL own members approved → slice done, skip it); PLAN→analysis; PROJECT→test.
```

---

### Step 1 — Resolve the stack
```
··  YOU   (human touch-point — only the orchestrator may prompt the human)
IN   requirement_text ; target.md (if present)
DO   from target.md → else infer from requirement_text → else ask the human once (Preconditions).
OUT  stack_decision   (plan-architect writes the real target.md in step 3)
```

### Step 2 — Capture the requirement
```
▶▶ INVOKE requirement-analyst
IN   conventions ; requirement_text ; current_date (you hold the date; subagent has no clock) ; existing REQUIREMENT.md (append, never renumber)
OUT  REQUIREMENT.md { raw, refined, req_ids: REQ-* }   — refined list is a dated changelog
```

### Step 3 — Plan the indexes, slices & stack
```
▶▶ INVOKE plan-architect
IN   conventions ; REQUIREMENT.md ; existing .sdd/specs/ + indexes (classify NEW vs existing-SDD) ; target.md (if present) ; scot/ui-schema (forms) ; stack_decision
OUT  target.md ; PLAN.md { entities, ordered Slice plan } — rewritten afresh; on existing-SDD a **delta** (only NEW/MODIFY entities + their slices)   →  orchestrator reads slice_list from here
```

### Step 4 — Gate the plan
```
▶▶ GATE plan-gatekeeper
IN   conventions ; target.md ; REQUIREMENT.md ; PLAN.md ; .sdd/specs/ indexes (existing — ids + depends_on) ; current_date
OUT  verdict_record { phase: analysis, scope: PLAN, iteration: n/3 }
 PASS                              → enter the per-slice loop (step 5) over slice_list.
 REJECT                            → by routing:
        plan-architect      → re-INVOKE plan-architect (step 3) with reasons[]; loop step 4.
        requirement-analyst → requirement itself at fault: re-INVOKE requirement-analyst (step 2),
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
     IN  index_rows of the slice's `MODIFY` members (per the PLAN delta) whose status ∈ {reviewed, implemented, approved}
     OUT those rows → status: draft (§5).
         `NEW` members start at draft; unchanged `depends_on`-closure members stay `approved` (read-only deps — never demoted, never re-worked).
         CURSOR (TOP-LEVEL entry only — from step 4/8, NEW or evolution-MODIFY slice): set the slice cursor phase:analysis, iteration:1, **KEEP verdict_seq** (monotonic; never reset → new verdicts continue past the prior run's `<nn>`, no collision). A spec-bug demote routed from the code/test loop is NESTED → do NOT reset (Loop cursor §). This top-level reset is why a re-evolved slice gets a FULL budget, never eroded by old verdicts (conventions §7).

5b ▶▶ INVOKE spec-writer   (the narrative authority)
     IN  conventions ; scot ; ui-schema ; target.md ; PLAN.md ; REQUIREMENT.md ; the indexes + existing specs ; REUSE-REPORT.md (hand-off edits) ; templates ; slice ; [spec-bug re-INVOKE: + reasons[]]
     OUT index_rows + spec_paths (5 levels incl. MOD-build/MOD-schema), all status: draft

5c ▶▶ INVOKE reuse-analyst
     IN  conventions ; ui-schema ; target.md ; the indexes (modules.index.md + per-module <MOD>.index.md) ; spec_paths (this slice) + existing SHR-*/COMP-* specs
     OUT promoted/re-homed SHR-*/COMP-* specs ; updated index_rows (re-home: old row removed, new row carries new spec path) ; REUSE-REPORT.md { promoted, demote_ids[], re_homed[]: {id, old_path → new_path} }
   ··  YOU   then:
     (1) for each re_homed {old_path → new_path} → `mv old_path new_path` AND move its exact mirror `.impl-notes.md` file (Bash — authors have no move/delete tool);
     (2) for each id in demote_ids → set index_rows.status: draft

5d ▶▶ GATE analysis-gatekeeper   (the only spec-phase blocker)
     IN  conventions ; scot ; ui-schema ; target.md ; REQUIREMENT.md ; the indexes (full depends_on graph) + in-scope spec_paths ; REUSE-REPORT.md ; current_date
     OUT verdict_record { phase: analysis, scope: slice_id, iteration: n/3 }
      PASS         → YOU set slice spec index_rows.status: draft → reviewed; step 6.
      REJECT       → by routing (each re-invoke carries the verdict reasons[]):
                       spec defect → spec-writer (5b) · duplication → reuse-analyst (5c);
                       re-run reuse-analyst if spec_paths changed; loop step 5d.
      OVERFLOW(>3) → ESCALATE; stop slice.
```

### Step 6 — Implement the slice   `[code budget 3]`   `status: reviewed → implemented`
```
6a ▶▶ INVOKE code-implementer   (minimal diffs over rewrites)
     IN  conventions ; target.md (idioms map) ; scot/ui-schema (per kind) ; one reviewed spec + every spec it references by id (depends_on + each CALL/COMP, via Glob .sdd/specs/**/<id>.spec.md — bind against their real interfaces) + their impl-notes ; existing src_paths ; [code-bug re-INVOKE: + reasons[]]
     DO  minimal Edit by default; regenerate only by exception (§8).
     OUT src_paths (the spec's declared `source:` paths) ; impl_note

6b ··  YOU   run canonical `install` → install_result
   ▶▶ GATE code-gatekeeper
     IN  conventions ; scot ; ui-schema (gui) ; target.md (build/lint) ; the gated spec(s) + impl_note + src_paths + the indexes ; install_result ; current_date
     OUT verdict_record { phase: code, scope: slice_id, iteration: n/3 }
      PASS         → YOU set slice index_rows.status: reviewed → implemented; step 7.
      REJECT       → by routing (each re-invoke carries the verdict reasons[]):
        code-implementer → minimal diff (6a); loop step 6b.
        spec-writer      → re-validate spec: demote status → draft, re-run
                           spec-writer (5b) → reuse-analyst (5c, if changed) → analysis-gatekeeper (5d)
                           → re-advance to reviewed, then resume code; loop step 6b.
      OVERFLOW(>3) → ESCALATE; stop slice.
```

### Step 7 — Test the slice   `[test budget 5]`   `status: implemented → approved`
```
7a ▶▶ INVOKE test-writer   (independent oracle — NEVER reads src/ or .sdd/impl-notes/; it DOES read its contracts: target.md §2 idioms map, conventions, scot/ui-schema)
     IN  conventions ; scot ; ui-schema ; target.md (§2 idioms map + tests/ layout) ; the indexes + in-scope spec_paths (behavioral sections only) + every spec they reference by id (depends_on + CALL/COMP, via Glob .sdd/specs/**/<id>.spec.md — to derive stubs) ; [test-bug re-INVOKE: + reasons[] + prior test_paths]   (NEVER src/ or impl-notes — the firewall)
     OUT test_paths : ≥1 test per ACn and per SCoT arm; GUI → Playwright e2e per (journey) AC,
                      selecting by accessible role/name from the spec.

7b ▶▶ INVOKE test-runner
     IN  conventions (§14 report) ; target.md (install/build/test commands + layout) ; test_paths + src_paths ; slice.member_ids as {scope}
     OUT `.sdd/verdicts/<slice>/_test-report.md` (unit + integration + component, + e2e for GUI)

7c ▶▶ GATE test-gatekeeper
     IN  conventions ; scot ; ui-schema (gui) ; target.md ; `.sdd/verdicts/<slice>/_test-report.md` ; the indexes + in-scope spec_paths (coverage) ; tests/** ; src/** (read-only, triage only) ; current_date
     OUT verdict_record { phase: test, scope: slice_id, coverage, routing, iteration: n/5 }
      PASS (green + full coverage) → YOU set slice index_rows.status: implemented → approved; step 8.
      REJECT → route per triage (§7); each loop returns to the sub-step that re-gates the fix:
        spec bug → spec-writer (5b)      : re-INVOKE with verdict reasons[]; demote status → draft, loop step 5, then step 6 regenerates code.
        code bug → code-implementer (6a) : re-INVOKE with verdict reasons[]; loop step 6 (re-pass the code gate before re-testing).
        test bug → test-writer (7a)      : re-INVOKE with verdict reasons[] + the offending test_paths (minimal edit); loop step 7a.
      routing: escalate (suite never ran / app won't boot / e2e skipped) → ESCALATE immediately (NOT a budget iteration).
      OVERFLOW(>5) → ESCALATE; stop slice.
```

### Step 8 — Next slice
```
··  YOU
IN   slice_list ; index_rows.status   (re-read from PLAN.md + the indexes — not from memory of prior slices)
DO   (1) GARBAGE-COLLECT the just-approved slice — drop its scratch from working context (verdict bodies,
         agent return payloads, TEST-/REUSE-REPORT text, reasons[]): spent, the durable record is on disk.
     (2) compute remaining = slices in slice_list whose index_rows.status ≠ approved.
OUT  next_target :
       if remaining ≠ ∅ → next slice, entered as a COLD START — reconstruct from files per the STATE law, → step 5 ;
       else            → → step 9
```

### Step 9 — Final unscoped sweep
```
▶▶ INVOKE test-runner   (whole suite, NO scope)
▶▶ GATE   test-gatekeeper (whole project)
IN   conventions ; target.md (test command, NO scope) ; whole approved project (indexes + all spec_paths + tests/** + src/**) ; current_date
OUT  `.sdd/verdicts/PROJECT/_test-report.md` (whole suite) ; verdict_record { phase: test, scope: PROJECT }
      regression → route per §7 to the owning slice, re-run its step 7 (bounded by test budget).
      green      → project done.
```

---

## Escalation   `the only human touch-point after stack resolution`
```
ON budget overflow OR routing: escalate:
  STOP that slice (or step 1–4); report concisely:
    · scope            (slice_id or PLAN)
    · step + iteration vs budget
    · failing verdict_record verbatim + its reasons[] from its `.sdd/verdicts/` file
    · author           last routed to
NEVER silently retry past budget. NEVER advance status on an unresolved scope.
Slices already approved stay approved.
```

## Resume   `from files only`
```
Loop state is reconstructable:
  - READ the scope CURSOR `.sdd/verdicts/<scope>/_cursor.md` → phase + iteration + verdict_seq (O(1); no scan).
  - index_rows.status shows which slices are done.
  - cursor missing → rebuild per the Loop cursor § (iteration:1; verdict_seq = highest <nn> in the folder; phase = least-advanced OWN-member status; PLAN→analysis, PROJECT→test).
TO RESUME re-run /sdd-auto:
  - skip every approved slice;
  - re-enter the first non-approved slice at the phase its cursor implies;
  - continue that loop's iteration from the cursor.
The per-slice loop performs this SAME file-only reconstruction at every slice boundary (the STATE law),
not only after an interruption — so a long run never depends on accumulated conversation history.
```

## Status transitions   `you make them; gatekeepers never do`
```
step 5d      analysis PASS             → slice spec index_rows.status: draft → reviewed.
step 6b      code PASS                 → reviewed → implemented.
step 7c      test PASS (green + full   → implemented → approved.
             coverage)
any REJECT                             → status unchanged (never regresses).
```

## Outputs
```
REQUIREMENT.md · PLAN.md (incl. Slice plan) · target.md
per slice: index_rows · spec_paths · src_paths · impl_note · test_paths · `.sdd/verdicts/<slice>/_test-report.md`
.sdd/verdicts/<scope>/_cursor.md  per-scope loop cursor (control state; overwritten)
.sdd/verdicts/<scope>/<nn>-…      per-scope append-only verdict log — audit cronaca (prunable; never scanned by the loop)
index_rows.status advanced to approved for every completed slice
```
