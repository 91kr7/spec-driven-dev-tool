# SDD Workflow — Handoff & Decision History

A document to **resume the work** in a future session. It summarizes what this is about, the current state, and — above all — **every decision made and why**, so whoever reopens the topic (you or an assistant) can pick up without redoing the whole journey.

> Note: automatic export of the raw transcript was not available in this session, so this is a curated summary (more useful than a dump for resuming).

---

## How to resume (quick start)

To start again, give Claude/Claude Code this context:
> "I'm designing a **meta-prompt** for Claude Code that builds an agentic Spec-Driven Development workflow. The prompt is in `Prompt-SDD-for-ClaudeCode.md`, the explanation in `SDD-Workflow-Technical-Document.md`, the decision history in `SDD-Handoff-and-History.md`. I'd like to [continue / change …]."

The three reference files:
1. **`Prompt-SDD-for-ClaudeCode.md`** — the final prompt to paste into Claude Code (in English).
2. **`SDD-Workflow-Technical-Document.md`** — how the workflow works (in English).
3. **`SDD-Handoff-and-History.md`** — this document.

---

## Project goal

The aim is not to *run* a workflow, but to **write the prompt** that asks Claude Code to **build** the agent system (subagents in `.claude/agents/`, slash commands in `.claude/commands/`, templates, etc.). So the main deliverable is a **meta-prompt**.

Core principle: **Markdown is the source of truth.**

---

## Key decisions and rationale (chronological)

1. **Deliverable format**: a full design document was produced first; then it was clarified that the goal was to **improve the user's prompt**, not to write the agents yet. → The deliverable is the meta-prompt.

2. **Language**: the prompt must be **100% English** (Claude Code convention). Explanations to the user are in their language.

3. **Stack chosen by the user prompt**: the technologies (for a new app or new feature) come from the user prompt; it is **the AI that writes them into `.sdd/target.md`**. If missing, the AI asks.

4. **Broad scope**: the workflow must work for different architectures (monolith, modular, microservices, CLI, libraries) and stacks (Java, Kotlin, .NET, Node/TS, Python, Go + React/Angular/Vue/Svelte). Agents and templates are **agnostic**; the target is a parameter.

5. **Role prompt at the top of every agent** (ROLE/MISSION/MINDSET/NON-GOALS), always carrying the two values: MD source of truth and reuse (DRY).

6. **Reuse as a primary objective + UI library**: a default UI component library (Header/Body/Footer/Panel + helpers) progressively enriched, atomic-design style; screens reference components by id. Added a dedicated **reuse-analyst**.

7. **Spec ↔ source link**: each spec declares in its **front-matter** the files it produces (`source:`); the code carries a header pointing back to the spec. The mapping has a **single authoritative home** (the front-matter); indexes and traceability are **derived**.

8. **Isolation — major evolution**:
   - Initially: the code translator had to read **only MD** (hard source-blind), with a `PreToolUse` hook + lock.
   - Then the user wanted the **ability to read and edit the source** (bugs and evolution live in the concretization layer that SCoT doesn't describe). → The principle was **reframed as authority**, not blindness.
   - Finally the hook was **dropped entirely** (it was fragile and buggy: "deny by default" would have blocked all reads). The **test-writer**'s independence remains, but **soft** (role prompt + no Bash + test-gatekeeper check).

9. **Bug-fix / Change policy**: no whole-file rewrites; **minimal diffs** by default (for bug-fix and evolution); full regeneration only as an exception. Concretization SCoT omits goes into the **Implementation Notes**.

10. **Implementation Notes outside the spec**: the (gated) behavioral spec is **never touched**; the notes live in `.sdd/impl-notes/<id>.md`, owned by the implementer.

11. **The 5 modes (defined by the user)**: Plan (plans indexes/specs) → Plan approval (writes them) → Implement (code) → Test → Automatic (everything, human out of the loop). The first 4 = manual step-by-step path.

12. **Status in the indexes**: each entry's `status` lives in the **indexes** (multi-level). Gatekeepers only judge (verdict to `state.md`); the **orchestrator/command** advances the status.

13. **`traceability.md` removed**: redundant; traceability is reconstructed on demand from indexes + front-matter. Optional `/sdd-trace` command.

14. **Behavioral drift-check**: "code ↔ spec" equivalence is proven by the **tests** (enforced coverage: every AC and every SCoT branch), not by a textual comparison (LLM regeneration isn't deterministic).

15. **Four spec levels**: feature (orchestration) / **entity-data-model** (fields/types/relations/constraints → code + migrations) / class (SCoT) / UI (schematic).

16. **Other expert choices applied**: a project/infra spec `MOD-build` (build/CI/migrations); **topological** dependency order; **canonical SCoT grammar** in `.sdd/scot.md` (+ `ui-schema.md`); auto mode by **vertical slices**; reuse-analyst made a **pure author** (only the analysis-gatekeeper blocks); ACs with ids in Given/When/Then form; `requirements:` as a list; build/test commands in `target.md`.

17. **DTO / STUB / interfaces / enums / config**: no SCoT (SCoT is for behavior only). Added a **`kind:`** field to each spec that decides the form: *behavioral* (service, controller, use-case) → SCoT; *structural* (entity, dto, enum, interface, config) → declarative (field/signature table); *gui* → schematic. A **stub** is not specced separately: it is **derived from the contract/interface**.

18. **Indexes per level (not per module)**: one index per type (modules, features, entities, classes, ui-components), global, with a `module` column. Adding a multi-module feature = add rows to the existing indexes + one spec per entry. For large projects, an option to **split the fine-grained indexes per module** (decided in the plan). Added a concrete, readable **example** (User registration: Database + API) to both the prompt and the technical document.

---

## Current state

The prompt is considered **coherent, mature, and final** (the version from which the actual implementation of the prompts/agents starts). Covered: the principle, 4 spec levels + declarative structural artifacts (DTO/interfaces/enums/stub), `kind:` to choose the form, indexes per level with a readable example, separated author/judge roles, reuse + UI library, minimal-diff fixes, tests as an oracle with coverage, reconstructable traceability, iteration budget with escalation.

**Next step (decided):** start the **actual implementation** — have Claude Code generate the `.claude/agents/*` and `.claude/commands/*` files from the prompt, then review them.

---

## Open points / possible future work

- **Prompt readability**: it has grown long; a pure trimming pass (without changing substance) is possible.
- **Test independence "by construction"**: today it's "by process" (convention + gatekeeper). If needed, a minimal *correct* guard for the test-writer only can be reintroduced.
- **Non-functional requirements** (performance, security, accessibility): today they enter only if expressible as testable acceptance criteria; there is no dedicated level.
- **Actual generation of the `.claude/agents` and `.claude/commands` files**: not produced yet — the prompt asks Claude Code to create them. Likely next step: have them generated and review them.
- **Spec versioning**: left to git; no internal changelog mechanism.

---

## Quick glossary

- **SDD**: Spec-Driven Development — you start from the specs, the code is derived.
- **SCoT**: Structured Chain-of-Thought — structured pseudo-code (sequence/branch/loop) to describe logic.
- **Gatekeeper**: a reviewer agent with veto power over a phase.
- **Impl-notes**: concretization notes (library, API, idioms) that SCoT doesn't capture.
- **Source-blind / implementation-independent**: an agent that doesn't look at the code (today only the test-writer, softly).
- **Vertical slice**: a feature/module worked end-to-end, to bound the context.
