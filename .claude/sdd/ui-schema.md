# UI Schematic — canonical convention (source form of every GUI spec)

> **Canonical contract.** How `kind: gui` specs describe a UI — the UI's **source
> form**, not SCoT and not framework code. Framework-agnostic (React/Angular/Vue/
> Svelte/…); the concrete framework is chosen at implementation time from `target.md`.

**Two kinds of `gui` spec:**
- **Shared component** (`specs/ui-components/COMP-*.spec.md`) — a reusable atom/molecule/organism/layout. Specified **once**, referenced everywhere by id.
- **Screen** (`specs/classes/CLS-*.spec.md`, `kind: gui`) — composes library components **by id**; specifies layout + screen-specific behavior only. Never re-describes a widget already in `ui-components.index.md`.

**Discover before create:** read `specs/indexes/ui-components.index.md` before specifying any widget; if it exists, reference by id; if a recurring widget is missing, the reuse-analyst promotes it.

---

## 1. The five sections of a GUI spec (in order)
1. **Wireframe** — ASCII sketch of the layout.
2. **Component tree** — composition referencing library components by id.
3. **State** — table (name, type, initial, description).
4. **Events** — table mapping events → handlers → effects.
5. **Acceptance criteria** — Given/When/Then, each with a stable `ACn` id.

`COMP-*` specs add **Props · Variants · Visual states · Events · Slots/children · Accessibility** (§6).

---

## 2. Wireframe notation
Box-drawing for structure (not pixels). Bind dynamic text with `{stateVar}`. Mark interactive elements: `[ ]` button, `(•)`/`( )` radio, `[x]`/`[ ]` checkbox, `▼` dropdown, `___` text field. The wireframe is **indicative**; the authoritative composition is the component tree (§3).

---

## 3. Component tree (composition by id)
Indented tree; each node is a **library component referenced by id** (with its props) or a layout slot. Never inline a library component.

```
COMP-appShell
├─ COMP-header   props: { brand: "Example", nav: [Home, Account], user: {user} }
├─ COMP-body
│  └─ COMP-panel props: { title: "Create account" }
│     └─ COMP-stack props: { gap: "md" }
│        ├─ COMP-formField props: { label: "Email", error: {errors.email} }
│        │  └─ COMP-textInput props: { type: "email", value: {email}, onChange: setEmail }
│        └─ COMP-button props: { variant: "primary", label: "Register", onClick: submit, loading: {submitting} }
└─ COMP-footer   props: { version: {appVersion} }
```
`{name}` = binding to a state var (§4); `onClick: submit` = a handler in the Events table (§5).

---

## 4. State table
Local view state only. Cross-screen/shared state is named here but **owned** by a service/shared spec and referenced by id.

| Name | Type | Initial | Description |
|---|---|---|---|
| `email` | String | `""` | Email field value |
| `errors` | Map<String,String> | `{}` | Field-level messages |
| `submitting` | Bool | `false` | True while the call runs |

---

## 5. Events table + journey ACs

| Event | Trigger | Handler | Effect |
|---|---|---|---|
| `setEmail` | types Email | `email <- value` | re-render; clear `errors.email` |
| `submit` | click Register | SCoT snippet below | calls `FEAT-001`; nav on success |
| `cancel` | click Cancel | navigate `/home` | leaves screen |

**Non-trivial** handler → attach a small SCoT snippet (grammar = `scot.md`) with branch ids so it is covered:

```
# handler: submit   (error_style: result if it returns/raises)
FUNCTION submit() -> Void
  submitting <- true
  [B1] IF NOT CALL validateLocally(email, password, name) THEN
    errors <- CALL collectErrors(...)     # B1.then
    submitting <- false
    RETURN
  END                                     # B1.else
  result <- AWAIT CALL FEAT-001.register({ email, password, name })
  [B2] IF result.isOk THEN
    CALL navigate("/welcome")             # B2.then
  ELSE
    errors <- { form: result.error.message }  # B2.else
  END
  submitting <- false
END
```
Trivial handlers (single assignment / navigation) need no SCoT.

**Journey acceptance criteria (the e2e contract).** A screen's ACs come in two altitudes, **marked** so coverage is mechanical:
- **journey AC** — tagged `(journey)` after the id (`**AC1** (journey) — …`): outcome **crosses the running stack** (nav-on-success, persisted/emitted effect, rendered service-error banner). Validated **end-to-end by a Playwright test**.
- **view AC** (untagged) — inline validation, disabled-while-busy, clear-on-edit, accessibility: covered by an in-process **component** test with the feature mocked.

A screen that **calls a feature** (a `CALL FEAT-…`/`CALL CLS-…` on its handler's success path) MUST declare **≥1 `(journey)` AC** (primary success) + a `(journey)` AC per rendered end-to-end failure. test-writer writes one e2e per `(journey)` AC; test-gatekeeper requires it; analysis-gatekeeper blocks a feature-calling screen with no `(journey)` AC.

**Call target:** a screen invokes its feature **by id** — the feature's coordinator if it owns one, else the controller it orchestrates (e.g. `CLS-regCtrl.register`). A purely-compositional feature (`source: []`) has no callable code, so the screen calls its controller.

---

## 6. Extra sections for `COMP-*`

- **Props** — table (prop, type, required, default, description).
- **Variants** — named variants + when to use each.
- **Visual states** — `default/hover/focus/active/disabled/loading/error` (behavior, no colors/pixels — those are design tokens from `target.md`).
- **Events** — table (event, payload, when).
- **Slots / children** — named composition slots.
- **Accessibility** — roles, keyboard, focus order, ARIA intent (e.g. "Tab-reachable; Enter/Space activate; `aria-busy` while `loading`") — as behavior, turned into ACs where testable.

---

## 7. Atomic-design layering
Every component declares `layer:`; higher layers compose lower layers **by id** (a molecule never re-describes its atoms).
- **atom** — Button, TextInput, Icon, Badge, Spinner, Checkbox…
- **molecule** — FormField, SearchBar, Pagination…
- **organism** — Header, Footer, Panel, Table, Modal, Menu…
- **layout** — AppShell, Stack, Grid, Section.

---

## 8. A GUI spec must NOT contain
- Framework code (JSX, templates, `useState`, signals…).
- CSS/colors/pixels — only design-token **names** (`gap: "md"`), resolved from `target.md`.
- Re-description of an existing library component — reference by id.
- Business logic beyond view orchestration — that lives in a `service`/`use-case` spec, called by id.

---

## 9. Reusable component catalog (compose, don't hand-roll)
A GUI project composes its screens from reusable components instead of hand-rolling duplicated markup — but it creates **only the components a screen actually composes**. The table below is a **catalog of common candidates** (canonical ids/layers to reuse when you need them), **NOT a mandatory set**: reach for a frame component (`appShell`/`header`/`footer`/…) only when the app's views share that structure — a single-screen app may need none of it. The `spec-writer` materializes a catalog component (from `templates/ui-component.template.md`) into `specs/ui-components/` + index the **first time a screen composes it**, never up front. The `analysis-gatekeeper` blocks a screen that **inlines/hand-rolls** a component instead of composing one by id, and any **unused (orphan)** component (each must carry its consumers' `requirements:` — §traceability).

| id | layer | Purpose |
|---|---|---|
| `COMP-appShell` | layout | app frame (header/body/footer slots) |
| `COMP-header` | organism | top bar (brand, nav, actions/user) |
| `COMP-body` | layout | scrollable content region |
| `COMP-footer` | organism | bottom bar (copyright, version) |
| `COMP-panel` | organism | card/panel container (title, variant; header/body/footer slots) |
| `COMP-stack` | layout | 1-D flex helper (direction, gap, align, justify, wrap) |
| `COMP-grid` | layout | 2-D grid helper (columns, gap, areas) |
| `COMP-section` | layout | titled content section |

**Progressive enrichment:** beyond this catalog, recurring widgets (Button, TextInput, Select, Modal, Table, Toast, …) are promoted by the `reuse-analyst` the moment a **second** screen needs one — specified once (layer per §7), referenced by id.
