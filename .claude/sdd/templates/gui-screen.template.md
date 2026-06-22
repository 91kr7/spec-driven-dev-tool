---
id: CLS-<lowerCamel>            # required — matches filename specs/classes/CLS-<lowerCamel>.spec.md and the classes.index row
name: <ScreenName>              # required — human name, e.g. "Login Screen"
kind: gui                       # required — fixed: this is a feature-specific SCREEN spec
module: MOD-<web-or-ui-module>  # required — the web/ui module this screen belongs to (e.g. MOD-web)
status: draft                   # draft|reviewed|approved — index row is canonical; this MIRRORS it
depends_on: [COMP-<id>, ...]    # ids — every COMP-* this screen composes + any FEAT-*/CLS-*/service it calls; topological
requirements: [REQ-<nnn>]       # requirement id(s) this screen serves (back-link)
source: [src/<web>/<ScreenName>.<ext>]   # AUTHORITATIVE spec→source mapping; proposed from .sdd/target.md conventions for a new screen
owns_sections: []               # only if this screen co-owns a shared aggregator file; else []
---

<!--
  HOW TO USE THIS TEMPLATE
  ------------------------
  Copy this file to specs/classes/CLS-<lowerCamel>.spec.md and fill every <placeholder>.
  This is a feature-specific SCREEN (kind: gui) — its UI form obeys .claude/sdd/ui-schema.md EXACTLY.
  A screen COMPOSES shared library components BY ID; it never re-describes a button, input,
  panel, etc. that already exists. DISCOVER BEFORE CREATE: read
  specs/indexes/ui-components.index.md first; reference each widget by its COMP-* id. If a
  recurring widget is missing from the library, the reuse-analyst promotes it — you do not
  inline it here.

  Values that bind this file (see .claude/sdd/conventions.md): Markdown is the source of truth
  (authority); reuse over repetition (DRY).

  Required sections for a kind: gui spec (.claude/sdd/conventions.md §3 + ui-schema §1):
    # Purpose · # Public interface · # Invariants & rules ·
    then the FIVE UI-schema sections: Wireframe, Component tree, State, Events,
    Acceptance criteria. Delete these guidance comments in the real spec.

  What a GUI spec MUST NOT contain (ui-schema §8):
    - No framework code (JSX/TSX, templates, useState, signals…).
    - No CSS / colors / pixel values — only design-token NAMES (e.g. gap: "md").
    - No re-description of an existing library component — reference it by id.
    - No business logic beyond view orchestration — call a service/use-case by id.
-->

# Purpose

<!-- One paragraph: what this screen lets the user accomplish, in WHAT terms, not HOW. -->
<PlainEnglish description of the user-facing goal of this screen.>

# Public interface

<!-- The screen's contract with the rest of the app: what it receives, what it produces,
     what it can fail to do. For a screen this is route params, the feature(s) it calls,
     and navigation outcomes. -->

- **Inputs:** <route params / props passed in, e.g. `redirectTo: String` (optional).>
- **Outputs:** <navigation targets / events emitted on success, e.g. navigates to `<route>` after success.>
- **Errors:** <visible failure modes the screen renders, e.g. invalid input shown inline; service error shown as a form-level banner.>

# Invariants & rules

<!-- Rules that always hold for this screen, expressed as behavior (testable where possible). -->

- <e.g. The primary action is disabled while a submit is in flight.>
- <e.g. Field-level errors clear as soon as the user edits the offending field.>
- <e.g. Reuse over repetition (DRY): all widgets come from COMP-* — nothing is re-described here.>

# Wireframe

<!-- ASCII sketch per ui-schema §2. Box-drawing for structure (not pixels). Bind dynamic
     text with {stateVar}. Interactive markers: [ ] button, (•)/( ) radio, [x]/[ ] checkbox,
     ▼ dropdown, ___ text field. The wireframe is indicative; the component tree is authoritative. -->

```
┌──────────── <Region> (COMP-<id>) ────────────┐
│  <sketch the layout, bind dynamic text as {stateVar}>          │
│  <Label>   [______________________]   {errors.<field>}        │
│                                   [ <PrimaryAction> ]          │
└───────────────────────────────────────────────┘
```

# Component tree

<!-- Indented tree per ui-schema §3. Each node is a library component referenced BY ID with
     the props it is given, or a layout slot. {name} = binding to a State var (§ State).
     onClick: <handler> names a row in the Events table. NEVER inline a library component. -->

```
COMP-<rootLayoutId>
├─ COMP-<id>   props: { <prop>: <value-or-{binding}> }
└─ COMP-<id>   props: { ..., onClick: <handlerName> }
```

# State

<!-- Local view state only (ui-schema §4). Cross-screen/shared state is NAMED here but OWNED
     by a service/shared spec and referenced by id. -->

| Name        | Type               | Initial | Description                       |
|-------------|--------------------|---------|-----------------------------------|
| `<var>`     | `<NeutralType>`    | `<val>` | <what this view state holds>      |

# Events

<!-- ui-schema §5. Map each user/system event to a handler and its effect. Trivial handlers
     (single assignment / navigation) need only the table row. For a NON-TRIVIAL handler,
     attach a small SCoT snippet below (grammar = .claude/sdd/scot.md) with branch ids [Bn] so the
     test-writer can cover each arm. -->

| Event           | Trigger                | Handler                     | Effect                                  |
|-----------------|------------------------|-----------------------------|-----------------------------------------|
| `<setField>`    | user types in <Field>  | `<field> <- value`          | re-render; clear `errors.<field>`       |
| `<submit>`      | click **<PrimaryAction>** | see SCoT snippet below   | calls feature `FEAT-<nnn>`; nav on success |

```
# handler: <submit>          # error-style: result (declare the style this snippet uses)
ASYNC FUNCTION <submit>() -> Void
  <flag> <- true
  [B1] IF NOT CALL <validateLocally>(<fields>) THEN
    errors <- CALL <collectErrors>(<fields>)   # arm B1.then
    <flag> <- false
    RETURN
  END                                          # arm B1.else
  result <- AWAIT CALL FEAT-<nnn>.<action>({ <fields> })
  [B2] IF result.isOk THEN
    CALL navigate("<route>")                   # arm B2.then
  ELSE
    errors <- { form: result.error.message }   # arm B2.else
  END
  <flag> <- false
END
```

# Acceptance criteria

<!-- ui-schema §5 + conventions §3: Given/When/Then, each with a stable ACn id. Cover every
     non-trivial branch arm above and every visible rule. The test-writer turns these +
     the SCoT arms into tests. TWO altitudes (GUI project): a screen's user-JOURNEY ACs —
     those whose outcome crosses the running stack (navigation-on-success, a persisted/
     emitted effect, a service-error banner) — are validated end-to-end by a **Playwright
     e2e** test against the real running app; arm-level, accessibility, and pure-view ACs
     are covered by in-process **component** tests with the feature call mocked. -->

- **AC1** — Given <context>, When <action>, Then <observable outcome>.
- **AC2** — Given <context>, When <action>, Then <observable outcome>.

---

## Filled example

> A fully-filled instance of this template: the **Login** screen for `MOD-web`.
> It composes only library components by id (COMP-appShell / COMP-header /
> COMP-body / COMP-footer / COMP-panel / COMP-formField / COMP-textInput /
> COMP-button) and specifies one non-trivial handler (`submit`) with SCoT.
> The example's own front-matter:

```yaml
---
id: CLS-loginScreen
name: Login Screen
kind: gui
module: MOD-web
status: draft
depends_on:
  [
    COMP-appShell,
    COMP-header,
    COMP-body,
    COMP-footer,
    COMP-panel,
    COMP-formField,
    COMP-textInput,
    COMP-button,
    FEAT-002,
  ]
requirements: [REQ-003]
source: [src/web/screens/LoginScreen.tsx]
owns_sections: []
---
```

### Purpose

Lets a returning user authenticate with their email and password and land on
their dashboard. On bad credentials or a service error it keeps the user on the
screen and explains what went wrong, without losing the typed email.

### Public interface

- **Inputs:** `redirectTo: String` — optional route to return to after a
  successful login (defaults to `/dashboard`).
- **Outputs:** on success, navigates to `redirectTo`; emits no domain events
  itself (authentication is owned by `FEAT-002`).
- **Errors:** empty/invalid fields shown inline per field; rejected credentials
  and transport failures shown as a single form-level banner via `errors.form`.

### Invariants & rules

- The **Sign in** button is disabled while `submitting` is `true` (no double
  submit).
- A field's error in `errors` clears the moment the user edits that field.
- Markdown is the source of truth (authority); reuse over repetition (DRY): every
  widget on this screen is a `COMP-*` referenced by id — nothing is re-described.

### Wireframe

```
┌──────────────────────────── Header (COMP-header) ────────────────────────────┐
│  ◀ Acme                                              Need an account? Sign up │
├───────────────────────────────── Body (COMP-body) ────────────────────────────┤
│                                                                               │
│      ┌──────────── Panel (COMP-panel) · title="Sign in" ───────────┐          │
│      │  {errors.form}                                               │          │
│      │  Email     [____________________________]  {errors.email}    │          │
│      │  Password  [____________________________]  {errors.password} │          │
│      │                                                              │          │
│      │                                       [ Sign in ]            │          │
│      └──────────────────────────────────────────────────────────────┘          │
│                                                                               │
├──────────────────────────────── Footer (COMP-footer) ─────────────────────────┤
│  © {year} Acme                                              v{appVersion}      │
└───────────────────────────────────────────────────────────────────────────────┘
```

### Component tree

```
COMP-appShell
├─ COMP-header     props: { brand: "Acme", nav: [{ label: "Sign up", href: "/register" }] }
├─ COMP-body
│  └─ COMP-panel   props: { title: "Sign in", banner: {errors.form} }
│     ├─ COMP-formField   props: { label: "Email", error: {errors.email} }
│     │  └─ COMP-textInput    props: { type: "email", value: {email}, onChange: setEmail }
│     ├─ COMP-formField   props: { label: "Password", error: {errors.password} }
│     │  └─ COMP-textInput    props: { type: "password", value: {password}, onChange: setPassword }
│     └─ COMP-button      props: { variant: "primary", label: "Sign in", onClick: submit, disabled: {submitting}, loading: {submitting} }
└─ COMP-footer     props: { version: {appVersion} }
```

### State

| Name         | Type                  | Initial | Description                            |
|--------------|-----------------------|---------|----------------------------------------|
| `email`      | `String`              | `""`    | Email field value                      |
| `password`   | `String`              | `""`    | Password field value                   |
| `errors`     | `Map<String,String>`  | `{}`    | Field-level + `form` validation messages |
| `submitting` | `Bool`                | `false` | True while the authentication call runs |

### Events

| Event        | Trigger                | Handler                                  | Effect                                       |
|--------------|------------------------|------------------------------------------|----------------------------------------------|
| `setEmail`   | user types in Email    | `email <- value`                         | re-render; clear `errors.email` and `errors.form` |
| `setPassword`| user types in Password | `password <- value`                      | re-render; clear `errors.password` and `errors.form` |
| `submit`     | click **Sign in**      | see SCoT snippet below                   | calls `FEAT-002`; nav to `redirectTo` on success |

```
# handler: submit          # error-style: result
ASYNC FUNCTION submit() -> Void
  INPUT:  email, password (current field state); redirectTo (route param)
  OUTPUT: navigates on success; populates `errors` on failure

  submitting <- true

  [B1] IF NOT CALL validateLocally(email, password) THEN
    errors <- CALL collectErrors(email, password)   # arm B1.then — missing/invalid fields
    submitting <- false
    RETURN
  END                                               # arm B1.else — fields well-formed

  result <- AWAIT CALL FEAT-002.authenticate({ email, password })

  [B2] IF result.isOk THEN
    CALL navigate(redirectTo)                        # arm B2.then — success
  ELSE
    errors <- { form: result.error.message }         # arm B2.else — rejected / transport error
  END

  submitting <- false
END
```

Coverage set for `submit`: `B1.then`, `B1.else`, `B2.then`, `B2.else`.

### Acceptance criteria

- **AC1** — Given a registered user with correct credentials, When they fill
  Email and Password and click **Sign in**, Then `FEAT-002.authenticate` is
  called and the screen navigates to `redirectTo` (covers `submit#B1.else`,
  `submit#B2.then`).
- **AC2** — Given the Email field is empty, When the user clicks **Sign in**,
  Then no authentication call is made and `errors.email` shows a required-field
  message (covers `submit#B1.then`).
- **AC3** — Given valid-looking but wrong credentials, When the user clicks
  **Sign in**, Then `errors.form` shows the service error message and the user
  stays on the screen with the typed email preserved (covers `submit#B2.else`).
- **AC4** — Given a submit is in flight, When `submitting` is `true`, Then the
  **Sign in** button is disabled and shows its loading state (covers the
  no-double-submit invariant).
- **AC5** — Given `errors.email` is set, When the user edits the Email field,
  Then `errors.email` and `errors.form` are cleared on the next render (covers
  the clear-on-edit invariant via `setEmail`).
