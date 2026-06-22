---
id: COMP-grid
name: Grid
kind: gui
layer: layout
module: MOD-ui
status: reviewed
depends_on: []
requirements: []
source: [src/ui/layout/Grid]
variants: [default]
---

# Purpose

A two-dimensional grid layout helper. It arranges its children into columns (and,
optionally, named template areas) with a consistent gap. It is the default tool
for column-based layouts (card galleries, dashboards, form grids) so screens
never hand-roll grid CSS. It is a layout primitive holding an arbitrary
`children` slot.

# Public interface

- **Inputs:** the `children` slot plus `columns`, `gap`, `areas` props.
- **Outputs:** its children placed into a grid of the requested column count or
  named areas, with uniform spacing.
- **Errors:** none — pure layout. An empty `children` slot renders nothing.

# Invariants & rules

- Lays out in **two** dimensions; `columns` sets the track count (or a track
  template token) and rows flow automatically.
- The same `gap` token is applied uniformly between rows and columns.
- When `areas` is provided, children are placed by named area; `areas` takes
  precedence over `columns` for placement, and `columns` (if also given) only
  hints the track sizing.
- It carries no semantics of its own — a generic grouping container.

## Wireframe

```
columns=3, gap="md"                         areas (named template):
┌──────────┬──────────┬──────────┐          ┌─────────────┬──────────┐
│ child 1  │ child 2  │ child 3  │          │   header    │  header  │
├──────────┼──────────┼──────────┤          ├─────────────┼──────────┤
│ child 4  │ child 5  │ child 6  │          │   sidebar   │  content │
└──────────┴──────────┴──────────┘          └─────────────┴──────────┘
                                             areas: "header header" /
                                                    "sidebar content"
```

## Component tree

```
COMP-grid   props: { columns, gap, areas }
└─ [slot: children]  → one or more caller-supplied elements/components
```

## State

| Name | Type | Initial | Description |
|------|------|---------|-------------|
| — | — | — | Stateless; children are supplied via the `children` slot. |

## Props

| Prop       | Type                                   | Required | Default | Description                                                                 |
|------------|----------------------------------------|----------|---------|-----------------------------------------------------------------------------|
| `children` | Slot                                   | no       | —       | The elements to place in the grid.                                          |
| `columns`  | Int \| `auto-fit \| auto-fill`         | no       | `12`    | Number of column tracks, or a responsive auto track keyword.                |
| `gap`      | `none \| xs \| sm \| md \| lg \| xl`   | no       | `md`    | Spacing token applied between rows and columns.                             |
| `areas`    | List<String>                           | no       | —       | Named grid-area template rows (e.g. `["header header", "sidebar content"]`).|

## Variants

- `default` — the single configurable grid. Column-based vs. area-based layout is
  selected by the `columns`/`areas` props, not by separate variants.

## Visual states

- `default` — children placed into the column/area template with the chosen gap.

(All track and gap values are token **names** or counts; concrete sizing resolves
from `.sdd/target.md`.)

## Slots / children

- `children` — the placed elements. When `areas` is used, a child declares the
  named area it occupies (passed through by the caller).

## Events

| Event | Payload | When |
|-------|---------|------|
| — | — | The grid emits no events; it is a passive container. |

## Accessibility

- Purely presentational: it adds no role or landmark and preserves the DOM order
  of its children, so reading and focus order match source order regardless of
  visual placement.
- When visual placement diverges from source order (via `areas`), the source
  order is kept sensible so the reading order remains coherent.

# Acceptance criteria

- **AC1** — Given `columns` is 3 and six children, When the grid renders, Then the
  children fill two rows of three columns with a uniform gap.
- **AC2** — Given `columns` is `auto-fit` and a narrow viewport, When the grid
  renders, Then the column count reduces to fit while preserving the gap.
- **AC3** — Given an `areas` template, When the grid renders, Then each child is
  placed into its named area as described by the template rows.
- **AC4** — Given the grid renders, When assistive technology reads the page, Then
  the children appear in source order with no added roles, even when visually
  rearranged by `areas`.
