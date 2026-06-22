<!--
  FROZEN, ILLUSTRATIVE EXAMPLE — NOT part of the tool's real specs/.
  Part of the "User registration (Database + API)" worked example.
  Delete examples/user-registration/ before real use.
  Authority: .sdd/conventions.md (ids §2, front-matter §3, kind→form §3),
             .sdd/ui-schema.md (the FIVE GUI sections; compose library COMP-* BY ID),
             .sdd/scot.md (only the non-trivial submit handler carries a SCoT snippet).
  Discover before create: layout primitives come from the shared SDD UI library
  (referenced by id, NOT redefined). No framework code, no CSS/colors — token names only.
  Markdown is the source of truth (authority); reuse over repetition (DRY).
-->
---
id: CLS-registerScreen
name: RegisterScreen
kind: gui
module: MOD-web
status: reviewed
depends_on: [CLS-regCtrl, COMP-formField, COMP-textInput, COMP-button]
requirements: [REQ-001]
source: [src/web/screens/RegisterScreen.tsx]
owns_sections: []
---

# Purpose

`RegisterScreen` is the account-creation screen. It collects an email, a password,
and a display name, submits them to the registration feature, and routes the user
forward on success or surfaces the server's error message on failure. It owns only
layout and screen-specific behavior — it **composes library components by id** and
delegates all business logic to the registration controller `CLS-regCtrl` (the
`FEAT-001` use-case entry point).

# Public interface

- **Inputs:** route/props — an optional `redirectTo` route (defaults to
  `/welcome`); the form fields (`email`, `password`, `name`) are local state.
- **Outputs:** on success navigates to `/welcome`; emits no domain events itself
  (registration is owned by `CLS-regCtrl`).
- **Errors:** field-level validation shown inline per field (`errors.email`,
  `errors.password`, `errors.name`); a controller error (`EmailAlreadyTaken`,
  `WeakPassword`) shown as a form-level banner via `errors.form`.

# Invariants & rules

- The **Register** button is in its `loading` state while `submitting` is `true`
  (no double submit).
- A field's error clears the moment the user edits that field.
- Every widget on this screen is a `COMP-*` referenced by id — nothing is
  re-described here (reuse over repetition / DRY).

# Wireframe

```
┌──────────────────────── Header (COMP-header) ────────────────────────┐
│  ◀ Example                                            Sign in         │
├──────────────────────────── Body (COMP-body) ────────────────────────┤
│                                                                       │
│   ┌──────────── Panel (COMP-panel) · title="Create account" ──────┐   │
│   │  Email     [____________________________]  {errors.email}     │   │
│   │  Password  [____________________________]  {errors.password}  │   │
│   │  Name      [____________________________]  {errors.name}      │   │
│   │                                            {errors.form}      │   │
│   │                                                               │   │
│   │                               [ Cancel ]   [ Register ]       │   │
│   └───────────────────────────────────────────────────────────────┘   │
│                                                                       │
├─────────────────────────── Footer (COMP-footer) ─────────────────────┤
│  © {year} Example                                       v{appVersion} │
└───────────────────────────────────────────────────────────────────────┘
```

The wireframe is indicative; the authoritative composition is the component tree
below. When `submitting` is true the **Register** button shows its loading state.

# Component tree

<!-- Composition by id (ui-schema §3). The layout primitives COMP-appShell /
     COMP-header / COMP-body / COMP-footer / COMP-panel / COMP-stack are provided by
     the SHARED SDD UI library — referenced here, NEVER redefined in this slice. -->

```
COMP-appShell
├─ COMP-header        props: { brand: "Example", nav: ["Sign in"] }
├─ COMP-body
│  └─ COMP-panel      props: { title: "Create account" }
│     └─ COMP-stack   props: { gap: "md" }
│        ├─ COMP-formField  props: { label: "Email",    required: true, error: {errors.email} }
│        │  └─ COMP-textInput   props: { type: "email",    value: {email},    onChange: setEmail }
│        ├─ COMP-formField  props: { label: "Password", required: true, error: {errors.password} }
│        │  └─ COMP-textInput   props: { type: "password", value: {password}, onChange: setPassword }
│        ├─ COMP-formField  props: { label: "Name",     required: true, error: {errors.name} }
│        │  └─ COMP-textInput   props: { type: "text",     value: {name},     onChange: setName }
│        └─ COMP-stack   props: { direction: "row", justify: "end", gap: "sm" }
│           ├─ COMP-button  props: { variant: "secondary", label: "Cancel",   onClick: cancel }
│           └─ COMP-button  props: { variant: "primary",   label: "Register", onClick: submit, loading: {submitting} }
└─ COMP-footer       props: { version: {appVersion} }
```

# State

| Name         | Type               | Initial | Description                                  |
|--------------|--------------------|---------|----------------------------------------------|
| `email`      | String             | `""`    | Email field value                            |
| `password`   | String             | `""`    | Password field value                         |
| `name`       | String             | `""`    | Display-name field value                     |
| `errors`     | Map<String,String> | `{}`    | Field-level + form-level validation messages |
| `submitting` | Bool               | `false` | True while the registration call is in flight|

# Events

| Event         | Trigger                | Handler                          | Effect                                            |
|---------------|------------------------|----------------------------------|---------------------------------------------------|
| `setEmail`    | user types in Email    | `email <- value`                 | re-render; clear `errors.email`                   |
| `setPassword` | user types in Password | `password <- value`              | re-render; clear `errors.password`                |
| `setName`     | user types in Name     | `name <- value`                  | re-render; clear `errors.name`                    |
| `submit`      | click **Register**     | see SCoT snippet below           | calls `CLS-regCtrl.register`; navigates on Ok              |
| `cancel`      | click **Cancel**       | navigate to `/home`              | leaves the screen without submitting              |

`setEmail` / `setPassword` / `setName` and `cancel` are trivial (a single
assignment or a navigation) — the table row is enough. `submit` is non-trivial, so
it carries a SCoT snippet (grammar = scot.md) with branch ids for coverage:

```
# handler: submit
FUNCTION submit() -> Void
  submitting <- true
  result <- AWAIT CALL CLS-regCtrl.register({ email, password, name })
  [B1] IF result.isOk THEN
    CALL navigate("/welcome")                  # arm B1.then — success
  ELSE
    errors <- { form: result.error.message }   # arm B1.else — EmailAlreadyTaken | WeakPassword
  END
  submitting <- false
END
```

Coverage set for `submit`: `B1.then`, `B1.else`.

# Acceptance criteria

- **AC1** — Given valid email, an 8+ character password, and a name, When the user
  clicks **Register** and `CLS-regCtrl.register` returns `Ok`, Then the screen
  navigates to `/welcome`. *(covers B1.then)*
- **AC2** — Given a submission that `CLS-regCtrl.register` rejects (e.g.
  `EmailAlreadyTaken` or `WeakPassword`), When the call returns `Err`, Then the
  screen stays put and `errors.form` shows the server's message. *(covers B1.else)*
- **AC3** — Given the user clicks **Register**, When the registration call is in
  flight (`submitting == true`), Then the **Register** `COMP-button` is in its
  `loading` state (blocking re-submission) until the call resolves.
