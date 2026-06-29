<!--
<instructions>
TEMPLATE: feature SCREEN spec (CLS-*, kind: gui). Copy to `.sdd/specs/<MOD>/classes/CLS-<lowerCamel>.spec.md`.
UI form = ui-schema.md EXACTLY. COMPOSES library components BY ID; never re-describes one.
Discover before create: read MOD-shared.index.md + `<MOD>.index.md` first. DRY.
No framework code, no CSS/colors — only design-token names; no business logic beyond view orchestration.
Delete the `<example>` block before saving.
</instructions>
-->
---
id: CLS-<lowerCamel>          # required
name: <ScreenName>            # required
kind: gui                     # required — feature-specific screen
module: MOD-<web-or-ui>       # required
depends_on: [COMP-<id>, FEAT-<nnn>] # every COMP-* composed + FEAT-*/CLS-* called; topological
requirements: [REQ-<nnn>]
source: [src/<web>/<ScreenName>.<ext>]
owns_sections: []
---

# Purpose
<instruction>What the user accomplishes here, in WHAT terms not HOW.</instruction>

# Public interface
<instruction>Inputs (route params/props), Outputs (nav targets/events), Errors (visible failure modes).</instruction>

# Invariants & rules
<instruction>Behavioral, testable where possible.</instruction>
- <rule>

# Wireframe
<instruction>ASCII per ui-schema §2; bind dynamic text with {stateVar}. Indicative; component tree is authoritative.</instruction>
```
┌──────── <Region> (COMP-<id>) ────────┐
│ <Label> [______]  {errors.<field>}   │
│                    [ <PrimaryAction> ]│
└──────────────────────────────────────┘
```

# Component tree
<instruction>Indented tree per ui-schema §3: library components BY ID with props; {name}=state binding; onClick:<handler>=Events row. Never inline a library component.</instruction>
```
COMP-<rootLayoutId>
├─ COMP-<id>  props: { <prop>: {binding} }
└─ COMP-<id>  props: { …, onClick: <handler> }
```

# State
<instruction>Local view state only (ui-schema §4); shared state named here but owned by service/shared spec.</instruction>
| Name | Type | Initial | Description |
|------|------|---------|-------------|
| `<var>` | `<NeutralType>` | `<val>` | <what it holds> |

# Events
<instruction>Map each event → handler → effect. Trivial = row only; non-trivial = SCoT snippet (scot.md) with [Bn] arms.</instruction>
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
<instruction>Given/When/Then, stable ACn; cover every non-trivial arm + visible rule. TAG altitude (ui-schema §5): `(journey)` = Playwright e2e; untagged = component test. Feature-calling screen MUST declare ≥1 `(journey)` AC.</instruction>
- **AC1** (journey) — Given <context>, When <action>, Then <observable cross-stack outcome>.
- **AC2** — Given <context>, When <action>, Then <observable view outcome>.

---
<example name="CLS-loginScreen (MOD-web)">
```yaml
id: CLS-loginScreen
name: Login Screen
kind: gui
module: MOD-web
depends_on: [COMP-appShell, COMP-header, COMP-body, COMP-footer, COMP-panel, COMP-formField, COMP-textInput, COMP-button, FEAT-002]
requirements: [REQ-003]
source: [src/web/screens/LoginScreen.tsx]
```

**Purpose**
A returning user authenticates with email + password and lands on their dashboard; on failure they stay, with typed email preserved.

**Invariants**
- Sign in disabled while `submitting`
- Field error clears on edit
- Every widget is a `COMP-*` by id

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

**State**
`email: String=""` · `password: String=""` · `errors: Map<String,String>={}` · `submitting: Bool=false`

**Events**
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

- **AC1** (journey) — Given correct credentials, When user submits, Then `FEAT-002.authenticate` called and nav to `redirectTo`. *(B1.else, B2.then → e2e)*
- **AC2** — Given empty Email, When user clicks Sign in, Then no call made and `errors.email` shows required message. *(B1.then)*
- **AC3** (journey) — Given wrong credentials, When user submits, Then `errors.form` shows service error and email preserved. *(B2.else → e2e)*
- **AC4** — Given submit in flight, When `submitting` is true, Then Sign in is disabled + shows loading.
</example>
