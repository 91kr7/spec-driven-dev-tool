---
description: Mode 3 — generate source from approved specs and pass the code gate
argument-hint: "<optional: a spec id, feature id, or slice; default = all reviewed specs>"
---

# /sdd-implement — Mode 3 (Implement)

Turn `reviewed` specs into source code and drive the code gate to PASS. This is
mode 3 of the 5-mode flow (manual, human-in-control): it sits between
`/sdd-specify` (mode 2, which leaves specs at `reviewed`) and `/sdd-test`
(mode 4, which adds green tests and promotes to `approved`). This command writes
`src/` and impl-notes only — it does **not** set `approved`.

Two cross-cutting values bind every step: **Markdown is the source of truth
(authority); reuse over repetition (DRY).** Read `.claude/sdd/conventions.md` (esp. §5
separation of duties, §7 budgets/routing, §8 change policy, §9 roster, §12
topological order) before acting; it is the single source of truth.

## Preconditions
- `.claude/sdd/conventions.md`, `.claude/sdd/scot.md`, `.claude/sdd/ui-schema.md` exist (canonical contracts).
- `.sdd/target.md` is complete (stack/architecture + canonical build/test/run commands and any budget overrides).
- The target specs in scope are `status: reviewed` in their index row (analysis gate passed). Specs still at `draft` are out of scope — finish `/sdd-specify` for them first.
- `.sdd/state.md` exists (append-only verdict log); `.sdd/impl-notes/` is writable (created on demand).

## Steps (you, the main session, perform these)

1. **Resolve the work set.** From `$ARGUMENTS`:
   - empty → every spec whose index row is `status: reviewed`, across all level indexes;
   - a `FEAT-…`/`MOD-…` (or "slice") → that entity plus its `depends_on` closure, restricted to specs that are `reviewed`;
   - a single `CLS-`/`ENT-`/`COMP-`/`SHR-`/`MOD-` id → just that spec (its `reviewed` `depends_on` must already be implemented or stubbable from an `interface`).
   Read indexes first (lazy loading); open only the spec rows you need.

2. **Order the work set topologically** by `depends_on` (dependencies first), per §12. On a dependency **cycle**, schedule the `interface`/`contract` specs first so implementations can depend on the interface (stubs are auto-derived from the interface meanwhile), then the concrete specs. This ordering is the loop's iteration order.

3. **For each entity in order (or for the single sliced entity):** invoke **`code-implementer`** via the Task tool, passing only file paths/ids: the spec path (`specs/<level>/<id>.spec.md`), its `.sdd/impl-notes/<id>.md` (if present), and the declared `source:` paths. Per §8 it **creates** the new file(s) or applies a **minimal diff** to existing ones (never a whole-file rewrite for a small change), writes a traceability header into each source file, records concretizations (library/idiom/edge-case choices) in `.sdd/impl-notes/<id>.md`, and **never edits the spec**.

4. **Provision deps once the build manifest exists** (after `MOD-build` is implemented): run the canonical `install` from `.sdd/target.md` so the `code-gatekeeper`'s read-only compile/type-check actually runs and catches syntax/type errors **at the code gate**, not only later at the test phase. (Before the manifest exists the compile check is best-effort and the test phase gives the definitive compile.) Then **invoke `code-gatekeeper`** via the Task tool for the just-implemented entity (pass the spec id, its `source:` paths, and its impl-notes path). It judges code ≡ spec read-only and **appends one verdict record** to `.sdd/state.md` (`phase: code`).

5. **Read the latest `.sdd/state.md` record** for that scope (per §6 format) and decide:
   - **PASS** → record stands in `state.md` (do **not** advance the index `status` yet — see "Status transitions"); proceed to the next entity in the topological order.
   - **REJECT** → route by the verdict's `routing` field, incrementing the iteration against the **code budget (3)**:
     - `routing: code-implementer` (code bug) → re-invoke `code-implementer` with the verdict `reasons` and the same paths; apply a minimal diff to converge.
     - `routing: spec-writer` (the gate blames the spec — spec is wrong/ambiguous, MD is authority) → the spec must be **re-validated before code can be trusted** (§5): YOU demote the entity `reviewed → draft`, then re-run the spec-phase agents **inline** — `spec-writer` (with the reasons) → `reuse-analyst` (if its specs changed) → `analysis-gatekeeper` — and on its PASS re-advance `draft → reviewed`; only then resume `code-implementer` for it. (Do **not** re-run the whole `/sdd-specify` command here — its human-approval step does not belong mid-implementation.) Spec rework does **not** consume the code budget.
   - Re-invoke `code-gatekeeper` after each fix and re-read the latest record.

6. **Budget overflow.** If the code loop for one entity reaches **3** REJECTs without a PASS, **ESCALATE to the human**: stop that entity's loop, print a concise summary (the entity id, the persistent blocking reasons from `state.md`, and whether the failures point at code or spec), and do not silently continue. Other independent entities already PASS-ed may proceed.

7. **Iterate** through the whole topological work set until every entity in scope has a latest `code` verdict of PASS in `.sdd/state.md` (or has been escalated).

## Status transitions
- This command **does not** advance any index `status`. `reviewed → approved`
  requires green tests + a test-gate PASS, which only `/sdd-test` (mode 4) can
  grant (§5: `approved` = implemented + code-gate PASS + tests green + test-gate
  PASS). Implemented entities stay at `status: reviewed`.
- The only persistent record this command produces about gating is the appended
  **code-gate PASS/REJECT** in `.sdd/state.md`; `/sdd-test` reads those forward.
- Per §5 separation of duties, gatekeepers never touch `status`; the main session
  advances it — but here there is no advance to make until the test phase.

## Outputs
- `src/…` — created or minimally-diffed source files, each carrying a spec
  traceability header (e.g. `// spec: CLS-regCtrl … — specs/classes/CLS-regCtrl.spec.md`).
- `.sdd/impl-notes/<id>.md` — implementer-owned concretization notes (not part of
  the gated spec).
- `.sdd/state.md` — one appended `code`-phase verdict record per gate run.

## Next command (manual path)
Run **`/sdd-test`** (mode 4) on the same scope: it writes spec-derived tests,
runs them, triages failures, and on full green promotes each entity
`reviewed → approved`. If any entity escalated here, resolve it (fix the code or
re-spec via `/sdd-specify`) and re-run `/sdd-implement` for that id before moving on.
