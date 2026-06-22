---
description: Helper — print a per-entity lifecycle dashboard (draft/reviewed/approved + latest verdict + resume point) from the indexes and state.md (read-only).
argument-hint: "<optional: a slice/feature id (FEAT-001) or module id (MOD-api) to scope the dashboard; default = whole project>"
---

# /sdd-status — Helper (read-only)

Render a **status dashboard** of the project: where every entity stands in its
lifecycle (`draft → reviewed → approved`), the latest gate verdict for each scope,
what is currently **blocked** (and on whom), and **where `/sdd-auto` would resume**.
This is a **helper**, not one of the 5 flow modes — like `/sdd-trace` it changes
nothing, advances no status, runs no gate, and invokes no subagent. Use it to
supervise a run, decide the next manual command, or resume an interrupted `/sdd-auto`.
Mindset: **Markdown is the source of truth (authority); reuse over repetition (DRY).**

It complements `/sdd-trace` (which reconstructs the REQ→FEAT→CLS→SOURCE→TEST chain and
coverage gaps): `/sdd-status` answers *"where does each thing stand and what's next?"*

## Preconditions

- `specs/indexes/` exists with the per-level indexes (the `status` column is the
  canonical lifecycle home — `.sdd/conventions.md` §5).
- `.sdd/state.md` exists (the append-only verdict log — §6 format).
- `$ARGUMENTS`, if given, is a single scope id (`FEAT-…` / `MOD-…` / `CLS-…` / `ENT-…`
  / `COMP-…` / `SHR-…`); empty → whole project.
- **Read-only:** it writes nothing — not specs, code, tests, `.sdd/state.md`, or any
  index `status`. It invokes no subagent.

## Steps (you, the main session, perform these)

1. **Resolve scope.** Empty → whole project. A `FEAT-`/`MOD-` id → that entity plus its
   `depends_on` closure (the slice). Another id → just that entity. If the id is in no
   index, print a one-line "unknown id" notice (listing the indexes searched) and stop.

2. **Read the indexes** (lazy — only the scope). From each per-level index
   (`modules`, `features`, `model`, `classes`, `ui-components`; honor the §1 per-module
   split if the project uses it) collect each in-scope row's `id`, `name`, `module`,
   `depends_on`, and **`status`**. Remember `SHR-*` rows live in `classes.index` (§4).

3. **Read `.sdd/state.md`** and, for each entity/scope, find the **latest** verdict
   record (the log is append-only, so the last matching record wins): its `phase`
   (`analysis | code | test`), `verdict` (`PASS | REJECT`), `iteration: n/budget`, and
   — on REJECT — its `routing` and `reasons`.

4. **Derive, per entity:** the lifecycle `status`, the phase it last cleared or last
   failed, and — if the latest verdict is **REJECT** — that it is **blocked**, the
   author it is **routed** to, and its iteration vs budget (analysis 3 / code 3 /
   test 5 — §7, honoring any `.sdd/target.md` §4 overrides). Mark an **escalation**
   when iteration is at/over budget.

5. **Compute the resume point** (matches `/sdd-auto`'s *Resume* note): order the
   in-scope slices by `depends_on`; the resume point is the **first non-`approved`**
   slice, at the sub-phase its latest `.sdd/state.md` record implies — *Specify* if it
   is `draft` (or its last verdict is `analysis`), *Implement* if it is `reviewed` with
   a `code` verdict, *Test* if a `code` PASS exists but no `test` PASS — continuing that
   phase's iteration count (not restarting the budget).

6. **Print** (output only — write nothing):
   - a **table**, one row per in-scope entity:
     `id | name | module | status | last phase | last verdict | iter/budget | blocked-on | depends_on`
   - a **summary**: counts per `status` (draft / reviewed / approved / total); which
     slices/features are fully `approved` vs in-progress; the **blocked** entities with
     their `routing` + the one-line reason; any **escalations** (budget overflow).
   - the **resume line**: the slice + sub-phase + iteration where `/sdd-auto` would
     pick up — or `all approved — nothing to resume`.
   Surface (but never fix) any inconsistency it **observes**: an index `status` that
   contradicts the latest verdict (e.g. `approved` while the latest `test` verdict is
   REJECT), or an entity at `reviewed`/`approved` with no matching verdict in
   `.sdd/state.md`.

## Status transitions

- **None.** `/sdd-status` is read-only: it advances no index `status`
  (`draft`/`reviewed`/`approved` are owned by the flow commands per
  `.sdd/conventions.md` §5) and appends no verdict to `.sdd/state.md`. It only *reads
  and reports* what the indexes and the log already record.

## Outputs

- A printed-to-session **dashboard**: the per-entity table, the summary, and the
  resume line. **No files are written.** If the scope has no entities yet (e.g. before
  `/sdd-specify` has run), print `no specs yet — run /sdd-plan then /sdd-specify`.

## Next command (manual path)

`/sdd-status` is an on-demand helper with no fixed successor. Typical uses:

- **Supervise / decide the next step** — read the dashboard, then run the command for
  the next unblocked work: `/sdd-implement` for a `reviewed` slice, `/sdd-test` for an
  implemented one.
- **Unblock** — for a blocked entity, act on its `blocked-on` routing: re-run the
  command that owns that phase for the id (§7 routing).
- **Resume an interrupted `/sdd-auto`** — start at the printed resume point.
- **Pair with `/sdd-trace <id>`** to see that node's REQ→…→TEST chain and coverage gaps.
