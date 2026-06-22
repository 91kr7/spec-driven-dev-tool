---
id: COMP-appShell
name: AppShell
kind: gui
layer: layout
module: MOD-ui
status: reviewed
depends_on: [COMP-header, COMP-body, COMP-footer]
requirements: []
source: [src/ui/layout/AppShell]
variants: [default]
---

# Purpose

The top-level application frame. It establishes the canonical vertical layout
shared by every screen: a fixed top bar, a flexible middle region that fills the
remaining viewport height, and a bottom bar. It is the one place a screen plugs
its content into; feature screens compose `COMP-appShell` and fill its slots
rather than re-describing the page chrome. It composes lower layers **by id** and
never redefines header, body, or footer.

# Public interface

- **Inputs:** the three composition slots (`header`, `body`, `footer`) plus the
  optional props below.
- **Outputs:** a full-height page region that arranges its three slots top /
  middle / bottom.
- **Errors:** none — a pure layout container. A missing `header` or `footer` slot
  collapses that band to zero height (no error); a missing `body` slot renders an
  empty middle region.

# Invariants & rules

- The shell occupies the full available viewport height; `body` absorbs all
  height not used by `header` and `footer`.
- Only `header` / `footer` are pinned to the top / bottom; the `body` region is
  the only scroll owner (the shell itself never scrolls).
- The shell defines **structure only** — no spacing, color, or typography of its
  own beyond layout; visual detail belongs to the slotted components.

## Wireframe

```
┌──────────────────────── AppShell (COMP-appShell) ────────────────────────┐
│  ┌──────────────────── slot: header (COMP-header) ────────────────────┐  │
│  │  ◀ Brand                     Nav                       Actions      │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────── slot: body (COMP-body) ───────────────────────┐   │
│  │                                                                    │   │
│  │                       {screen content}                  ▲ scrolls  │   │
│  │                                                         ▼          │   │
│  └────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────── slot: footer (COMP-footer) ───────────────────┐   │
│  │  © {year} Brand                                       v{version}   │   │
│  └────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

## Component tree

```
COMP-appShell
├─ [slot: header]  → COMP-header   props: { brand, nav, actions }
├─ [slot: body]    → COMP-body     props: { maxWidth, padding }
│  └─ {screen content provided by the composing screen}
└─ [slot: footer]  → COMP-footer   props: { version }
```

## State

| Name | Type | Initial | Description |
|------|------|---------|-------------|
| — | — | — | Stateless; all content is supplied via slots. |

## Props

| Prop          | Type     | Required | Default | Description                                              |
|---------------|----------|----------|---------|----------------------------------------------------------|
| `header`      | Slot     | no       | —       | Content for the top band (typically a `COMP-header`).    |
| `body`        | Slot     | no       | —       | Main content for the middle region (a `COMP-body`).      |
| `footer`      | Slot     | no       | —       | Content for the bottom band (typically a `COMP-footer`). |
| `fullHeight`  | Bool     | no       | `true`  | When true, the shell stretches to the full viewport height. |

## Variants

- `default` — the standard top / middle / bottom three-band frame. The library
  intentionally ships a single shell variant; new page archetypes (e.g. a
  sidebar layout) are promoted as separate layout components, not as shell
  variants.

## Visual states

- `default` — three bands laid out vertically, `body` filling the gap.
- `error` — not applicable; the shell renders whatever its slots contain. Slot
  components surface their own error states.

(Layout only: no color/pixel detail here — design tokens are resolved at
implementation time from `.sdd/target.md`.)

## Slots / children

- `header` — top band, natural height.
- `body` — middle band, grows to fill; the sole scroll region.
- `footer` — bottom band, natural height.

## Events

| Event | Payload | When |
|-------|---------|------|
| — | — | The shell emits no events; it is a passive container. |

## Accessibility

- Renders three page landmarks in DOM order: `header` (banner), `body` (main),
  `footer` (contentinfo), so assistive tech and "skip to content" work.
- The `body` slot carries the page's primary landmark and is the natural target
  of a skip-to-content control placed in the header.
- Focus order follows DOM order: header → body → footer.
- The shell adds no focusable elements of its own.

# Acceptance criteria

- **AC1** — Given a screen that supplies all three slots, When the shell renders,
  Then header is pinned at the top, footer at the bottom, and body fills the
  vertical space between them.
- **AC2** — Given content in `body` taller than the viewport, When the user
  scrolls, Then only the `body` region scrolls while header and footer remain
  fixed in place.
- **AC3** — Given the `footer` slot is omitted, When the shell renders, Then the
  bottom band collapses to zero height and `body` extends to the viewport bottom,
  with no error raised.
- **AC4** — Given assistive technology inspects the page, When it lists
  landmarks, Then exactly one banner, one main, and one contentinfo region are
  exposed in header → body → footer order.
