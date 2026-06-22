---
description: Mode 2 — capture plan approval, then write indexes + specs and pass the analysis gate.
argument-hint: "none (acts on the approved plan) or a slice id (e.g. FEAT-001)"
---

# /sdd-specify — Mode 2 (Plan approval → write specs)

Manual, human-in-control mode (mode 2 of the 5-mode flow; modes 1–4 are manual, mode 5 is automatic). This command represents the human's **approval of `plan/PLAN.md`**, then turns the approved plan into the spec corpus — the per-level indexes and the `*.spec.md` files (4 levels + `MOD-build`) — and drives them through the analysis gate until they are internally sound. It advances each spec from `draft` to `reviewed`. It writes no code and no tests.

Authority reminder (per `.sdd/conventions.md`): **Markdown is the source of truth (authority); reuse over repetition (DRY).** You (the main session) are the orchestrator: you invoke subagents via the Task tool, read their verdicts from `.sdd/state.md`, and own all index `status` edits. Subagents do not spawn subagents; hand-off is **file-based** — pass only paths/ids.

## Preconditions
- `plan/PLAN.md` exists and **passed the plan gate** (latest `plan-gatekeeper` record for the plan in `.sdd/state.md` is `PASS`). If the plan never passed, stop and tell the human to run `/sdd-plan` first.
- `.sdd/target.md` is complete (stack/architecture/framework + canonical build/test/run commands).
- The canonical contracts exist: `.sdd/conventions.md`, `.sdd/scot.md`, `.sdd/ui-schema.md`.
- `requirements/REQUIREMENT.md` exists (specs back-link to its requirement ids).
- **Optional arg** `$ARGUMENTS`: if a slice id (e.g. `FEAT-001`) is given, scope spec-writing to that feature and its `depends_on` closure; if empty, act on the whole approved plan.

## Steps (you, the main session, perform these)

1. **Capture human approval (gate of this mode).** Confirm the human approves `plan/PLAN.md`. This command *represents* that approval — **do not proceed without an explicit go-ahead**. If the human has changes, stop and send them back to `/sdd-plan`.

2. **Write indexes + specs.** Invoke `spec-writer` via Task. Pass the input paths/ids it needs: `plan/PLAN.md`, `requirements/REQUIREMENT.md`, the scope (`$ARGUMENTS` or "full plan"), and the canonical contracts (`.sdd/conventions.md`, `.sdd/scot.md`, `.sdd/ui-schema.md`, `.sdd/target.md`). It writes the per-level indexes under `specs/indexes/` and the `*.spec.md` files (modules incl. `MOD-build`, features, model, classes, ui-components) — **every new index row starts `status: draft`** and each spec's front-matter `status:` mirrors it. For **feature evolution** on an existing SDD project, before an entity already at `reviewed`/`approved` has its spec modified, YOU demote its index row to `draft` (§5) so the changed spec re-flows through the gate.

3. **Dedupe + promote shared specs.** Invoke `reuse-analyst` via Task (pure author — judges nothing, writes no verdict). Pass `specs/` and `.sdd/conventions.md`. It removes duplication, promotes recurring abstractions into `specs/shared/` (`SHR-*`) and `specs/ui-components/` (`COMP-*`), rewires `depends_on`/component-tree references to the promoted ids, and writes `specs/REUSE-REPORT.md`. Promoted specs are also `status: draft`.

4. **Run the analysis gate.** Invoke `analysis-gatekeeper` via Task, passing the scope ids and `specs/`. It JUDGES only and appends one verdict record to `.sdd/state.md`. Read the **latest** `analysis-gatekeeper` record for this scope from `.sdd/state.md`.

5. **Decide from the verdict:**
   - **PASS →** YOU advance every in-scope spec's index `status` from `draft` to `reviewed` (edit the index rows; mirror `status: reviewed` into each spec's front-matter — the gatekeeper never touches `status`). Then tell the human the specs are ready to review and that the next command is **`/sdd-implement`**.
   - **REJECT →** read the record's `routing` field and re-invoke the routed author with the verdict's `reasons` (and the offending spec/AC/branch ids): `routing: spec-writer` → re-invoke `spec-writer`; `routing: reuse-analyst` → re-invoke `reuse-analyst`. After the author rewrites, re-run `reuse-analyst` if specs changed, then go back to step 4.

6. **Track the loop against the analysis budget (3).** Count each REJECT→re-author→re-judge cycle as one iteration. On the **4th** would-be iteration (budget overflow), **ESCALATE to the human** with a concise summary — the unresolved blocking reasons, the spec ids involved, and the iteration count — and **stop the loop**. Do not advance status on overflow.

## Status transitions
- **`reviewed`/`approved` → `draft`** (feature evolution) — before re-writing an entity whose spec already passed a gate, YOU demote it so it re-earns `reviewed` through the analysis gate (§5).
- **`draft → reviewed`** on an analysis-gate **PASS**, for every in-scope spec's index row. **You (the command) write this**; the `analysis-gatekeeper` only judges and records *why* in `.sdd/state.md`. `reviewed → approved` is owned later by `/sdd-test`, not here.

## Outputs
- `specs/indexes/*.index.md` — the per-level indexes (rows authored, `status: draft` → `reviewed` on PASS).
- `specs/modules/`, `specs/features/`, `specs/model/`, `specs/classes/`, `specs/ui-components/`, `specs/shared/` — the `*.spec.md` corpus (incl. `MOD-build`).
- `specs/REUSE-REPORT.md` — the reuse-analyst's dedupe/promotion report.
- `.sdd/state.md` — appended analysis-gate verdict record(s) (written by the gatekeeper).

## Next command (manual path)
On PASS and status advanced to `reviewed`: hand control back to the human to review the specs, then run **`/sdd-implement`** (Mode 3) to generate source from the reviewed specs in `depends_on` order. On budget overflow: control stays with the human (resolve the escalation, then re-run `/sdd-specify`).
