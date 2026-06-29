# UI Schematic — canonical convention (source form of every GUI spec)

<instruction>Canonical contract. How `kind: gui` specs describe a UI — the UI's source form, not SCoT and not framework code. Framework-agnostic; concrete framework chosen from `target.md`.</instruction>

**Two kinds of `gui` spec:**
<rules>
- Shared component (`.sdd/specs/<MOD>/ui-components/COMP-*.spec.md`) — reusable atom/molecule/organism/layout. Specified once, referenced everywhere by id.
- Screen (`.sdd/specs/<MOD>/classes/CLS-*.spec.md`, `kind: gui`) — composes library components by id; specifies layout + screen-specific behavior only. Never re-describes a widget already indexed.
Discover before create: read `ui-component` rows in indexes before specifying any widget; reference by id if it exists.
</rules>

## 1. The five sections of a GUI spec (in order)
<instruction>
1. Wireframe — ASCII sketch of layout.
2. Component tree — composition referencing library components by id.
3. State — table (name, type, initial, description).
4. Events — table mapping events → handlers → effects.
5. Acceptance criteria — Given/When/Then, each with stable `ACn` id.
`COMP-*` specs add Props · Variants · Visual states · Events · Slots/children · Accessibility (§6).
</instruction>

## 2. Wireframe notation
<instruction>Box-drawing for structure (not pixels). Bind dynamic text with `{stateVar}`. Mark interactive elements: `[ ]` button, `(•)`/`( )` radio, `[x]`/`[ ]` checkbox, `▼` dropdown, `___` text field. Indicative only; authoritative composition is the component tree.</instruction>

## 3. Component tree (composition by id)
<instruction>Indented tree; each node is a library component referenced by id (with its props) or a layout slot. Never inline a library component.</instruction>
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
<instruction>`{name}` = binding to a state var; `onClick: submit` = handler in Events table.</instruction>

## 4. State table
<instruction>Local view state only. Cross-screen/shared state is named here but owned by a service/shared spec and referenced by id.</instruction>
| Name | Type | Initial | Description |
|---|---|---|---|
| `email` | String | `""` | Email field value |
| `errors` | Map<String,String> | `{}` | Field-level messages |
| `submitting` | Bool | `false` | True while the call runs |

## 5. Events table + journey ACs
| Event | Trigger | Handler | Effect |
|---|---|---|---|
| `setEmail` | types Email | `email <- value` | re-render; clear `errors.email` |
| `submit` | click Register | SCoT snippet below | calls `FEAT-001`; nav on success |
| `cancel` | click Cancel | navigate `/home` | leaves screen |

<instruction>Non-trivial handler → attach a small SCoT snippet (`scot.md`) with branch ids:</instruction>
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
<instruction>Trivial handlers need no SCoT.</instruction>

<rules>
Journey ACs: A screen's ACs come in two altitudes, marked so coverage is mechanical:
- `(journey)` AC — outcome crosses running stack (nav-on-success, persisted effect). Validated end-to-end by Playwright test.
- view AC (untagged) — inline validation, disabled-while-busy, a11y: covered by component test with feature mocked.
A screen that calls a feature MUST declare ≥1 `(journey)` AC (primary success) + a `(journey)` AC per rendered end-to-end failure. Call target: invoke feature by id.
</rules>

## 6. Extra sections for `COMP-*`
<instruction>
- Props — table (prop, type, required, default, description).
- Variants — named variants + when to use each.
- Visual states — `default/hover/focus/active/disabled/loading/error` (behavior, no colors/pixels).
- Events — table (event, payload, when).
- Slots / children — named composition slots.
- Accessibility — roles, keyboard, focus order, ARIA intent — as behavior, turned into ACs.
</instruction>

## 7. Atomic-design layering
<instruction>Every component declares `layer:`; higher layers compose lower layers by id.</instruction>
- atom — Button, TextInput, Icon, Badge, Spinner, Checkbox…
- molecule — FormField, SearchBar, Pagination…
- organism — Header, Footer, Panel, Table, Modal, Menu…
- layout — AppShell, Stack, Grid, Section.

## 8. A GUI spec must NOT contain
<rules>
- Framework code (JSX, templates, `useState`, signals…).
- CSS/colors/pixels — only design-token names (`gap: "md"`), resolved from `target.md`.
- Re-description of an existing library component — reference by id.
- Business logic beyond view orchestration — that lives in a `service`/`use-case` spec.
</rules>

## 9. Reusable component catalog (compose, don't hand-roll)
<instruction>Compose screens from reusable components. Create only components a screen actually composes. The table is a catalog of common candidates, NOT mandatory. The spec-writer materializes a catalog component the FIRST TIME a screen composes it.</instruction>

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

<instruction>Progressive enrichment: recurring widgets are promoted by reuse-analyst the moment a SECOND screen needs one — specified once, referenced by id.</instruction>
