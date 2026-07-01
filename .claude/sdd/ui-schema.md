# UI Schematic — canonical convention (source form of every GUI spec)

> **Canonical contract.** How `kind: gui` specs describe a UI — the UI's **source
> form**, not SCoT, not framework code. Framework-agnostic (React/Angular/Vue/
> Svelte/…); concrete framework chosen at implementation time from `target.md`.

**Two kinds of `gui` spec:**
- **Shared component** (`.sdd/specs/<MOD>/ui-components/COMP-*.spec.md`) — a reusable atom/molecule/organism/layout. Placement is by **nature, not consumer count** (conventions [§13](conventions.md#s13)): a **generic primitive** (Button, Panel, Form, Header, Footer — agnostic to the domain) lives in **`MOD-shared`** (the design-system kit); a **domain component** (one that names a domain concept, e.g. a `UserCard`) lives in its own module's `ui-components/` and composes the generic kit by id. Specify **once**; reference everywhere by id.
- **Screen** (`.sdd/specs/<MOD>/classes/CLS-*.spec.md`, `kind: gui`) — composes library components **by id**; specifies layout + screen-specific behavior only. Never re-describes an already-indexed widget.

**Discover before create:**
- Before specifying any widget, read the `ui-component` rows in `MOD-shared.index.md` + the module's own `<MOD>.index.md`.
- If it exists, reference by id.
- If a generic primitive is missing from the kit, it is materialized in `MOD-shared` at **first** use (no second-use wait); a domain widget is authored in its own module.

---

<a id="s1"></a>
## 1. The five sections of a GUI spec (in order)
1. **Wireframe** — ASCII sketch of layout.
2. **Component tree** — composition referencing library components by id.
3. **State** — table (name, type, initial, description).
4. **Events** — table: events → handlers → effects.
5. **Acceptance criteria** — Given/When/Then, each with a stable `ACn` id.

`COMP-*` specs add **Props · Variants · Visual states · Events · Slots/children · Accessibility** ([§6](#s6)).

---

<a id="s2"></a>
## 2. Wireframe notation
- Box-drawing for structure (not pixels).
- Bind dynamic text with `{stateVar}`.
- Mark interactive elements: `[ ]` button, `(•)`/`( )` radio, `[x]`/`[ ]` checkbox, `▼` dropdown, `___` text field.
- Wireframe is **indicative**; authoritative composition = component tree ([§3](#s3)).

---

<a id="s3"></a>
## 3. Component tree (composition by id)
- Indented tree.
- Each node = a **library component referenced by id** (with its props) or a layout slot.
- Never inline a library component.

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
`{name}` = binding to a state var ([§4](#s4)); `onClick: submit` = a handler in Events table ([§5](#s5)).

---

<a id="s4"></a>
## 4. State table
- Local view state only.
- Cross-screen/shared state: name it here, but **owned** by a service/shared spec and referenced by id.

| Name | Type | Initial | Description |
|---|---|---|---|
| `email` | String | `""` | Email field value |
| `errors` | Map<String,String> | `{}` | Field-level messages |
| `submitting` | Bool | `false` | True while the call runs |

---

<a id="s5"></a>
## 5. Events table + journey ACs

| Event | Trigger | Handler | Effect |
|---|---|---|---|
| `setEmail` | types Email | `email <- value` | re-render; clear `errors.email` |
| `submit` | click Register | SCoT snippet below | calls `FEAT-001`; nav on success |
| `cancel` | click Cancel | navigate `/home` | leaves screen |

**Non-trivial** handler → attach a small SCoT snippet (grammar = `scot.md`) with branch ids so it's covered:

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

A screen that **calls a feature** (`CALL FEAT-…`/`CALL CLS-…` on its handler's success path) MUST declare:
- **≥1 `(journey)` AC** (primary success), plus
- a `(journey)` AC per rendered end-to-end failure.

Enforcement:
- test-writer writes one e2e per `(journey)` AC.
- test-gatekeeper requires it.
- analysis-gatekeeper blocks a feature-calling screen with no `(journey)` AC.

**Call target:** a screen invokes its feature **by id**:
- the feature's coordinator if it owns one, else
- the controller it orchestrates (e.g. `CLS-regCtrl.register`).

A purely-compositional feature (`source: []`) has no callable code → screen calls its controller.

---

<a id="s6"></a>
## 6. Extra sections for `COMP-*`

- **Props** — table (prop, type, required, default, description).
- **Variants** — named variants + when to use each.
- **Visual states** — `default/hover/focus/active/disabled/loading/error` (behavior, no colors/pixels — those are design tokens from `target.md`).
- **Events** — table (event, payload, when).
- **Slots / children** — named composition slots.
- **Accessibility** — roles, keyboard, focus order, ARIA intent (e.g. "Tab-reachable; Enter/Space activate; `aria-busy` while `loading`") — as behavior, → ACs where testable.

---

<a id="s7"></a>
## 7. Atomic-design layering
Every component declares `layer:`; higher layers compose lower layers **by id** (a molecule never re-describes its atoms).
- **atom** — Button, TextInput, Icon, Badge, Spinner, Checkbox…
- **molecule** — FormField, SearchBar, Pagination…
- **organism** — Header, Footer, Panel, Table, Modal, Menu…
- **layout** — AppShell, Stack, Grid, Section.

---

<a id="s8"></a>
## 8. A GUI spec must NOT contain
- Framework code (JSX, templates, `useState`, signals…).
- CSS/colors/pixels — only design-token **names** (`gap: "md"`), resolved from `target.md`.
- Re-description of an existing library component — reference by id.
- Business logic beyond view orchestration — lives in a `service`/`use-case` spec, called by id.

---

<a id="s9"></a>
## 9. Reusable component catalog (compose, don't hand-roll)
- A GUI project composes screens from reusable components instead of hand-rolling duplicated markup — but creates **only the components a screen actually composes**.
- The table below = a **catalog of common candidates** (canonical ids/layers to reuse when needed), **NOT a mandatory set**. Reach for a frame component (`appShell`/`header`/`footer`/…) only when the app's views share that structure — a single-screen app may need none.
- `spec-writer` materializes a catalog component (from `templates/ui-component.template.md`) into **`MOD-shared/ui-components/`** (these frame/kit components are domain-agnostic primitives — [§1](#s1), conventions [§13](conventions.md#s13)) + index the **first time a screen composes it**, never up front.
- `analysis-gatekeeper` blocks:
  - a screen that **inlines/hand-rolls** a component instead of composing one by id, and
  - any **unused (orphan)** component (each must carry its consumers' `requirements:` — conventions [§13](conventions.md#s13)).

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

**Build the kit by nature, not by count:** beyond this frame catalog, the generic widgets (Button, TextInput, Select, Modal, Table, Toast, …) are materialized in `MOD-shared` the **first** time any screen composes one — because they are domain-agnostic primitives, not because a second screen repeated them. Specified once (layer per [§7](#s7)), referenced by id. A component that **names a domain concept** is not a kit primitive: author it in its own module, composing the generic kit by id.
