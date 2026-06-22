# UI Schematic — canonical convention (the source form of every GUI spec)

> **Status: canonical contract. Authored once, used by every agent.**
> This file defines how `kind: gui` specs describe a UI. It is the UI's **source
> form** — *not* SCoT and *not* framework code. A schematic is **framework-agnostic**:
> the same spec can target React, Angular, Vue, Svelte, or a non-web UI. The
> concrete framework is chosen at implementation time from `.sdd/target.md`.

There are **two kinds** of `gui` spec, both using this convention:

- **Shared UI component** (`specs/ui-components/COMP-*.spec.md`) — a reusable
  atom/molecule/organism/layout. Specified **once**, referenced everywhere by id.
- **Screen / feature view** (`specs/classes/CLS-*.spec.md`, `kind: gui`) — a
  feature-specific screen. It **composes library components by id** and only
  specifies layout + screen-specific behavior. It must **never** re-describe a
  button, input, panel, etc. that already exists in `ui-components.index.md`.

**Discover before create:** before specifying any widget, read
`specs/indexes/ui-components.index.md`. If a component exists, reference it by id.
If a recurring widget is missing, the reuse-analyst promotes it into the library.

---

## 1. The five sections of a GUI spec

Every `gui` spec contains these sections, in order:

1. **Wireframe** — an ASCII sketch of the layout (the visual intent).
2. **Component tree** — the composition, referencing library components by id.
3. **State** — a table of view state (name, type, initial, description).
4. **Events** — a table mapping user/system events to handlers and effects.
5. **Acceptance criteria** — Given/When/Then, each with a stable `ACn` id.

Shared-component specs add **Props**, **Variants**, **Visual states**, and
**Accessibility** sections (see §6).

---

## 2. Wireframe notation (ASCII)

Use box-drawing to convey structure, not pixel-perfect design. Bind dynamic text
with `{stateVar}` placeholders. Mark interactive elements with `[ ]` (button),
`(•)`/`( )` (radio), `[x]`/`[ ]` (checkbox), `▼` (dropdown), `___` (text field).

```
┌───────────────────────────────── Header (COMP-header) ─────────────────────────────────┐
│  ◀ Logo                                  Nav: Home · Account            {user.name} ▼    │
├──────────────────────────────────────── Body (COMP-body) ───────────────────────────────┤
│                                                                                          │
│   ┌──────────── Panel (COMP-panel) · title="Create account" ───────────┐                │
│   │  Email     [____________________________]   {errors.email}          │                │
│   │  Password  [____________________________]   {errors.password}       │                │
│   │  Name      [____________________________]                           │                │
│   │                                                                     │                │
│   │                                   [ Cancel ]   [ Register ]         │                │
│   └─────────────────────────────────────────────────────────────────────┘                │
│                                                                                          │
├──────────────────────────────────────── Footer (COMP-footer) ───────────────────────────┤
│  © {year} Example                                              v{appVersion}             │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

The wireframe is **indicative**. The authoritative composition is the component
tree (§3); the wireframe just makes it human-readable.

---

## 3. Component tree (composition by id)

An indented tree. Each node is either a **library component referenced by id**
(with the props it is given) or a layout slot. Never inline a component that
exists in the library.

```
COMP-appShell
├─ COMP-header        props: { brand: "Example", nav: [Home, Account], user: {user} }
├─ COMP-body
│  └─ COMP-panel      props: { title: "Create account" }
│     └─ COMP-stack   props: { gap: "md" }
│        ├─ COMP-formField  props: { label: "Email",    error: {errors.email} }
│        │  └─ COMP-textInput   props: { type: "email",    value: {email},    onChange: setEmail }
│        ├─ COMP-formField  props: { label: "Password", error: {errors.password} }
│        │  └─ COMP-textInput   props: { type: "password", value: {password}, onChange: setPassword }
│        ├─ COMP-formField  props: { label: "Name" }
│        │  └─ COMP-textInput   props: { type: "text",     value: {name},     onChange: setName }
│        └─ COMP-stack   props: { direction: "row", justify: "end", gap: "sm" }
│           ├─ COMP-button  props: { variant: "secondary", label: "Cancel",   onClick: cancel }
│           └─ COMP-button  props: { variant: "primary",   label: "Register", onClick: submit, loading: {submitting} }
└─ COMP-footer       props: { version: {appVersion} }
```

`{name}` denotes a binding to a state variable (§4). `onClick: submit` names an
event handler defined in the Events table (§5).

---

## 4. State table

| Name         | Type            | Initial | Description                          |
|--------------|-----------------|---------|--------------------------------------|
| `email`      | String          | `""`    | Email field value                    |
| `password`   | String          | `""`    | Password field value                 |
| `name`       | String          | `""`    | Display name field value             |
| `errors`     | Map<String,String> | `{}` | Field-level validation messages      |
| `submitting` | Bool            | `false` | True while the registration call runs |

Local view state only. Cross-screen/shared state (stores, query caches) is named
here but **owned** by a service/shared spec and referenced by id.

---

## 5. Events table

| Event      | Trigger                  | Handler                              | Effect                                   |
|------------|--------------------------|--------------------------------------|------------------------------------------|
| `setEmail` | user types in Email      | `email <- value`                     | re-render; clear `errors.email`          |
| `submit`   | click **Register**       | see SCoT snippet below               | calls feature `FEAT-001`; nav on success |
| `cancel`   | click **Cancel**         | navigate to `/home`                  | leaves the screen                        |

For a **non-trivial** handler, attach a small SCoT snippet (grammar = `.sdd/scot.md`),
with branch ids so it is covered by tests:

```
# handler: submit
FUNCTION submit() -> Void
  submitting <- true
  [B1] IF NOT CALL validateLocally(email, password, name) THEN
    errors <- CALL collectErrors(...)     # arm B1.then
    submitting <- false
    RETURN
  END                                     # arm B1.else
  result <- AWAIT CALL FEAT-001.register({ email, password, name })
  [B2] IF result.isOk THEN
    CALL navigate("/welcome")             # arm B2.then
  ELSE
    errors <- { form: result.error.message }   # arm B2.else
  END
  submitting <- false
END
```

Trivial handlers (a single assignment or a navigation) need no SCoT — the table
row is enough.

**Handler snippet notes.** (1) *Call target:* a screen invokes its feature **by id**
— call the feature's coordinator if it owns one, otherwise the **controller** the
feature orchestrates (e.g. `CLS-regCtrl.register`). A purely-compositional feature
(`source: []`) has **no callable code of its own**, so the screen calls its
controller, not the feature id. (2) *Error style:* if a handler's own SCoT
returns/raises errors, declare `error-style:` at its top (`.sdd/scot.md` §6); a
trivial result check like the one above needs none.

---

## 6. Extra sections for shared components (`COMP-*`)

A reusable component spec adds, after the wireframe/tree:

### Props

| Prop       | Type                       | Required | Default     | Description                 |
|------------|----------------------------|----------|-------------|-----------------------------|
| `label`    | String                     | yes      | —           | Button text                 |
| `variant`  | `primary \| secondary \| ghost \| danger` | no | `primary` | Visual style    |
| `disabled` | Bool                       | no       | `false`     | Blocks interaction          |
| `loading`  | Bool                       | no       | `false`     | Shows spinner, blocks click |
| `onClick`  | Event                      | no       | —           | Emitted on activation       |

### Variants

List the named variants and when to use each (`primary` for the main action,
`danger` for destructive actions, …).

### Visual states

`default`, `hover`, `focus`, `active`, `disabled`, `loading`, `error` — describe
what changes in each (do **not** specify colors/pixels here; those live in design
tokens referenced from `.sdd/target.md`).

### Events

| Event     | Payload        | When                          |
|-----------|----------------|-------------------------------|
| `onClick` | `{}`           | user activates the control    |

### Slots / children

Name any composition slots (e.g. a Panel has `header`, `body`, `footer` slots).

### Accessibility

Roles, keyboard interaction, focus order, ARIA intent (e.g. "button is reachable
by Tab; Enter/Space activate; `aria-busy` while `loading`"). Expressed as
behavior, not framework code, and turned into acceptance criteria where testable.

---

## 7. Atomic-design layering

Every component declares its `layer:` in front-matter so the library stays
organized and discoverable:

- **atom** — Button, TextInput, Icon, Badge, Spinner, Checkbox…
- **molecule** — FormField (label+control+error), SearchBar, Pagination…
- **organism** — Header, Footer, Panel, Table, Modal, Menu…
- **layout** — AppShell, Stack, Grid, Section (the app-shell template lives here).

Higher layers compose lower layers **by id**. A molecule never re-describes its
atoms; an organism never re-describes its molecules.

---

## 8. What a GUI spec must NOT contain

- No framework code (JSX/TSX, templates, directives, `useState`, signals…).
- No CSS / colors / pixel values — only design-token **names** (e.g. `gap: "md"`),
  resolved at implementation time from tokens declared in `.sdd/target.md`.
- No re-description of an existing library component — reference it by id.
- No business logic beyond view orchestration — that lives in a `service`/`use-case`
  spec and is called by id.

---

## 9. Mandatory baseline library (every GUI project)

For **any** project with a GUI the workflow **GUARANTEES** a default UI component
library, so screens are composed — never hand-rolled. These baseline components are
**not shipped as files**: the `spec-writer` **materializes** them (from
`.sdd/templates/ui-component.template.md`) into the project's `specs/ui-components/`
and registers them in `specs/indexes/ui-components.index.md` **before** any screen
composes them. The baseline = layout primitives + the panel container:

| id | layer | Purpose | Key props / slots |
|----|-------|---------|-------------------|
| `COMP-appShell` | layout | app frame: header on top, body fills the middle, footer at the bottom | slots: header, body, footer |
| `COMP-header` | organism | application top bar | brand, nav, actions/user |
| `COMP-body` | layout | scrollable content region between header and footer | maxWidth, padding; children slot |
| `COMP-footer` | organism | application bottom bar | leading (copyright/links), trailing (version) |
| `COMP-panel` | organism | card / panel container | title, variant (`default\|outlined\|elevated`); header/body/footer slots |
| `COMP-stack` | layout | one-dimensional flex helper | direction, gap, align, justify, wrap |
| `COMP-grid` | layout | two-dimensional grid helper | columns, gap, areas |
| `COMP-section` | layout | titled content section | title, description; children slot |

**Progressive enrichment.** Beyond the baseline, recurring widgets — Button,
TextInput, TextArea, Select, Checkbox/Radio/Toggle, FormField, Modal, Table, Tabs,
Toast, Badge, Avatar, Icon, Spinner, Pagination, Breadcrumb, Menu, … — are added to
the library as patterns emerge: the `reuse-analyst` promotes one the moment a second
screen needs it, never duplicated. Each is specified once (atomic-design layer per §7)
and referenced by id. The `analysis-gatekeeper` blocks a GUI project whose baseline is
missing, or whose screens inline a component instead of composing the library by id.
