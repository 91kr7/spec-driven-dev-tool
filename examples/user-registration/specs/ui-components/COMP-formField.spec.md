<!--
  FROZEN, ILLUSTRATIVE EXAMPLE — NOT part of the tool's real specs/.
  Part of the "User registration (Database + API)" worked example.
  Delete examples/user-registration/ before real use.
  Authority: .sdd/conventions.md (ids §2, front-matter §3),
             .sdd/ui-schema.md (§6 shared-component sections, §7 layers — molecule composes atoms BY ID).
  Promoted by the reuse-analyst from FEAT-001. A molecule never re-describes its atoms.
  Markdown is the source of truth (authority); reuse over repetition (DRY).
-->
---
id: COMP-formField
name: FormField
kind: gui
module: MOD-web
layer: molecule
status: reviewed
depends_on: [COMP-textInput]
requirements: [REQ-001]
source: [src/web/ui/FormField.tsx]
---

# Purpose

A molecule that wraps one labelled form control with its inline validation text.
It pairs a `label`, a control slot (typically a `COMP-textInput`), and an error
message into a single accessible unit, so every field in a form gets consistent
labelling, required-marking, and error presentation in one place (DRY).

# Wireframe

```
┌──────────────────────────────────────────────┐
│ {label} *            (the * shows if required)│
│ [____________ control slot ____________]      │
│ {error}             (shown only when present) │
└──────────────────────────────────────────────┘
```

# Composition / slots

```
COMP-formField
└─ control            slot: { holds one input control, typically COMP-textInput }
```

Slots: `control` — the single form control this field labels (the screen places a
`COMP-textInput` here by id). The label and error text are owned by this molecule;
the control is supplied via the slot, never re-described here.

# Props

| Prop       | Type      | Required | Default | Description                                                  |
|------------|-----------|----------|---------|--------------------------------------------------------------|
| `label`    | `String`  | yes      | —       | The field's visible label text (and the control's a11y name).|
| `required` | `Bool`    | no       | `false` | Marks the field as required (shows a required indicator).    |
| `error`    | `String`  | no       | `""`    | Inline validation message; shown only when non-empty.        |
| `htmlFor`  | `String`  | no       | —       | Associates the label with the slotted control's identity.    |

# Variants

- This molecule exposes no named visual variants (`variants:` is empty); its
  appearance is driven by `required` and the presence of `error`.

# Visual states

- `default` — label above the control; no error text shown.
- `error` — the `error` message is shown beneath the control and the field signals
  invalidity to its control (which reflects `aria-invalid`).
- `disabled` — when the slotted control is disabled, the field is presented dimmed
  (the control owns the disabled behavior).

# Events

| Event | Payload | When                                                          |
|-------|---------|---------------------------------------------------------------|
| —     | —       | This molecule emits no events of its own; interaction events come from the slotted control (e.g. `COMP-textInput.onChange`). |

# Accessibility

- Role: a labelled form-field grouping; the `label` is programmatically associated
  with the slotted control (via `htmlFor` / the control's identity).
- Keyboard: focus flows to the slotted control; the label is not separately
  focusable.
- ARIA: when `error` is non-empty the field marks the control `aria-invalid` and
  associates the error text as the control's description; `required` is reflected
  to the control.

# Acceptance criteria

- **AC1** — Given a `COMP-formField` with a `label` and a non-empty `error`, When
  it renders, Then the error text is shown beneath the control and the control
  reflects `aria-invalid`.
- **AC2** — Given a `COMP-formField` with `required = true` and an empty `error`,
  When it renders, Then the required indicator is shown, no error text appears,
  and the `label` is associated with the slotted control.
