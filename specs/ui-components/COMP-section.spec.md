---
id: COMP-section
name: Section
kind: gui
layer: layout
module: MOD-ui
status: reviewed
depends_on: []
requirements: []
source: [src/ui/layout/Section]
variants: [default]
---

# Purpose

A titled content-section wrapper. It groups a block of content under a heading
and an optional description, providing consistent vertical rhythm between the
sections of a screen. It is the default way to introduce a labeled region inside
the page body (e.g. "Account", "Notifications") without re-describing heading and
spacing each time. It is a layout primitive holding a `children` slot.

# Public interface

- **Inputs:** the `children` slot plus `title` and `description` props.
- **Outputs:** a heading (and optional description) followed by the content block,
  with consistent spacing between sections.
- **Errors:** none — pure layout. An empty `children` slot renders just the
  heading area.

# Invariants & rules

- The heading renders when `title` is present; the description renders only when
  both present and non-empty.
- The section emits a single semantic heading for its `title` so the page outline
  stays correct.
- Consistent spacing is applied below the heading/description and between
  adjacent sections; it owns no other visual styling.
- It carries content grouping semantics only — no business behavior.

## Wireframe

```
┌──────────────────────── Section (COMP-section) ────────────────────────────┐
│  {title}                                                                    │  ← heading
│  {description}                                                              │  ← optional sub-text
│                                                                             │
│   {children: content block}                                                 │  ← content
│                                                                             │
└──────────────────────────────────────────────────────────────────────────────┘
```

## Component tree

```
COMP-section   props: { title, description }
└─ [slot: children]  → caller content block (forms, panels, lists)
```

## State

| Name | Type | Initial | Description |
|------|------|---------|-------------|
| — | — | — | Stateless; content is supplied via the `children` slot. |

## Props

| Prop          | Type   | Required | Default | Description                                          |
|---------------|--------|----------|---------|------------------------------------------------------|
| `title`       | String | no       | —       | Section heading text; renders the heading when set.  |
| `description` | String | no       | —       | Optional supporting sub-text below the heading.      |
| `children`    | Slot   | no       | —       | The content block grouped under the heading.         |

## Variants

- `default` — heading + optional description + content block. Heading level and
  emphasis are resolved from design tokens, not via separate variants.

## Visual states

- `default` — heading (and description, if any) shown above the content block
  with consistent spacing.

(Heading sizing and spacing are token **names**; concrete values resolve from
`.sdd/target.md`.)

## Slots / children

- `children` — the section's content block.

## Events

| Event | Payload | When |
|-------|---------|------|
| — | — | The section emits no events; it is a passive container. |

## Accessibility

- Renders a semantic heading for `title`, contributing one level to the document
  outline so screen-reader users can navigate by heading.
- The content block is associated with its heading as a labeled region where the
  platform supports it, so the group is announced by name.
- Reading and focus order follow source order: heading → description → content.

# Acceptance criteria

- **AC1** — Given a `title` and content, When the section renders, Then the title
  appears as a heading above the content block.
- **AC2** — Given a `title` and a `description`, When the section renders, Then
  the description appears as sub-text between the heading and the content.
- **AC3** — Given no `description`, When the section renders, Then only the
  heading and content show, with no empty sub-text gap.
- **AC4** — Given assistive technology navigates by heading, When it reaches the
  section, Then the `title` is exposed as a heading labeling the content block.
