---
id: COMP-stack
name: Stack
kind: gui
layer: layout
module: MOD-ui
status: reviewed
depends_on: []
requirements: []
source: [src/ui/layout/Stack]
variants: [default]
---

# Purpose

A one-dimensional flex layout helper that arranges its children in a single row
or column with a consistent gap and configurable alignment, justification, and
wrapping. It is the default tool for spacing groups of elements (form fields,
button rows, list items) so screens never hand-roll spacing. It is a layout
primitive holding an arbitrary `children` slot.

# Public interface

- **Inputs:** the `children` slot plus `direction`, `gap`, `align`, `justify`,
  `wrap` props.
- **Outputs:** its children laid out along one axis with the requested spacing
  and alignment.
- **Errors:** none — pure layout. An empty `children` slot renders nothing.

# Invariants & rules

- Lays out along exactly **one** axis: `row` (horizontal) or `column` (vertical).
- The same `gap` token is applied uniformly between adjacent children only (no
  leading/trailing gap).
- `align` controls the cross-axis; `justify` controls the main axis.
- When `wrap` is false (default), children stay on a single line/column; when
  true, they wrap onto additional lines using the same `gap`.
- It carries no semantics of its own — it is a generic grouping container.

## Wireframe

```
direction="row", gap="md", justify="end"            direction="column", gap="sm"
┌──────────────────────────────────────┐            ┌──────────────────┐
│              [child] gap [child] gap[child]│        │ [child]          │
└──────────────────────────────────────┘            │  gap             │
                                                     │ [child]          │
                                                     │  gap             │
                                                     │ [child]          │
                                                     └──────────────────┘
```

## Component tree

```
COMP-stack   props: { direction, gap, align, justify, wrap }
└─ [slot: children]  → one or more caller-supplied elements/components
```

## State

| Name | Type | Initial | Description |
|------|------|---------|-------------|
| — | — | — | Stateless; children are supplied via the `children` slot. |

## Props

| Prop        | Type                                                      | Required | Default    | Description                                          |
|-------------|----------------------------------------------------------|----------|------------|------------------------------------------------------|
| `children`  | Slot                                                     | no       | —          | The elements to lay out along one axis.              |
| `direction` | `row \| column`                                          | no       | `column`   | Main-axis direction.                                 |
| `gap`       | `none \| xs \| sm \| md \| lg \| xl`                     | no       | `md`       | Spacing token applied between adjacent children.     |
| `align`     | `start \| center \| end \| stretch \| baseline`          | no       | `stretch`  | Cross-axis alignment of children.                    |
| `justify`   | `start \| center \| end \| between \| around \| evenly`  | no       | `start`    | Main-axis distribution of children.                  |
| `wrap`      | Bool                                                     | no       | `false`    | Whether children wrap onto additional lines.         |

## Variants

- `default` — the single configurable stack. Row vs. column behavior is selected
  by the `direction` prop, not by separate variants.

## Visual states

- `default` — children arranged along the chosen axis with the chosen spacing.

(All spacing/alignment values are token **names**; concrete values resolve from
`.sdd/target.md`.)

## Slots / children

- `children` — the laid-out elements.

## Events

| Event | Payload | When |
|-------|---------|------|
| — | — | The stack emits no events; it is a passive container. |

## Accessibility

- Purely presentational: it adds no role or landmark and does not alter the DOM
  order of its children, so reading order and focus order match source order.
- When `direction` is `row`, arrow-key navigation among interactive children
  follows the visual order; the stack itself adds no key handling.

# Acceptance criteria

- **AC1** — Given `direction` is `row` and three children, When the stack
  renders, Then the children are laid out horizontally in source order with a
  uniform gap between adjacent ones.
- **AC2** — Given `direction` is `column` (default) and three children, When the
  stack renders, Then the children are stacked vertically with a uniform gap.
- **AC3** — Given `justify` is `end` in a row, When the stack renders, Then the
  group of children is pushed to the end of the main axis.
- **AC4** — Given `wrap` is true and children exceed the available width, When the
  stack renders, Then children wrap onto a new line using the same gap.
- **AC5** — Given the stack renders, When assistive technology reads the page,
  Then the children appear in the same order as the source with no added roles.
