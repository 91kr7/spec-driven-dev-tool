---
id: MOD-web
name: Web UI
kind: module
module: MOD-web
status: reviewed
depends_on: [MOD-api]
requirements: [REQ-001]
source: []
---

> **⚠️ FROZEN EXAMPLE** — part of `examples/user-registration/`. Delete before
> real use. Conventions: `.sdd/conventions.md`; UI form: `.sdd/ui-schema.md`.
> Values: **Markdown is the source of truth (authority); reuse over repetition (DRY).**

# Purpose

`MOD-web` is the **Web UI** module. It owns the registration screen the end user
interacts with and composes it from the shared SDD UI component library
(discover-before-create — it never re-describes a library widget).

# Public interface

- **Owns:** `CLS-registerScreen` (`RegisterScreen`, `kind: gui`).
- **Inputs:** user keystrokes/clicks on the form.
- **Outputs:** a `RegisterCmd` submitted to `FEAT-001` / `CLS-regCtrl.register`;
  navigation to `/welcome` on success or an error message on failure.
- **Depends on:** `MOD-api` (for the registration feature/controller) and the
  shared UI library.

# Invariants & rules

- The screen **composes UI components by id** (`COMP-formField`, `COMP-textInput`,
  `COMP-button`, and the shared layout primitives `COMP-appShell` / `COMP-header`
  / `COMP-body` / `COMP-footer` / `COMP-panel` / `COMP-stack`); it never inlines
  or re-specifies them (`.sdd/ui-schema.md` §3).
- No business logic lives in the view — it calls the feature/controller by id and
  reacts to the `Result` (`.sdd/ui-schema.md` §8).
- Layout primitives come from the **shared SDD UI library**, not from this
  example's promoted components.

# Contents

| id | name | kind | spec |
|----|------|------|------|
| CLS-registerScreen | RegisterScreen | gui | specs/classes/CLS-registerScreen.spec.md |

# Acceptance criteria

- **AC1** — Given a filled, valid form, When the user clicks **Register** and the
  feature returns `Ok`, Then the app navigates to `/welcome`.
- **AC2** — Given the feature returns `Err`, When the call resolves, Then the
  screen shows the error message and stays on the form.
