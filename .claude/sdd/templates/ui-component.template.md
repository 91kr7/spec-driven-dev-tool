<!--
<instructions>
TEMPLATE: shared UI component (COMP-*, kind: gui). Copy to `.sdd/specs/<MOD>/ui-components/COMP-<lowerCamel>.spec.md`.
Authority: conventions §2/§3/§5 + ui-schema §6 + §7 (layers) + scot.md.
Discover before create: read MOD-shared.index.md first. Higher layer composes lower layers BY ID.
No framework code, no CSS/colors/pixels — only design-token names. DRY.
Delete the `<example>` block before saving.
</instructions>
-->
---
id: COMP-<lowerCamel>         # required
name: <HumanName>             # required
kind: gui                     # required — always gui for a UI component
module: MOD-<kebab>           # required
layer: <atom|molecule|organism|layout> # required (ui-schema §7)
depends_on: [<COMP-id>]       # LOWER-layer components composed; [] if none
requirements: [REQ-<nnn>]     # back-link
source: [<src/path/Component.ext>]
variants: [<variantA>]        # optional — named visual variants
---

# Purpose
<instruction>The single reusable responsibility. No HOW, no styling detail.</instruction>

# Wireframe
<instruction>ASCII per ui-schema §2. Bind dynamic text with {propName}.</instruction>
```
<ascii of the default visual state>
```

# Composition / slots
<instruction>Tree per ui-schema §3: compose lower-layer COMP-* BY ID. Name any slots.</instruction>
```
COMP-<lowerCamel>
└─ <COMP-childId>  props: { … }     # or: (atom — no child components)
```
Slots: `<slotName>` — <what goes in it>.

# Props
| Prop | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `<prop>` | `<Type>` (literal union for variants) | <yes/no> | `<default>` | <what it controls> |

# Variants
<instruction>Each named variant + when to use it (intent, no colors/pixels).</instruction>
- `<variantA>` — <when>.

# Visual states
<instruction>For each applicable state, WHAT CHANGES behaviorally. Set: default/hover/focus/active/disabled/loading/error.</instruction>
- `default` — <…> · `disabled` — <interaction blocked> · `loading` — <spinner; activation blocked>

# Events
<instruction>name | payload | when. A non-trivial handler gets a small SCoT snippet (scot.md) with branch ids.</instruction>
| Event | Payload | When |
|-------|---------|------|
| `<onX>` | `<payload>` | <condition> |

# Accessibility
<instruction>Roles, keyboard, focus order, ARIA intent — as behavior; testable items become ACs.</instruction>
- Role: <…> · Keyboard: <reachability + activation keys> · ARIA: <state attrs>

# Acceptance criteria
- **AC1** — Given <context>, When <action>, Then <observable outcome>.
- **AC2** — Given <context>, When <action>, Then <observable outcome>.

---
<example name="COMP-button (atom)">
```yaml
id: COMP-button
name: Button
kind: gui
module: MOD-ui
layer: atom
depends_on: [COMP-spinner]
requirements: [REQ-014]
source: [src/ui/components/Button.tsx]
variants: [primary, secondary, ghost, danger]
```

**Purpose**
Canonical clickable control; every actionable button is this atom with a different `variant`, so styling + a11y stay in one place.

**Composition**
```
COMP-button
└─ COMP-spinner props: { size: "sm" }  # only while loading
```
Slots: none (label via prop).

**Props**
`label: String` (req) · `variant: primary|secondary|ghost|danger` (`primary`) · `disabled: Bool` (`false`) · `loading: Bool` (`false`) · `type: button|submit` (`button`) · `onClick: Event`.

**Variants**
primary (main action) · secondary (supporting) · ghost (low-emphasis) · danger (destructive).

**Visual states**
disabled: dimmed, not activatable, not in tab order, no `onClick`; loading: shows `COMP-spinner`, activation suppressed.

**Events**
| `onClick` | `{}` | user activates an enabled, idle button (click or Enter/Space) |
```
# handler: activate
FUNCTION activate() -> Void
  [B1] IF disabled OR loading THEN RETURN   # B1.then — suppressed
  ELSE EMIT onClick({}) END                 # B1.else — propagate
END
```

**Accessibility**
button role, name = `label`; Tab-reachable when enabled; Enter/Space activate; reflects `aria-disabled` / `aria-busy`.

**Acceptance criteria**
- **AC1** — Given an enabled idle button, When activated by click or Enter/Space, Then exactly one `onClick` with empty payload is emitted.
- **AC2** — Given a `disabled` or `loading` button, When activation is attempted, Then no `onClick`; a loading button shows `COMP-spinner` and exposes `aria-busy`.
</example>
