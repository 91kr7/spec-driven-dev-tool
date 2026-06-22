---
id: COMP-footer
name: Footer
kind: gui
layer: organism
module: MOD-ui
status: reviewed
depends_on: []
requirements: []
source: [src/ui/organism/Footer]
variants: [default]
---

# Purpose

The application bottom bar. It hosts a leading region (typically copyright /
legal links) and a trailing region (secondary links, status, build version). It
is the canonical contentinfo region used at the bottom of `COMP-appShell`, so
screens never re-describe the page footer. Content is supplied via slots; the
`version` prop offers a convenient way to surface the app build/version on the
right.

# Public interface

- **Inputs:** the `left` and `right` slots plus the `version` prop.
- **Outputs:** a horizontal bar with leading content at the start and trailing
  content (including `version`) at the end.
- **Errors:** none. Omitted regions render empty.

# Invariants & rules

- Exactly one contentinfo per page; `COMP-appShell` places it in its `footer`
  slot.
- Layout order is leading (`left`) → trailing (`right`), left-to-right (mirrored
  in RTL).
- Natural (content-driven) height; it does not scroll and is pinned to the bottom
  by the shell.
- When `version` is provided and no explicit `right` content overrides it, the
  version label is rendered in the trailing region.

## Wireframe

```
┌──────────────────────────── Footer (COMP-footer) ───────────────────────────┐
│  {left: © {year} Brand · Terms · Privacy}                  {right} v{version} │
└──────────────────────────────────────────────────────────────────────────────┘
```

## Component tree

```
COMP-footer
├─ [slot: left]   → leading content (copyright / legal links)
└─ [slot: right]  → trailing content; defaults to the `version` label
```

## State

| Name | Type | Initial | Description |
|------|------|---------|-------------|
| — | — | — | Stateless; content is supplied via props/slots. |

## Props

| Prop      | Type            | Required | Default | Description                                                       |
|-----------|-----------------|----------|---------|-------------------------------------------------------------------|
| `left`    | Slot \| String  | no       | —       | Leading region content (copyright / legal).                       |
| `right`   | Slot            | no       | —       | Trailing region content; overrides the default `version` display. |
| `version` | String          | no       | —       | App build/version string shown in the trailing region by default. |

## Variants

- `default` — two-region bar with leading and trailing content. Additional
  archetypes are promoted only when a real screen requires them.

## Visual states

- `default` — both regions visible and laid out horizontally.
- `focus` — a focused interactive child (e.g. a legal link) shows a focus
  indicator (delegated to the child control).

(Structure only; spacing and color resolve from design tokens at implementation
time.)

## Slots / children

- `left` — leading region.
- `right` — trailing region; falls back to the `version` label when empty.

## Events

| Event | Payload | When |
|-------|---------|------|
| — | — | The footer emits no events; slotted controls emit their own. |

## Accessibility

- Renders as the page `contentinfo` landmark; exactly one per page.
- Interactive children (links) are reachable by Tab in left → right order and
  activate via Enter (behavior owned by the child controls).
- The `version` label is plain readable text, exposed to assistive tech as
  content (not an interactive element).

# Acceptance criteria

- **AC1** — Given `left` content and a `version` value, When the footer renders,
  Then the leading content appears at the start and the version label at the end
  of a single horizontal bar.
- **AC2** — Given both `right` content and a `version` value, When the footer
  renders, Then the `right` slot content is shown in the trailing region and the
  default version label is not duplicated.
- **AC3** — Given no `right` and no `version`, When the footer renders, Then the
  trailing region is empty and the leading content is unaffected, with no error.
- **AC4** — Given assistive technology inspects the page, When it lists
  landmarks, Then the footer is exposed as the single contentinfo region.
