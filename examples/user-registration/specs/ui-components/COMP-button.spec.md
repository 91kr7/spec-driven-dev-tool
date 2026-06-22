<!--
  FROZEN, ILLUSTRATIVE EXAMPLE — NOT part of the tool's real specs/.
  Part of the "User registration (Database + API)" worked example.
  Delete examples/user-registration/ before real use.
  Authority: .sdd/conventions.md (ids §2, front-matter §3),
             .sdd/ui-schema.md (§6 shared-component sections, §7 layers),
             .sdd/scot.md (the non-trivial onClick guard carries a SCoT snippet).
  Promoted by the reuse-analyst from FEAT-001. No framework code, no CSS — token names only.
  Markdown is the source of truth (authority); reuse over repetition (DRY).
-->
---
id: COMP-button
name: Button
kind: gui
module: MOD-web
layer: atom
status: reviewed
depends_on: []
requirements: [REQ-001]
source: [src/web/ui/Button.tsx]
variants: [primary, secondary, ghost, danger]
---

# Purpose

A single, reusable clickable control that triggers one action. It is the library's
canonical button atom: every actionable button — Register, Cancel, a destructive
confirm, a low-emphasis action — is this component with a different `variant`, so
styling and accessibility behavior stay defined in exactly one place (DRY).

# Wireframe

```
default                    loading                     disabled
┌───────────────┐          ┌───────────────┐           ┌───────────────┐
│   {label}     │          │  ◐ {label}    │           │   {label}     │   (dimmed,
└───────────────┘          └───────────────┘           └───────────────┘    no click)
   [ Register ]               [ ◐ Register ]              [ Register ]
```

# Composition / slots

```
COMP-button
└─ (atom — no child components; the loading spinner is an internal cue)
```

Slots: `none` — the label is supplied via the `label` prop, not a child slot.

# Props

| Prop       | Type                                       | Required | Default     | Description                                               |
|------------|--------------------------------------------|----------|-------------|-----------------------------------------------------------|
| `label`    | `String`                                   | yes      | —           | Visible button text (also its accessible name).           |
| `variant`  | `primary \| secondary \| ghost \| danger`  | no       | `primary`   | Visual emphasis / intent of the action.                   |
| `disabled` | `Bool`                                     | no       | `false`     | Blocks all interaction; the control is not activatable.   |
| `loading`  | `Bool`                                     | no       | `false`     | Shows a busy cue and blocks activation while a task runs. |
| `onClick`  | `Event`                                    | no       | —           | Emitted when the user activates an enabled, idle button.  |

# Variants

- `primary` — the single main / affirmative action of a view (e.g. **Register**).
- `secondary` — a supporting action shown alongside the primary one (e.g. **Cancel**).
- `ghost` — a low-emphasis action that should not compete visually (e.g. inline **Edit**).
- `danger` — a destructive or irreversible action (e.g. **Delete account**).

# Visual states

- `default` — resting appearance for the chosen `variant`; activatable.
- `hover` — pointer-over emphasis cue; activatability unchanged.
- `focus` — visible keyboard-focus indicator when reached via Tab.
- `active` — pressed-state cue while the pointer/key is held down.
- `disabled` — dimmed and non-interactive; `onClick` is never emitted; not in the tab order.
- `loading` — a busy cue is shown beside the label and activation is blocked; `onClick` is suppressed.
- `error` — not a state of this atom; error presentation belongs to `COMP-formField`.

# Events

| Event     | Payload | When                                                                  |
|-----------|---------|-----------------------------------------------------------------------|
| `onClick` | `{}`    | the user activates the button (pointer click or Enter/Space) while it is neither `disabled` nor `loading`. |

```
# handler: activate  (guards the onClick emission)
FUNCTION activate() -> Void
  [B1] IF disabled OR loading THEN
    RETURN                       # arm B1.then — suppressed, no event emitted
  ELSE
    EMIT onClick({})             # arm B1.else — propagate activation to the parent
  END
END
```

# Accessibility

- Role: native button semantics (an activatable control with an accessible name equal to `label`).
- Keyboard: reachable by Tab when enabled; **Enter** and **Space** both activate it; removed from the tab order when `disabled`.
- ARIA: reflects `aria-disabled` while `disabled`, and `aria-busy` while `loading`.

# Acceptance criteria

- **AC1** — Given an enabled, idle button, When the user activates it via pointer
  click or the Enter/Space key, Then exactly one `onClick` event with an empty
  payload is emitted. *(covers B1.else)*
- **AC2** — Given a button that is `disabled` or `loading`, When the user attempts
  to activate it by any means, Then no `onClick` event is emitted, and a `loading`
  button additionally exposes `aria-busy`. *(covers B1.then)*
