<!--
  FROZEN, ILLUSTRATIVE EXAMPLE — NOT part of the tool's real specs/.
  Part of the "User registration (Database + API)" worked example.
  Delete examples/user-registration/ before real use.
  Authority: .sdd/conventions.md (§4 index row schema — ui-components adds layer + variants),
             .sdd/ui-schema.md (§7 atomic-design layers).
  Markdown is the source of truth (authority); reuse over repetition (DRY).
-->

# UI-components index — `COMP-*` (worked example)

> The shared UI component library for this slice. One row per component. This
> index adds a `layer` (atom|molecule|organism|layout, ui-schema §7) and a
> `variants` column to the standard index schema. **Discover before create**: a
> screen (`CLS-registerScreen`) composes these by id and never re-describes them.
>
> The three rows below were **promoted by the reuse-analyst from `FEAT-001`** —
> recurring widgets lifted out of the screen into reusable specs.

| id | name | description | module | layer | variants | depends_on | spec | source | status |
|----|------|-------------|--------|-------|----------|------------|------|--------|--------|
| `COMP-textInput` | TextInput | Single-line text entry atom with controlled value and change event | `MOD-web` | atom | — | — | `specs/ui-components/COMP-textInput.spec.md` | `src/web/ui/TextInput.tsx` | reviewed |
| `COMP-button` | Button | Reusable clickable control with variants and a loading state | `MOD-web` | atom | `primary`, `secondary`, `ghost`, `danger` | — | `specs/ui-components/COMP-button.spec.md` | `src/web/ui/Button.tsx` | reviewed |
| `COMP-formField` | FormField | Molecule pairing a label + control slot + error text for one field | `MOD-web` | molecule | — | `COMP-textInput` | `specs/ui-components/COMP-formField.spec.md` | `src/web/ui/FormField.tsx` | reviewed |

## Notes

- **Layout primitives are NOT redefined here.** `COMP-appShell`, `COMP-header`,
  `COMP-body`, `COMP-footer`, `COMP-panel`, and `COMP-stack` (the `layout` and
  `organism` shell) are provided by the **shared SDD UI library**. They are
  referenced by id from `CLS-registerScreen`'s component tree (ui-schema §3) and
  appear in **that** library's index, not in this example slice — reuse over
  repetition (DRY).
- `COMP-formField` composes `COMP-textInput` (its control slot) **by id**; a
  molecule never re-describes its atoms (ui-schema §7).
- `COMP-button` exposes named `variants`; the screen uses `secondary` for
  **Cancel** and `primary` for **Register** (with `loading` bound to `submitting`).
