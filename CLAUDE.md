# CLAUDE.md — how to maintain this tool

**This repo IS the tool.** `.claude/` is the entire product: a spec-driven-development (SDD)
orchestrator built entirely from Markdown. There is no application code — the Markdown files
ARE the deliverable. Edit and commit **only** under `.claude/` (and this file). Anything else on
disk — `.sdd/`, `src/`, `tests/`, field-test dirs — is generated/throwaway, never the product.

## Language

- **Replies to the user: clear, natural Italian.** Write as a native speaker would — never a
  word-for-word calque of an English sentence. Plain, idiomatic phrasing; if a literal translation
  reads awkwardly, rephrase it the way an Italian would actually say it.
- **All repository content stays in English** — every `.md` (conventions, agents, templates, this
  file), source code, and commit message. Only the chat replies to the user are in Italian.

---

## Where each kind of rule lives (the doc architecture)

Know the home for a fact before you write it. (Authoritative file list: [conventions §1](.claude/sdd/conventions.md#1-file-and-folder-layout);
agent roster: [§9](.claude/sdd/conventions.md#9-agent-roster-and-isolation-matrix) — this table is the *maintenance* view, not a second copy of those.)

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
> **Everywhere else, link to it (e.g. `[§6](../sdd/conventions.md#6-verdict-records)`) — never restate it.**

Before writing any normative sentence (a "must/never", a format, an id scheme, a lifecycle, a
routing/verdict/resume rule) in ANY file, ask:

1. Does this concept already have a home? → **link it** (e.g. `[§6](../sdd/conventions.md#6-verdict-records)`), do not re-describe it.
2. No home yet? → put it in the right home **first**, then link from the point of use.

A restated rule is not documentation — it is a **desync bug waiting to happen**: a later change
updates one copy and the others silently drift. (Most state-machine bugs this tool has fought came
from exactly that.) The clause *"on conflict this file wins"* is a symptom, not a licence to copy.

**Pointer format.** A pointer is a real link to the target section's **own heading anchor** — the
slug the renderer auto-generates from the title — never a hand-written `<a id>` (those resolve only
on GitHub web; IntelliJ, VS Code, and the GitHub mobile app ignore them, so the jump silently dies).
A real one: `[§6](../sdd/conventions.md#6-verdict-records)` — general shape
`[§<n>](<relative-path>#<n>-<lowercased-title-with-hyphens>)`, where the slug is the section
**number + lowercased title, spaces → hyphens** (`<…>` = fill-in, not literal text). For the slug to resolve
**identically across renderers, keep section titles short and symbol-free**: no `&` (write "and"),
no em-dashes, `()` subtitles, or `` `code` `` inside a heading — each of those slugs differently per
tool. Renaming a title changes its slug, so update the links pointing to it (the leading **number**
is stable; run the link-checker after any title edit). Relative path is from the *linking* file:
`../sdd/conventions.md#…` from an agent, `../conventions.md#…` from a template. (The GitHub mobile
app may not scroll to any in-page anchor — a client limit no format fixes; the link still opens the
right file, and `§N` names the section.)

### The ONLY permitted duplication
1. The `MINDSET` two-values line that **[conventions §11](.claude/sdd/conventions.md#11-role-header) explicitly mandates** in every agent file.
2. At most a **one-line inline echo** of a *safety-critical* rule inside an agent/command prompt,
   when subagent reliability genuinely needs the rule in its own context — and then only if it is
   **marked `(canonical: §N)`**, is never the full rule, and never drifts from its home.

Anything else that states the same thing twice is a defect to collapse into a pointer.

### Who reads these files — an AI agent executing the prompt

Every file here is loaded **as an agent's operating prompt**, usually run by a **single subagent**
acting on its own file. Write for that reader — it also draws the line for the echo above:

- A **pointer carries the authority + the full rule**: the agent has `conventions.md` in context and
  follows the link (e.g. `[§6](../sdd/conventions.md#6-verdict-records)`) for the complete statement. That is why re-narrating a rule is pure waste.
- Where an agent must **act on** a rule (a gatekeeper's accept/reject test, an author's
  what-to-produce), keep the **one-line discriminating criterion inline** (marked `(canonical: §N)`).
  `orphan = empty requirements:` beats *"verify requirements match §13"* — a gate told only to
  "conform to §N" judges less reliably than one handed the exact trigger.
- So the cut is **restatement vs. actionable criterion**: collapse the paragraph that re-explains a
  rule; keep the terse trigger the agent branches on. Never flatten a decisive check to a bare "see §N".
- A **template's filled example is content to imitate, not a rule** — never gut it to a pointer (a
  `Purpose` that reads "see §2" teaches an author nothing).

---

## When you add or change a shared rule

1. Edit the **canonical home only**.
2. **`rg "<key phrase>" .claude/` first** — find every echo. Update the home; convert stray copies
   to real section links (or delete them). Leave only the two permitted echoes above.
3. **Smell test:** a single-concept change that forces edits in **>2 files** means you're editing
   *copies*, not the source. Stop and collapse to a pointer first, then make the change once.
