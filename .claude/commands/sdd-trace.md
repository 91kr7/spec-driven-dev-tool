---
description: Reconstruct & print the REQUIREMENT→FEATURE→CLASS/ENTITY/COMPONENT→SOURCE→TEST chain (read-only).
argument-hint: "<optional: a requirement id (REQ-001), feature id (FEAT-001), or spec id (CLS-…, ENT-…, COMP-…); default = whole project>"
---

# /sdd-trace — Helper (read-only)

Reconstructs and prints the full traceability chain
**REQUIREMENT → FEATURE → CLASS/ENTITY/COMPONENT → SOURCE → TEST** on demand, from
the indexes and spec front-matter (per `.claude/sdd/conventions.md` §13 — there is **no**
separate traceability file; it is rebuilt every time). This is a **helper**, not
one of the 5 flow modes: it changes nothing, advances no status, runs no gate. It
exists for audits and impact analysis **before** a change. Mindset throughout:
**Markdown is the source of truth (authority); reuse over repetition (DRY).**

## Preconditions

- `specs/indexes/` exists with at least `features.index.md`, `classes.index.md`,
  `model.index.md`, and (when the project has UI) `ui-components.index.md`.
- `requirements/REQUIREMENT.md` exists (the back-link target for `requirements:`).
- The argument `$ARGUMENTS`, if given, is a single focus id:
  - a **requirement** id (`REQ-001`) — trace everything serving that requirement;
  - a **feature** id (`FEAT-001`) — trace that one feature's slice;
  - an **entity/class/component** id (`ENT-…`, `CLS-…`, `COMP-…`, `SHR-…`) — trace
    that node up to its requirements and down to its source + tests.
  - If `$ARGUMENTS` is empty → focus = the **whole project**.
- This command is **read-only**: it never writes specs, code, tests, `state.md`,
  or any index `status`. It does not invoke any subagent (no authoring, no gate).

## Steps (you, the main session, perform these)

1. **Resolve focus.** Parse `$ARGUMENTS`. Determine its id prefix to pick the
   entry level (`REQ-` / `FEAT-` / `ENT-`/`CLS-`/`COMP-`/`SHR-`). Empty → whole
   project. If the id is not found in any index, print a one-line "unknown id"
   notice listing the indexes searched, and stop (still writing nothing).
2. **Read the indexes** with Read/Glob/Grep (lazy — open only what the focus needs):
   - `specs/indexes/features.index.md` — the feature→module partitioning and each
     feature's `depends_on` (which classes/entities it serves).
   - `specs/indexes/classes.index.md`, `specs/indexes/model.index.md`,
     `specs/indexes/ui-components.index.md` — for the class/entity/component rows,
     their `module`, `depends_on`, the **derived `source`** column, and `status`.
   - Honor the scaling option: if fine-grained indexes are split per module under
     `specs/indexes/<level>/<MOD-id>.index.md`, read those too.
3. **Read each relevant spec's front-matter** (not the whole body) to get the
   authoritative links: `requirements:` (back-link up to `REQ-…`) and `source:`
   (the single authoritative spec→source mapping; the index `source` column is
   derived from it — if they disagree, the **spec front-matter wins** and you note
   the drift). Use `requirements:` to group nodes under each requirement.
4. **Cross-reference the source files' traceability headers.** For each path in a
   spec's `source:`, Grep the file for its `// spec: <id> … — specs/…` header
   (comment syntax per target language) and confirm it points back to the same
   spec id. A missing or mismatched header is a **traceability gap** to highlight.
5. **Cross-reference the tests' coverage ids.** Grep `tests/` for the fully-
   qualified coverage ids `<spec-id>::<function>#<arm-id>` and the `ACn` ids each
   behavioral spec declares (per `.claude/sdd/scot.md` §7). Map every covering test back
   to the node it covers. For `gui`/structural specs, map tests to their `ACn`.
6. **Compute coverage gaps.** For each behavioral node, derive its expected
   coverage set (every SCoT arm id + every `ACn`) and diff it against the coverage
   ids actually found in `tests/`. Any uncovered **arm** or **AC** is a gap.
7. **Print the chain** (output only — write nothing). For the focus or the whole
   project, render a tree **and** a flat table linking each layer:
   - **Requirement** (`REQ-…`, one line from `requirements/REQUIREMENT.md`)
     - → **Feature(s)** serving it (`FEAT-…` + module)
       - → **Class / Entity / Component** nodes (`CLS-…`/`ENT-…`/`COMP-…`/`SHR-…`,
         with index `status`)
         - → **Source** file(s) from `source:` (flag header mismatches)
         - → **Test(s)** covering it (by coverage id / `ACn`)
   - Highlight, inline and in a final summary: **AC coverage gaps**, **SCoT branch-
     arm coverage gaps**, orphan nodes (a spec with no requirement back-link),
     dangling `source:` paths (no file / no header), and `source`-column drift
     between an index and the owning spec.
8. **Render unresolved links explicitly** rather than silently dropping them
   (e.g. a feature whose `depends_on` names a class absent from the index, or a
   node whose `source:` is `[]` because it is purely compositional — that is
   expected for a compositional feature and is **not** a gap; note it as such).

## Status transitions

- **None.** `/sdd-trace` is read-only: it advances no index `status`
  (`draft`/`reviewed`/`approved` are owned by the flow commands per
  `.claude/sdd/conventions.md` §5) and appends no verdict to `.sdd/state.md`. It only
  *reads and reports* the `status` already recorded in each index row.

## Outputs

- A printed-to-session traceability report for the focus (or whole project):
  1. a **tree** REQ → FEAT → CLS/ENT/COMP → SOURCE → TEST, and
  2. a **flat table** (`requirement | feature | node | module | source | tests | status | gaps`),
  3. a **gap summary**: uncovered ACs, uncovered SCoT branch arms, orphan nodes,
     dangling/headerless `source:` paths, and index↔spec `source` drift.
- **No files are written.** If nothing is found for a given focus, a single
  "no traceability found for `<focus>`" line is printed.

## Next command (manual path)

`/sdd-trace` is a helper invoked **on demand**; it has no fixed successor and is
not part of the 5-mode sequence. Typical uses:

- **Impact analysis before a change** — run it on the target id, read which
  features/sources/tests are affected, then proceed to `/sdd-implement` (mode 3)
  or `/sdd-test` (mode 4) for the touched slice.
- **Audit after a slice** — run it (whole-project or per-feature) to confirm every
  requirement is served and fully covered before moving the next slice through the
  flow (or before letting `/sdd-auto` continue, mode 5).
