<!--
  TEMPLATE — shared UI component (COMP-*, kind: gui). Copy to .sdd/specs/ui-components/COMP-<lowerCamel>.spec.md.
  Authority: conventions §2/§3/§5 + ui-schema §6 (the EXTRA sections a COMP-* adds) + §7 (layers) + scot.md (only a non-trivial handler).
  Discover before create: read ui-components.index.md first; a higher layer composes lower layers BY ID, never re-described.
  No framework code, no CSS/colors/pixels — only design-token names. Markdown is the source of truth; reuse over repetition (DRY).
  Delete the "## Filled example".
-->
---
id: COMP-<lowerCamel>         # required — matches filename + a ui-components.index row
name: <HumanName>            # required
kind: gui                    # required — always gui for a UI component
module: MOD-<kebab>          # required — e.g. MOD-ui
layer: <atom|molecule|organism|layout>   # required (ui-schema §7)
depends_on: [<COMP-id>]      # LOWER-layer components this composes; [] if none
requirements: [REQ-<nnn>]    # back-link
source: [<src/path/Component.ext>]
variants: [<variantA>, <variantB>]   # optional — named visual variants
---

# Purpose
<One paragraph: the single reusable responsibility. No HOW, no styling detail.>

# Wireframe
<ASCII per ui-schema §2 (indicative). Bind dynamic text with {propName}.>
```
<ascii of the default visual state>
```

# Composition / slots
<Tree per ui-schema §3: compose lower-layer COMP-* BY ID; an atom usually has none. Name any slots.>
```
COMP-<lowerCamel>
└─ <COMP-childId>  props: { … }     # or: (atom — no child components)
```
Slots: `<slotName>` — <what goes in it>.   <!-- or: none (leaf) -->

# Props
| Prop | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `<prop>` | `<Type>` (literal union for variants) | <yes/no> | `<default>` | <what it controls> |

# Variants
<Each named variant + when to use it (intent, no colors/pixels).>
- `<variantA>` — <when>.

# Visual states
<For each applicable state, WHAT CHANGES behaviorally (no token values). Set: default/hover/focus/active/disabled/loading/error.>
- `default` — <…> · `disabled` — <interaction blocked> · `loading` — <spinner; activation blocked>

# Events
<name | payload | when. A non-trivial handler gets a small SCoT snippet (scot.md) with branch ids.>
| Event | Payload | When |
|-------|---------|------|
| `<onX>` | `<payload>` | <condition> |

# Accessibility
<Roles, keyboard, focus order, ARIA intent — as behavior; testable items become ACs.>
- Role: <…> · Keyboard: <reachability + activation keys> · ARIA: <state attrs, e.g. aria-busy/aria-disabled>

# Acceptance criteria
- **AC1** — Given <context>, When <action>, Then <observable outcome>.
- **AC2** — Given <context>, When <action>, Then <observable outcome>.

---

## Filled example — `COMP-button` (atom)

```yaml
id: COMP-button · name: Button · kind: gui · module: MOD-ui · layer: atom
depends_on: [COMP-spinner] · requirements: [REQ-014]
source: [src/ui/components/Button.tsx] · variants: [primary, secondary, ghost, danger]
```

**Purpose** — The library's canonical clickable control; every actionable button is this atom with a different `variant`, so styling + a11y stay in one place.

**Composition** — `COMP-button └─ COMP-spinner props: { size: "sm" }  # only while loading`. Slots: none (label via prop).

**Props** — `label: String` (req) · `variant: primary|secondary|ghost|danger` (`primary`) · `disabled: Bool` (`false`) · `loading: Bool` (`false`) · `type: button|submit` (`button`) · `onClick: Event`.

**Variants** — primary (main action) · secondary (supporting, e.g. Cancel) · ghost (low-emphasis) · danger (destructive; pair with confirm).

**Visual states** — disabled: dimmed, not activatable, not in tab order, no `onClick`; loading: shows `COMP-spinner`, activation suppressed; error: owned by the enclosing `COMP-formField`, not this atom.

**Events** — | `onClick` | `{}` | user activates an enabled, idle button (click or Enter/Space) |
```
# handler: activate
FUNCTION activate() -> Void
  [B1] IF disabled OR loading THEN RETURN   # B1.then — suppressed
  ELSE EMIT onClick({}) END                 # B1.else — propagate
END
```

**Accessibility** — button role, name = `label`; Tab-reachable when enabled; Enter/Space activate; reflects `aria-disabled` / `aria-busy`.

- **AC1** — Given an enabled idle button, When activated by click or Enter/Space, Then exactly one `onClick` with empty payload is emitted.
- **AC2** — Given a `disabled` or `loading` button, When activation is attempted, Then no `onClick`; a loading button shows `COMP-spinner` and exposes `aria-busy`.
