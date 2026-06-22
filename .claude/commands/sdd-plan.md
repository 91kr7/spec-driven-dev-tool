---
description: Mode 1 (Plan) — turn a requirement into a plan of indexes/specs; no specs, no code.
argument-hint: "<free-text requirement, or a feature description>"
---

# /sdd-plan — Mode 1 (Plan)

Turn the user's requirement into a **plan** of how to create/modify the indexes and
specs — the first of the 5 modes, fully **manual / human-in-control**. It writes
**no specs and no code**; it stops at a plan the human must approve before
`/sdd-specify`. Authority and DRY hold throughout: Markdown is the source of truth
(authority); reuse over repetition (DRY).

## Preconditions
- A requirement is available — either in `requirements/REQUIREMENT.md` already, or
  passed inline as `$ARGUMENTS`. If neither exists, ask the human for the
  requirement and stop.
- `.sdd/conventions.md` exists (canonical contract; read by every agent).
- This command does **not** require any prior plan, target, or specs — it is the
  entry point and may bootstrap `.sdd/target.md`, `.sdd/scot.md`, `.sdd/ui-schema.md`.

## Steps (you, the main session, perform these)
1. **Capture the requirement.** If `$ARGUMENTS` is non-empty, write/refine
   `requirements/REQUIREMENT.md` so it captures (raw + refined) the requirement
   from `$ARGUMENTS`. If `$ARGUMENTS` is empty and `requirements/REQUIREMENT.md`
   already holds a requirement, use it as-is. If both are empty, ESCALATE: ask the
   human for the requirement and stop.
2. **Invoke `plan-architect`** via Task. Pass only paths: the input
   `requirements/REQUIREMENT.md` and the contracts under `.sdd/` it must obey.
   It will:
   - derive the stack/architecture into `.sdd/target.md` (including canonical
     build/test/run commands and any budget overrides);
   - author `.sdd/scot.md` and `.sdd/ui-schema.md` **only if they are absent**
     (new project); on an existing SDD project it leaves them untouched;
   - write the plan to `plan/PLAN.md` (which indexes/specs to create or modify, at
     which levels, in `depends_on` order — discover-before-create, reuse over new).
3. **Handle an unstated stack.** If `plan-architect` reports that the stack is
   unstated and it cannot derive `.sdd/target.md`, relay its exact question to the
   human and **stop** until the human answers; then re-invoke `plan-architect` with
   the answer. (This is human-in-control input, not an iteration against the
   budget.)
4. **Invoke `plan-gatekeeper`** via Task, passing `requirements/REQUIREMENT.md`,
   `plan/PLAN.md`, and `.sdd/target.md`. It JUDGES only and appends one verdict
   record to `.sdd/state.md`.
5. **Read the latest verdict** for this scope from `.sdd/state.md`
   (`phase: analysis`).
6. **Decide:**
   - **PASS** → the plan is ready. Tell the human: the plan in `plan/PLAN.md` and
     the derived `.sdd/target.md` are ready for approval; the next command is
     `/sdd-specify`. **Stop here** — do not advance to writing specs.
   - **REJECT** → re-invoke `plan-architect` with the verdict's blocking reasons
     (routing is `plan-architect` for the plan phase), then go back to step 4.
     Count each REJECT→re-author cycle against the **analysis budget (3)**.
7. **Budget overflow** (a 4th REJECT would be needed) → **ESCALATE** to the human
   with a concise summary of the unresolved blocking reasons from `.sdd/state.md`;
   stop the loop.

## Status transitions
- **None.** No index `status` is advanced here — `draft`/`reviewed`/`approved` are
  set only once specs exist (first `draft` is written by `/sdd-specify`). This mode
  produces only the plan and the derived contracts.

## Outputs
- `requirements/REQUIREMENT.md` — captured/refined requirement.
- `plan/PLAN.md` — the plan of indexes/specs to create or modify.
- `.sdd/target.md` — derived stack/architecture + canonical build/test/run commands.
- `.sdd/scot.md`, `.sdd/ui-schema.md` — authored **only if previously absent**.
- `.sdd/state.md` — one or more `plan-gatekeeper` verdict records (append-only).

## Next command (manual path)
On PASS and after the human approves the plan, proceed to **`/sdd-specify`**
(Mode 2 — Plan approval), which captures the approval and drives
`spec-writer` → `reuse-analyst` → `analysis-gatekeeper` to write specs and advance
status to `reviewed`. This command never proceeds to specs on its own.
