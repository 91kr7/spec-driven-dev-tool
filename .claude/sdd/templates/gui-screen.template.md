<!--
  TEMPLATE — feature SCREEN spec (CLS-*, kind: gui). Copy to .sdd/specs/<MOD>/classes/CLS-<lowerCamel>.spec.md.
  UI form = ui-schema.md EXACTLY. A screen COMPOSES library components BY ID; never re-describes one.
  Discover before create: read MOD-shared.index.md + the module's <MOD>.index.md first. Markdown is the source of truth; reuse over repetition (DRY).
  Sections: # Purpose · # Public interface · # Invariants & rules · then the FIVE ui-schema sections
  (Wireframe · Component tree · State · Events · Acceptance criteria). No framework code, no CSS/colors —
  only design-token names; no business logic beyond view orchestration (call a feature/service by id).
  Delete the "## Filled example".
-->
---
id: CLS-<lowerCamel>          # required — matches filename + a <MOD>.index.md row
name: <ScreenName>           # required
kind: gui                    # required — feature-specific screen
module: MOD-<web-or-ui>      # required
depends_on: [COMP-<id>, FEAT-<nnn>]   # every COMP-* composed + any FEAT-*/CLS-* called; topological
requirements: [REQ-<nnn>]    # back-link
source: [src/<web>/<ScreenName>.<ext>]
owns_sections: []
---

# Purpose
<One paragraph: what the user accomplishes here, in WHAT terms not HOW.>

# Public interface
- **Inputs:** <route params / props, e.g. `redirectTo: String` (optional)>.
- **Outputs:** <navigation targets / events on success>.
- **Errors:** <visible failure modes: inline field errors; service error as a form-level banner>.

# Invariants & rules
<Behavioral, testable where possible.>
- <e.g. primary action disabled while a submit is in flight>
- <e.g. a field error clears as soon as the user edits that field>

# Wireframe
<ASCII per ui-schema §2; bind dynamic text with {stateVar}. Indicative; the component tree is authoritative.>
```
┌──────── <Region> (COMP-<id>) ────────┐
│ <Label> [______]  {errors.<field>}   │
│                    [ <PrimaryAction> ]│
└──────────────────────────────────────┘
```

# Component tree
<Indented tree per ui-schema §3: library components BY ID with props; {name}=state binding; onClick:<handler>=an Events row. Never inline a library component.>
```
COMP-<rootLayoutId>
├─ COMP-<id>  props: { <prop>: {binding} }
└─ COMP-<id>  props: { …, onClick: <handler> }
```

# State
<Local view state only (ui-schema §4); shared state is named here but owned by a service/shared spec by id.>
| Name | Type | Initial | Description |
|------|------|---------|-------------|
| `<var>` | `<NeutralType>` | `<val>` | <what it holds> |

# Events
<Map each event → handler → effect. Trivial handlers need only the row; a non-trivial one gets a SCoT snippet (scot.md) with [Bn] arms.>
| Event | Trigger | Handler | Effect |
|-------|---------|---------|--------|
| `<setField>` | types in <Field> | `<field> <- value` | re-render; clear `errors.<field>` |
| `<submit>` | click **<PrimaryAction>** | SCoT snippet below | calls `FEAT-<nnn>`; nav on success |

```
# handler: <submit>          # error_style: result
ASYNC FUNCTION <submit>() -> Void
  <flag> <- true
  [B1] IF NOT CALL <validateLocally>(<fields>) THEN
    errors <- CALL <collectErrors>(<fields>)   # B1.then
    <flag> <- false
    RETURN
  END                                          # B1.else
  result <- AWAIT CALL FEAT-<nnn>.<action>({ <fields> })
  [B2] IF result.isOk THEN CALL navigate("<route>")   # B2.then
  ELSE errors <- { form: result.error.message } END   # B2.else
  <flag> <- false
END
```

# Acceptance criteria
<Given/When/Then, stable `ACn`; cover every non-trivial arm + visible rule. TAG the altitude (ui-schema §5): a **`(journey)`** AC crosses the running stack (nav-on-success / persisted effect / service-error banner) → Playwright e2e; an untagged **view** AC → component test (feature mocked). A feature-calling screen MUST declare ≥1 `(journey)` AC.>
- **AC1** (journey) — Given <context>, When <action>, Then <observable cross-stack outcome>.
- **AC2** — Given <context>, When <action>, Then <observable view outcome>.

---

## Filled example — `CLS-loginScreen` (MOD-web)

```yaml
id: CLS-loginScreen · name: Login Screen · kind: gui · module: MOD-web
depends_on: [COMP-appShell, COMP-header, COMP-body, COMP-footer, COMP-panel, COMP-formField, COMP-textInput, COMP-button, FEAT-002]
requirements: [REQ-003] · source: [src/web/screens/LoginScreen.tsx]
```

**Purpose** — A returning user authenticates with email + password and lands on their dashboard; on failure they stay, with the typed email preserved.

**Invariants** — Sign in disabled while `submitting`; a field's error clears on edit; every widget is a `COMP-*` by id.

**Component tree**
```
COMP-appShell
├─ COMP-header  props: { brand: "Acme", nav: [{ label: "Sign up", href: "/register" }] }
├─ COMP-body
│  └─ COMP-panel  props: { title: "Sign in", banner: {errors.form} }
│     ├─ COMP-formField props: { label: "Email", error: {errors.email} }
│     │  └─ COMP-textInput props: { type: "email", value: {email}, onChange: setEmail }
│     ├─ COMP-formField props: { label: "Password", error: {errors.password} }
│     │  └─ COMP-textInput props: { type: "password", value: {password}, onChange: setPassword }
│     └─ COMP-button props: { variant: "primary", label: "Sign in", onClick: submit, disabled: {submitting}, loading: {submitting} }
└─ COMP-footer  props: { version: {appVersion} }
```

**State** — `email: String=""` · `password: String=""` · `errors: Map<String,String>={}` · `submitting: Bool=false`.

```
# handler: submit          # error_style: result
ASYNC FUNCTION submit() -> Void
  submitting <- true
  [B1] IF NOT CALL validateLocally(email, password) THEN
    errors <- CALL collectErrors(email, password)   # B1.then
    submitting <- false
    RETURN
  END                                               # B1.else
  result <- AWAIT CALL FEAT-002.authenticate({ email, password })
  [B2] IF result.isOk THEN CALL navigate(redirectTo)   # B2.then
  ELSE errors <- { form: result.error.message } END    # B2.else
  submitting <- false
END
```
Coverage set: `B1.then/else`, `B2.then/else`.

- **AC1** (journey) — Given correct credentials, When the user submits, Then `FEAT-002.authenticate` is called and the screen navigates to `redirectTo`. *(B1.else, B2.then → e2e)*
- **AC2** — Given an empty Email, When the user clicks Sign in, Then no call is made and `errors.email` shows a required message. *(B1.then)*
- **AC3** (journey) — Given wrong credentials, When the user submits, Then `errors.form` shows the service error and the email is preserved. *(B2.else → e2e)*
- **AC4** — Given a submit in flight, When `submitting` is true, Then Sign in is disabled + shows loading.
