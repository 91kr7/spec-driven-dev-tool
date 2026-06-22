<!--
TEMPLATE: shared UI component spec (COMP-*, kind: gui).
Copy this file to specs/ui-components/COMP-<lowerCamel>.spec.md and replace every
<placeholder>. Delete the guidance comments (<!-- ... -->) before committing.

Canonical contracts (single source of truth — obey, do not restate):
- .claude/sdd/conventions.md  — ids (§2), front-matter (§3), status lifecycle (§5), rosters.
- .claude/sdd/ui-schema.md    — GUI spec form; §6 = the EXTRA sections a shared COMP-* adds;
                         §7 = atomic-design layers (atom|molecule|organism|layout).
- .claude/sdd/scot.md         — only if a non-trivial event handler needs a SCoT snippet.

VALUES THAT BIND THIS SPEC:
  Markdown is the source of truth (authority); reuse over repetition (DRY).
Discover before create: read specs/indexes/ui-components.index.md FIRST. A higher
layer composes lower layers BY ID; never re-describe a component that already exists.
This spec carries NO framework code, NO CSS/colors/pixels — only design-token names.
-->

---
id: COMP-<lowerCamel>          # required — matches filename + a ui-components.index row; e.g. COMP-button
name: <HumanName>             # required — human-readable component name; e.g. Button
kind: gui                     # required — always `gui` for a UI component
module: MOD-<kebab>           # required — the level's home module; e.g. MOD-ui
layer: <atom|molecule|organism|layout>   # required for COMP-* (ui-schema §7)
status: draft                 # draft|reviewed|approved — MIRRORS the index (index is canonical)
depends_on: [<COMP-id>, ...]  # ids of LOWER-layer components/specs this one composes; [] if none
requirements: [REQ-<nnn>]     # requirement id(s) this component serves (back-link)
source: [<src/path/to/Component.ext>]   # AUTHORITATIVE spec→source mapping (proposed for new, real for existing)
variants: [<variantA>, <variantB>]      # optional — the named visual variants this component exposes
---

# Purpose

<!-- One short paragraph: WHAT this component is and WHY it exists in the shared
library. State the single reusable responsibility. No HOW, no styling detail. -->
<One-paragraph statement of the component's reusable responsibility.>

# Wireframe

<!-- ASCII sketch per ui-schema §2. Indicative, not pixel-perfect. Mark interactive
elements: [ ] button, (•)/( ) radio, [x]/[ ] checkbox, ▼ dropdown, ___ text field.
Bind dynamic text with {propName}. -->

```
<ascii wireframe of the component in its default visual state>
```

# Composition / slots

<!-- Component tree per ui-schema §3: an indented tree that composes LOWER-layer
COMP-* BY ID with the props each is given. An atom usually has no children — say
so. Name any named slots (e.g. a Panel has `header`, `body`, `footer`). -->

```
COMP-<lowerCamel>
└─ <COMP-childId>   props: { <prop>: <value>, ... }     # or: "(atom — no child components)"
```

Slots: `<slotName>` — <what may be placed in it>.   <!-- or: "none (leaf component)" -->

# Props

<!-- ui-schema §6. One row per prop. `Type` may use a literal union for variants.
`Required` = yes|no. `Default` = the value used when omitted (— if required). -->

| Prop        | Type                              | Required | Default      | Description                          |
|-------------|-----------------------------------|----------|--------------|--------------------------------------|
| `<prop>`    | `<Type>`                          | <yes/no> | `<default>`  | <what it controls>                   |

# Variants

<!-- List each named variant from front-matter `variants:` and WHEN to use it.
Behavior/intent only — no colors/pixels (those resolve from design tokens). -->

- `<variantA>` — <when to use it>.
- `<variantB>` — <when to use it>.

# Visual states

<!-- ui-schema §6. For each applicable state describe WHAT CHANGES behaviorally.
Standard set: default, hover, focus, active, disabled, loading, error. No tokens'
concrete values here — only the named change. -->

- `default` — <appearance/behavior>.
- `hover` — <change>.
- `focus` — <change>.
- `active` — <change>.
- `disabled` — <change; interaction blocked>.
- `loading` — <change; e.g. spinner shown, activation blocked>.
- `error` — <change>.

# Events

<!-- ui-schema §6 events table. name | payload | when. For a non-trivial handler,
attach a small SCoT snippet (grammar = .claude/sdd/scot.md) with branch ids; trivial
events need only the row. -->

| Event     | Payload     | When                              |
|-----------|-------------|-----------------------------------|
| `<onX>`   | `<payload>` | <the user/system condition>       |

# Accessibility

<!-- ui-schema §6. Roles, keyboard interaction, focus order, ARIA intent — as
BEHAVIOR, not framework code. Anything testable becomes an AC below. -->

- Role: <semantic role>.
- Keyboard: <reachability + activation keys>.
- ARIA: <state attributes the component must reflect, e.g. aria-disabled / aria-busy>.

# Acceptance criteria

<!-- Each ACn in Given/When/Then. Cover the load-bearing behavior + a11y. -->

- **AC1** — Given <context>, When <action>, Then <observable outcome>.
- **AC2** — Given <context>, When <action>, Then <observable outcome>.

---

## Filled example

The block below is a complete, valid `COMP-button` spec authored from this
template. In a real spec it would live on its own at
`specs/ui-components/COMP-button.spec.md` with the front-matter at the top of the
file; it is reproduced here (front-matter shown as a YAML block) only to
demonstrate every section fully filled.

```yaml
---
id: COMP-button
name: Button
kind: gui
module: MOD-ui
layer: atom
status: draft
depends_on: [COMP-spinner]
requirements: [REQ-014]
source: [src/ui/components/Button.tsx]
variants: [primary, secondary, ghost, danger]
---
```

### Purpose

A single, reusable clickable control that triggers one action. It is the library's
canonical button atom: every actionable button in the app — submit, cancel,
destructive confirm, low-emphasis link-like action — is this component with a
different `variant`, so styling and accessibility behavior stay defined in exactly
one place (DRY).

### Wireframe

```
default                    loading                     disabled
┌───────────────┐          ┌───────────────┐           ┌───────────────┐
│   {label}     │          │  ◐ {label}    │           │   {label}     │   (dimmed,
└───────────────┘          └───────────────┘           └───────────────┘    no click)
   [ Register ]               [ ◐ Saving ]                [ Register ]
```

### Composition / slots

```
COMP-button
└─ COMP-spinner   props: { size: "sm" }     # rendered ONLY while loading == true
```

Slots: `none` — the label is supplied via the `label` prop, not a child slot.
(`COMP-spinner` is the only lower-layer atom this component composes, by id.)

### Props

| Prop        | Type                                          | Required | Default     | Description                                              |
|-------------|-----------------------------------------------|----------|-------------|----------------------------------------------------------|
| `label`     | `String`                                      | yes      | —           | Visible button text (also its accessible name).          |
| `variant`   | `primary \| secondary \| ghost \| danger`     | no       | `primary`   | Visual emphasis / intent of the action.                  |
| `disabled`  | `Bool`                                         | no       | `false`     | Blocks all interaction; control is not activatable.      |
| `loading`   | `Bool`                                         | no       | `false`     | Shows the spinner and blocks activation while a task runs.|
| `type`      | `button \| submit`                            | no       | `button`    | Whether the control submits an enclosing form.           |
| `onClick`   | `Event`                                        | no       | —           | Emitted when the user activates an enabled, idle button. |

### Variants

- `primary` — the single main / affirmative action of a view (e.g. **Register**, **Save**).
- `secondary` — a supporting action shown alongside the primary one (e.g. **Cancel**).
- `ghost` — a low-emphasis action that should not compete visually (e.g. inline **Edit**).
- `danger` — a destructive or irreversible action (e.g. **Delete account**); pair with a confirm flow.

### Visual states

- `default` — resting appearance for the chosen `variant`; activatable.
- `hover` — pointer-over emphasis cue; no change to activatability.
- `focus` — visible keyboard-focus indicator when reached via Tab.
- `active` — pressed-state cue while the pointer/key is held down.
- `disabled` — dimmed and non-interactive; `onClick` is never emitted; not in the tab order.
- `loading` — `COMP-spinner` is shown beside the label and activation is blocked; `onClick` is suppressed.
- `error` — not a state of this atom; error presentation is owned by the enclosing `COMP-formField` (molecule).

### Events

| Event     | Payload | When                                                                 |
|-----------|---------|----------------------------------------------------------------------|
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

### Accessibility

- Role: native button semantics (an activatable control with an accessible name equal to `label`).
- Keyboard: reachable by Tab when enabled; **Enter** and **Space** both activate it; removed from the tab order when `disabled`.
- ARIA: reflects `aria-disabled` while `disabled`, and `aria-busy` while `loading`; the spinner is decorative (not separately announced).

### Acceptance criteria

- **AC1** — Given an enabled, idle button, When the user activates it via pointer click or the Enter/Space key, Then exactly one `onClick` event with an empty payload is emitted.
- **AC2** — Given a button that is `disabled` or `loading`, When the user attempts to activate it by any means, Then no `onClick` event is emitted, and a `loading` button additionally shows `COMP-spinner` and exposes `aria-busy`.
