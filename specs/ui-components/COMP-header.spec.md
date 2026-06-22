---
id: COMP-header
name: Header
kind: gui
layer: organism
module: MOD-ui
status: reviewed
depends_on: []
requirements: []
source: [src/ui/organism/Header]
variants: [default]
---

# Purpose

The application top bar. It hosts the brand/logo, primary navigation, and a
trailing actions/user region. It is the canonical banner used at the top of
`COMP-appShell` so screens never re-describe the page header. It accepts brand,
nav, and actions content via props/slots and composes no lower-layer component by
default (atoms like a future `COMP-button` or `COMP-menu` are slotted in by the
caller).

# Public interface

- **Inputs:** `brand`, `nav`, `actions` content (props/slots).
- **Outputs:** a horizontal bar with brand at the start, nav in the middle, and
  actions/user at the end.
- **Errors:** none. Any omitted region simply renders empty.

# Invariants & rules

- Exactly one banner per page; `COMP-appShell` places it in its `header` slot.
- Layout order is always brand → nav → actions, left-to-right (mirrored in RTL).
- The header has a natural (content-driven) height and does not scroll.
- It describes structure and the three regions only; the concrete nav items and
  action controls are supplied by the caller, not redefined here.

## Wireframe

```
┌──────────────────────────── Header (COMP-header) ───────────────────────────┐
│  ◀ {brand}        {nav: Home · Account · Reports}            {actions} {user}▼│
└──────────────────────────────────────────────────────────────────────────────┘
```

## Component tree

```
COMP-header
├─ [slot: brand]    → brand/logo content (e.g. a logo image + product name)
├─ [slot: nav]      → primary navigation items (caller-supplied)
└─ [slot: actions]  → trailing actions + user menu (caller-supplied)
```

## State

| Name | Type | Initial | Description |
|------|------|---------|-------------|
| — | — | — | Stateless; all content is supplied via props/slots. |

## Props

| Prop      | Type             | Required | Default | Description                                             |
|-----------|------------------|----------|---------|---------------------------------------------------------|
| `brand`   | Slot \| String   | no       | —       | Brand/logo region content at the start of the bar.      |
| `nav`     | Slot             | no       | —       | Primary navigation region content.                      |
| `actions` | Slot             | no       | —       | Trailing region for action controls (right-aligned).    |
| `user`    | Slot             | no       | —       | Optional user/account affordance shown within `actions`.|
| `sticky`  | Bool             | no       | `true`  | Keeps the header pinned to the top while body scrolls.  |

## Variants

- `default` — full bar with brand, nav, and actions. Additional archetypes
  (e.g. a condensed/mobile header) are added as variants only when a real screen
  needs them, per progressive enrichment.

## Visual states

- `default` — all three regions visible and laid out horizontally.
- `focus` — a focused interactive child within the header shows a focus
  indicator (delegated to the child control).

(Layout/structure only; spacing and color resolve from design tokens at
implementation time.)

## Slots / children

- `brand` — leading region (logo/product name).
- `nav` — central navigation region.
- `actions` — trailing region for buttons and the `user` affordance.

## Events

| Event | Payload | When |
|-------|---------|------|
| — | — | The header itself emits no events; slotted controls emit their own. |

## Accessibility

- Renders as the page `banner` landmark; there is exactly one per page.
- The `nav` region is exposed as a navigation landmark so it is discoverable and
  skippable by assistive tech.
- All interactive children are reachable by Tab in brand → nav → actions order;
  Enter/Space activate them (behavior owned by the child controls).

# Acceptance criteria

- **AC1** — Given `brand`, `nav`, and `actions` are supplied, When the header
  renders, Then brand appears at the start, nav in the middle, and actions at the
  end of a single horizontal bar.
- **AC2** — Given `sticky` is true and the body scrolls, When the user scrolls
  the page, Then the header remains pinned at the top.
- **AC3** — Given the `nav` slot is empty, When the header renders, Then the nav
  region collapses without affecting brand/actions placement and no error occurs.
- **AC4** — Given assistive technology inspects the page, When it lists
  landmarks, Then the header is exposed as a single banner and its nav region as a
  navigation landmark.
