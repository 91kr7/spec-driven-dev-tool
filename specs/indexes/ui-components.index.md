<!--
  INDEX — the default shared UI component library (specs/ui-components/).
  One row per component. Indexes carry NO YAML front-matter (see specs/templates/index.template.md).
  Authority: .sdd/conventions.md (§4 index row schema — ui-components adds `layer` + `variants`),
             .sdd/ui-schema.md (§7 atomic-design layers).
  Markdown is the source of truth (authority); reuse over repetition (DRY).
-->

# UI Components Index

The shared UI component library. Each row is a `kind: gui` spec authored once per
`.sdd/ui-schema.md` and referenced **by id** everywhere it is used. This index is
the **discovery surface**: agents and screens read it first and open only the
component specs they need.

Cross-cutting values that govern this library:
**Markdown is the source of truth (authority); reuse over repetition (DRY).**

## How to use this library

- **Discover before you create.** Before specifying any widget on a screen, scan
  this index. If a component exists, reference it by id and pass props/slots —
  never re-describe a button, input, panel, header, or layout that already lives
  here. A feature screen (`specs/classes/CLS-*.spec.md`, `kind: gui`) composes
  these components by id and only adds screen-specific layout and behavior.
- **Progressive enrichment.** This is the **default scaffold** — the reuse-first
  frontend that is in place from the very start: layout primitives plus the panel
  container. The library grows as real patterns emerge. The reuse-analyst
  **promotes** a recurring widget into the library the moment a second screen
  needs it, rather than letting it be duplicated. Components are **added** as they
  prove out, for example:
  - **atoms** — `Button`, `TextInput`, `Icon`, `Badge`, `Spinner`, `Checkbox`…
  - **molecules** — `FormField` (label + control + error), `SearchBar`,
    `Pagination`…
  - **organisms** — `Table`, `Modal`, `Menu`, `Toast`… (joining `Header`,
    `Footer`, `Panel`).
- **Atomic-design layering.** Every component declares a `layer`:
  **atom → molecule → organism → layout**. Higher layers compose lower layers
  **by id**; a molecule never re-describes its atoms, an organism never
  re-describes its molecules, and a layout never re-describes the organisms it
  arranges. The app-shell template and the spacing/grid helpers live in the
  `layout` layer.

## Components

| id | name | description | module | layer | variants | depends_on | spec | source | status |
|----|------|-------------|--------|-------|----------|------------|------|--------|--------|
| COMP-appShell | AppShell | Top-level app frame: header on top, body filling the middle, footer at the bottom | MOD-ui | layout | default | COMP-header, COMP-body, COMP-footer | specs/ui-components/COMP-appShell.spec.md | src/ui/layout/AppShell | reviewed |
| COMP-header | Header | Application top bar with brand, navigation, and trailing actions/user region | MOD-ui | organism | default | — | specs/ui-components/COMP-header.spec.md | src/ui/organism/Header | reviewed |
| COMP-body | Body | Scrollable main content region between header and footer, with max-width and padding | MOD-ui | layout | default | — | specs/ui-components/COMP-body.spec.md | src/ui/layout/Body | reviewed |
| COMP-footer | Footer | Application bottom bar with leading copyright/links and trailing region/version | MOD-ui | organism | default | — | specs/ui-components/COMP-footer.spec.md | src/ui/organism/Footer | reviewed |
| COMP-panel | Panel | Card/panel container with header, body, and footer slots; optionally collapsible | MOD-ui | organism | default, outlined, elevated | — | specs/ui-components/COMP-panel.spec.md | src/ui/organism/Panel | reviewed |
| COMP-stack | Stack | One-dimensional flex layout helper (row/column) with gap, align, justify, wrap | MOD-ui | layout | default | — | specs/ui-components/COMP-stack.spec.md | src/ui/layout/Stack | reviewed |
| COMP-grid | Grid | Two-dimensional grid layout helper with columns, gap, and named template areas | MOD-ui | layout | default | — | specs/ui-components/COMP-grid.spec.md | src/ui/layout/Grid | reviewed |
| COMP-section | Section | Titled content-section wrapper with heading, optional description, and a content slot | MOD-ui | layout | default | — | specs/ui-components/COMP-section.spec.md | src/ui/layout/Section | reviewed |

## Notes

- `source` is **derived** from each spec's `source:` front-matter (filled by a
  command, never hand-edited). New components propose paths from the generic
  `src/ui/<layer>/<Name>` convention until they are implemented against real
  files.
- `depends_on` lists composition by id only. `COMP-appShell` is the only scaffold
  component that composes others (`COMP-header`, `COMP-body`, `COMP-footer`); the
  remaining layout/organism primitives stand alone and are composed **into**
  screens, not into each other.
- `status` here is the canonical lifecycle home for each component
  (`draft → reviewed → approved`); a component spec's front-matter mirrors it.
