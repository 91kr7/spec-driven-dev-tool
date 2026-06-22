---
id: COMP-panel
name: Panel
kind: gui
layer: organism
module: MOD-ui
status: reviewed
depends_on: []
requirements: []
source: [src/ui/organism/Panel]
variants: [default, outlined, elevated]
---

# Purpose

A card/panel container that groups related content under an optional titled
header, with a body and an optional footer (e.g. action buttons). It is the
default surface screens use to box a form, a list, or a summary, so screens do
not re-describe a card. It can optionally collapse its body to just the header.
It is an organism with three composition slots; callers fill them with
lower-layer content by id.

# Public interface

- **Inputs:** the `header`, `body`, `footer` slots and the `title`, `variant`,
  `collapsible`, `defaultCollapsed` props.
- **Outputs:** a bounded surface containing a header band, a content body, and an
  optional footer band.
- **Errors:** none. With no `title`/`header` content the header band is omitted;
  with no `footer` the footer band is omitted.

# Invariants & rules

- The header band renders when either a `title` prop or `header` slot content is
  present; otherwise it is omitted.
- When `collapsible` is true, the header carries a single toggle that expands or
  collapses the `body`; `footer` follows the collapsed state (hidden when
  collapsed).
- `collapsed` state is local view state, initialized from `defaultCollapsed`.
- The panel is a surface only — it adds no business behavior; slotted controls
  own their own logic.

## Wireframe

```
┌──────────────────────── Panel (COMP-panel) · variant="outlined" ───────────┐
│  {title}                                                            [ ▾ ]   │  ← header (toggle when collapsible)
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│   {body: slotted content}                                                  │  ← body
│                                                                            │
├────────────────────────────────────────────────────────────────────────────┤
│                                               {footer: actions}            │  ← footer (optional)
└────────────────────────────────────────────────────────────────────────────┘
```

## Component tree

```
COMP-panel   props: { title, variant, collapsible, defaultCollapsed }
├─ [slot: header]  → title text and/or caller header content + collapse toggle
├─ [slot: body]    → caller content (forms, lists, summaries)
└─ [slot: footer]  → caller content (typically action controls)
```

## State

| Name        | Type | Initial             | Description                                  |
|-------------|------|---------------------|----------------------------------------------|
| `collapsed` | Bool | `{defaultCollapsed}`| Whether the body/footer are currently hidden.|

## Props

| Prop               | Type                              | Required | Default     | Description                                                  |
|--------------------|-----------------------------------|----------|-------------|--------------------------------------------------------------|
| `title`            | String                            | no       | —           | Header title text; renders the header band when present.     |
| `header`           | Slot                              | no       | —           | Custom header content (supplements/replaces the title text). |
| `body`             | Slot                              | no       | —           | Main panel content.                                          |
| `footer`           | Slot                              | no       | —           | Footer content, typically actions.                           |
| `variant`          | `default \| outlined \| elevated` | no       | `default`   | Surface treatment of the panel.                              |
| `collapsible`      | Bool                              | no       | `false`     | When true, the header toggles the body open/closed.          |
| `defaultCollapsed` | Bool                              | no       | `false`     | Initial collapsed state (only meaningful when collapsible).  |

## Variants

- `default` — plain surface with no border emphasis (use for inline grouping).
- `outlined` — bordered surface (use to delineate a section on a flat page).
- `elevated` — raised surface (use to lift a card above the page, e.g. a modal
  body or a highlighted summary).

## Visual states

- `default` — header (if any), body, and footer (if any) all visible.
- `hover` — applies only when `collapsible`: the header toggle indicates it is
  interactive.
- `focus` — the collapse toggle (when present) shows a focus indicator.
- `collapsed` — body and footer are hidden; only the header band remains.

(Surface treatment per variant is expressed as token **names**; concrete
border/shadow values resolve from `.sdd/target.md`.)

## Slots / children

- `header` — title/header band content (plus the auto-added collapse toggle).
- `body` — main content region.
- `footer` — trailing action/content band.

## Events

| Event      | Payload              | When                                              |
|------------|----------------------|---------------------------------------------------|
| `onToggle` | `{ collapsed: Bool }`| user activates the collapse toggle (collapsible). |

Handler for the collapse toggle (non-trivial because it flips local state and
emits):

```
# handler: toggle
FUNCTION toggle() -> Void
  [B1] IF collapsed THEN
    collapsed <- false        # arm B1.then — expand
  ELSE
    collapsed <- true         # arm B1.else — collapse
  END
  EMIT onToggle({ collapsed: collapsed })
END
```

## Accessibility

- The panel is a labeled region: when a `title` is present it labels the panel so
  assistive tech announces the group by name.
- When `collapsible`, the header toggle is a button: reachable by Tab, activated
  by Enter/Space, exposes its expanded/collapsed state, and identifies the body
  region it controls.
- Focus order: header (toggle) → body → footer.

# Acceptance criteria

- **AC1** — Given a `title` and `body` content, When the panel renders, Then a
  header band with the title is shown above the body content.
- **AC2** — Given no `title` and no `header` slot, When the panel renders, Then
  the header band is omitted and only the body (and footer, if any) show.
- **AC3** — Given `variant` is `elevated`, When the panel renders, Then the
  elevated surface treatment is applied (per the design tokens).
- **AC4** — Given `collapsible` is true and the panel is expanded, When the user
  activates the header toggle, Then `collapsed` becomes true, the body and footer
  are hidden, and `onToggle` is emitted with `{ collapsed: true }` (covers
  `toggle#B1.else`).
- **AC5** — Given `collapsible` is true and the panel is collapsed, When the user
  activates the header toggle, Then `collapsed` becomes false, the body and footer
  are shown, and `onToggle` is emitted with `{ collapsed: false }` (covers
  `toggle#B1.then`).
- **AC6** — Given `collapsible` is true, When assistive technology inspects the
  header toggle, Then it is exposed as a button reporting its expanded/collapsed
  state and the body region it controls.
