# CLAUDE.md — how to maintain this tool

**This repo IS the tool.** `.claude/` is the entire product: a spec-driven-development (SDD)
orchestrator built entirely from Markdown. There is no application code — the Markdown files
ARE the deliverable. Edit and commit **only** under `.claude/` (and this file). Anything else on
disk — `.sdd/`, `src/`, `tests/`, field-test dirs — is generated/throwaway, never the product.

---

## Where each kind of rule lives (the doc architecture)

Know the home for a fact before you write it. (Authoritative file list: [conventions §1](.claude/sdd/conventions.md#s1);
agent roster: [§9](.claude/sdd/conventions.md#s9) — this table is the *maintenance* view, not a second copy of those.)

| File(s) | Holds | Role when writing |
|---|---|---|
| `.claude/sdd/conventions.md` | THE canonical contract — every cross-cutting rule, standard, format, id scheme, lifecycle | **single source of truth** ("on conflict this file wins") |
| `.claude/sdd/scot.md`, `ui-schema.md` | the two sibling grammars (behavioral / UI) conventions defers to | canonical **only** for their own domain |
| `.claude/commands/sdd-auto.md` | the orchestration PROCEDURE — steps, dataflow, control flow | *executes* rules; must **not redefine** them |
| `.claude/agents/*.md` | per-role subagent prompts (role · mission · mindset · non-goals) | thin; **reference** the contract |
| `.claude/sdd/templates/*.md` | the FORMS a new spec is copied from | shape, not rules |

---

## THE RULE — Single Source of Truth (this tool's own DRY, turned on itself)

The tool's first value is *"Reuse over repetition (DRY)"*. Its own construction MUST obey it.

> **Every rule / standard / format / vocabulary has exactly ONE canonical home.**
> **Everywhere else, link to it (`[§N](path#sN)`) — never restate it.**

Before writing any normative sentence (a "must/never", a format, an id scheme, a lifecycle, a
routing/verdict/resume rule) in ANY file, ask:

1. Does this concept already have a home? → **link it** (`[§N](path#sN)`), do not re-describe it.
2. No home yet? → put it in the right home **first**, then link from the point of use.

A restated rule is not documentation — it is a **desync bug waiting to happen**: a later change
updates one copy and the others silently drift. (Most state-machine bugs this tool has fought came
from exactly that.) The clause *"on conflict this file wins"* is a symptom, not a licence to copy.

**Pointer format.** A pointer is a real link, not bare text: `[§N](relative/path#sN)`. Every numbered
section of `conventions.md` / `scot.md` / `ui-schema.md` carries a stable `<a id="sN"></a>` anchor
(decoupled from the title, so a link survives a section rename). Do **not** rely on heading-text
auto-anchors — they break silently on any title edit (a fresh desync). Relative path is from the
*linking* file: `../sdd/conventions.md#s6` from an agent, `../conventions.md#s6` from a template.

### The ONLY permitted duplication
1. The `MINDSET` two-values line that **[conventions §11](.claude/sdd/conventions.md#s11) explicitly mandates** in every agent file.
2. At most a **one-line inline echo** of a *safety-critical* rule inside an agent/command prompt,
   when subagent reliability genuinely needs the rule in its own context — and then only if it is
   **marked `(canonical: §N)`**, is never the full rule, and never drifts from its home.

Anything else that states the same thing twice is a defect to collapse into a pointer.

---

## When you add or change a shared rule

1. Edit the **canonical home only**.
2. **`rg "<key phrase>" .claude/` first** — find every echo. Update the home; convert stray copies
   to `[§N](path#sN)` links (or delete them). Leave only the two permitted echoes above.
3. **Smell test:** a single-concept change that forces edits in **>2 files** means you're editing
   *copies*, not the source. Stop and collapse to a pointer first, then make the change once.

## When you change status / routing / resume (the state machine)

Simulate a **crash-resume for every backward route** before committing. Keep the two concerns
separate — **WHERE** to re-enter = own-member `status` (kept truthful by the demote) · **WHICH**
reasons = the **latest verdict** (by `current_ts` timestamp; [conventions §6](.claude/sdd/conventions.md#s6)/[§7](.claude/sdd/conventions.md#s7)). Express recovery
as a mechanical rule over file fields, never as prose a human must interpret.
