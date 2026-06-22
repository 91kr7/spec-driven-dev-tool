<!--
  FROZEN, ILLUSTRATIVE EXAMPLE — NOT part of the tool's real specs/.
  Part of the "User registration (Database + API)" worked example.
  Delete examples/user-registration/ before real use.
  Authority: .sdd/conventions.md (ids §2, front-matter §3),
             .sdd/ui-schema.md (§6 shared-component sections, §7 layers).
  Promoted by the reuse-analyst from FEAT-001. No framework code, no CSS — token names only.
  Markdown is the source of truth (authority); reuse over repetition (DRY).
-->
---
id: COMP-textInput
name: TextInput
kind: gui
module: MOD-web
layer: atom
status: reviewed
depends_on: []
requirements: [REQ-001]
source: [src/web/ui/TextInput.tsx]
---

# Purpose

A single, reusable single-line text-entry atom. It is the library's canonical
controlled input: every text/email/password field in the app is this component
with a different `type`, so entry behavior and accessibility stay defined in
exactly one place (DRY).

# Wireframe

```
default                     focus                       disabled
[____________________]      [|___________________]      [____________________]  (dimmed,
   placeholder shown           caret active                no input)
```

# Composition / slots

```
COMP-textInput
└─ (atom — no child components)
```

Slots: `none` (leaf component).

# Props

| Prop          | Type                                  | Required | Default  | Description                                          |
|---------------|---------------------------------------|----------|----------|------------------------------------------------------|
| `type`        | `text \| email \| password`           | no       | `text`   | Input mode for the entered value.                    |
| `value`       | `String`                              | yes      | —        | The controlled current value.                        |
| `placeholder` | `String`                              | no       | `""`     | Hint text shown when the value is empty.             |
| `disabled`    | `Bool`                                | no       | `false`  | Blocks input; the field is not editable.             |
| `onChange`    | `Event`                               | no       | —        | Emitted with the new value when the user types.      |

# Variants

- This atom exposes no named visual variants (`variants:` is empty); `type`
  selects the input mode, not a visual variant.

# Visual states

- `default` — resting field showing `value` or the `placeholder`; editable.
- `focus` — visible keyboard-focus indicator; caret active for editing.
- `disabled` — dimmed and non-editable; `onChange` is never emitted.
- `error` — error presentation is owned by the enclosing `COMP-formField`; this
  atom only reflects an invalid styling cue when its field reports an error.

# Events

| Event      | Payload          | When                                            |
|------------|------------------|-------------------------------------------------|
| `onChange` | `{ value }`      | the user edits the field while it is not `disabled`. |

# Accessibility

- Role: native text-input semantics; its accessible name comes from the label of
  the enclosing `COMP-formField`.
- Keyboard: reachable by Tab when enabled; standard text editing keys apply;
  removed from the tab order when `disabled`.
- ARIA: reflects `aria-disabled` while `disabled`; reflects `aria-invalid` when
  the enclosing field reports an error.

# Acceptance criteria

- **AC1** — Given an enabled `COMP-textInput`, When the user types into it, Then
  an `onChange` event carrying the new `{ value }` is emitted.
- **AC2** — Given a `disabled` `COMP-textInput`, When the user attempts to type,
  Then the value is unchanged and no `onChange` event is emitted.
