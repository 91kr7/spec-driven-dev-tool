---
id: COMP-body
name: Body
kind: gui
layer: layout
module: MOD-ui
status: reviewed
depends_on: []
requirements: []
source: [src/ui/layout/Body]
variants: [default]
---

# Purpose

The main content region of `COMP-appShell`, sitting between the header and the
footer. It owns the page's vertical scroll, optionally constrains its content to
a maximum readable width, and applies consistent content padding. Screens render
their content into its single `children` slot. It is a layout primitive and
composes no lower-layer component.

# Public interface

- **Inputs:** the `children` slot plus `maxWidth` and `padding` props.
- **Outputs:** a scrollable region that fills the height left by header/footer and
  centers/pads its content.
- **Errors:** none — pure layout. An empty `children` slot renders an empty
  scroll region.

# Invariants & rules

- It is the **sole scroll owner** of the page; the shell and the header/footer
  bands never scroll.
- It grows to fill all vertical space not consumed by header and footer.
- When `maxWidth` is set, content is constrained to that width and centered
  horizontally; the surrounding gutters remain part of the scroll region.
- It carries the page's primary `main` landmark.

## Wireframe

```
┌──────────────────────────── Body (COMP-body) ──────────────────────────────┐
│ │← gutter →│ ┌──── content (maxWidth) ────┐ │← gutter →│                  ▲ │
│            │ │  padding                    │                              │ │
│            │ │     {children}              │                       scrolls│ │
│            │ │                             │                              ▼ │
│            │ └─────────────────────────────┘                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

## Component tree

```
COMP-body
└─ [slot: children]  → screen content (caller-supplied)
```

## State

| Name | Type | Initial | Description |
|------|------|---------|-------------|
| — | — | — | Stateless; content is supplied via the `children` slot. |

## Props

| Prop       | Type                                         | Required | Default  | Description                                                         |
|------------|----------------------------------------------|----------|----------|---------------------------------------------------------------------|
| `children` | Slot                                         | no       | —        | The main page content to render and scroll.                         |
| `maxWidth` | `none \| sm \| md \| lg \| xl`               | no       | `lg`     | Maximum content width token; `none` lets content fill the region.   |
| `padding`  | `none \| sm \| md \| lg`                     | no       | `md`     | Inner padding token applied around the content.                     |

## Variants

- `default` — width-constrained, padded, scrollable content region. A future
  full-bleed variant is achieved with `maxWidth: none` rather than a new variant.

## Visual states

- `default` — content laid out within the optional max-width and padding.
- `scrolling` — a vertical scrollbar appears when content exceeds the region
  height; header/footer remain fixed.

(Width and padding are token **names** only; concrete values resolve from
`.sdd/target.md` at implementation time.)

## Slots / children

- `children` — the single content slot for the screen.

## Events

| Event | Payload | When |
|-------|---------|------|
| — | — | The body emits no events; it only owns scrolling. |

## Accessibility

- Exposed as the page `main` landmark (exactly one per page).
- Is a natural skip-to-content target from a header control.
- Keeps content reachable: scrolling never traps keyboard focus; focusable
  children remain in DOM order.

# Acceptance criteria

- **AC1** — Given header and footer of fixed height, When the body renders, Then
  it fills exactly the remaining vertical space between them.
- **AC2** — Given content taller than the available height, When the user
  scrolls, Then only the body scrolls and a vertical scrollbar appears within it.
- **AC3** — Given `maxWidth` is `md`, When the body renders on a wide viewport,
  Then content is constrained to the `md` width token and centered, with gutters
  on both sides.
- **AC4** — Given `maxWidth` is `none`, When the body renders, Then content fills
  the full available width with no centering gutters.
- **AC5** — Given assistive technology inspects the page, When it lists
  landmarks, Then the body is exposed as the single main landmark.
