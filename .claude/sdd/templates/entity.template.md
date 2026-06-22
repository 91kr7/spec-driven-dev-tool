<!--
  TEMPLATE — .claude/sdd/templates/entity.template.md
  Copy this file to specs/model/<ENT-id>.spec.md and fill every <placeholder>.
  KIND: entity (DECLARATIVE — NO SCoT, NO UI schematic). See .claude/sdd/conventions.md §3, §4.

  Governing values (apply to every entity spec):
    Markdown is the source of truth (authority); reuse over repetition (DRY).

  Rules to obey while filling this in:
    - id MUST be ENT-<kebab> and MUST match the filename and a row in
      specs/indexes/model.index.md (conventions §2, §5).
    - An entity is STRUCTURAL: describe WHAT data it holds and the rules that bind
      it — never HOW it is stored or how schema changes run. The DB schema change and the
      code entity both DERIVE from this spec (see "Derivation note").
    - status MIRRORS the index row; the index is canonical (conventions §5).
    - depends_on lists ONLY other entity ids this one references (relations).
    - source: PROPOSES paths from .sdd/target.md conventions for a NEW entity, or
      points at the real files for an EXISTING one. The index source column is
      DERIVED from this — never hand-edit the index source column (conventions §3).
    - Reuse over repetition: if a field group/value set recurs across entities,
      promote it (the reuse-analyst) rather than copying it here.
    - No "TODO" and no leftover <placeholder> in a real spec — placeholders are
      allowed ONLY in this template file.
-->

---
id: ENT-<kebab>                       # required — ENT-<kebab>, matches filename + model.index row
name: <HumanEntityName>              # required — human-readable name, e.g. Order
kind: entity                         # required — fixed value for this template (declarative)
module: MOD-<kebab>                  # required — the level's home module, e.g. MOD-domain
status: draft                        # draft|reviewed|approved — MIRRORS the index (index is canonical)
depends_on: [ENT-<other>, ...]       # ids of entities referenced by Relations (topological order); [] if none
requirements: [REQ-<nnn>, ...]       # requirement id(s) this entity serves (back-link)
source: [<path/to/entity-file>]      # AUTHORITATIVE spec→source mapping; proposed for new, real for existing
owns_sections: []                    # only for a co-owned aggregator file; otherwise []
---

# Purpose

<!-- One or two sentences: WHAT this entity represents in the domain and why it
     exists. No persistence/storage detail, no behavior. -->
<Single-paragraph statement of what this entity models in the domain.>

# Public interface

<!-- For an entity the "interface" is its construction contract and how callers
     obtain/identify it. Keep it declarative: inputs to create a valid instance,
     the identity it exposes, and the validation errors construction can yield. -->

- **Identity:** <which field(s) form the stable identity / primary key, e.g. `id: UUID`>.
- **Inputs (to construct a valid instance):** <list the required fields a caller
  must supply, e.g. `customerId`, `lines`, `currency`>.
- **Outputs / exposed:** <fields readable by callers, e.g. all fields below; any
  derived/read-only field is noted in the table>.
- **Errors (validation):** <named validation failures construction/mutation can
  produce, each tied to an invariant below, e.g. `InvalidTotal`, `EmptyOrder`>.

# Fields

<!-- The data shape. Use NEUTRAL type names from .claude/sdd/scot.md §2 (Int, Long,
     Decimal, Bool, String, Date, DateTime, UUID, List<T>, Map<K,V>, Option<T>,
     enum/entity by id or name). Constraints are declarative facts (required,
     unique, format, range, default, derived) — NOT storage directives. -->

| Field           | Type            | Constraints                                   | Description                                  |
|-----------------|-----------------|-----------------------------------------------|----------------------------------------------|
| `<fieldName>`   | `<NeutralType>` | `<required \| unique \| range \| format \| default \| derived>` | <what this field means>      |
| `<fieldName>`   | `<NeutralType>` | `<...>`                                        | <...>                                        |

# Relations

<!-- Associations to OTHER entities, referenced BY ID, with cardinality.
     Cardinality notation: 1, 0..1, 1..*, 0..*. State the foreign key field and
     the delete/ownership rule when relevant. Every related id MUST also appear in
     depends_on. Reuse over repetition: do not redefine the other entity here. -->

| Relation        | Target (id)   | Cardinality (this → target) | Via field        | Rule                                  |
|-----------------|---------------|-----------------------------|------------------|---------------------------------------|
| `<relationName>`| `ENT-<other>` | `<1 \| 0..1 \| 1..* \| 0..*>` | `<fkField>`     | <e.g. required; cascade/restrict on delete> |

# Invariants & rules

<!-- The constraints that MUST always hold for a valid instance. These are the
     truths the DB schema change (CHECK/UNIQUE/NOT NULL/FK) and the code entity
     validation BOTH derive from. Make each one testable; give it a short tag so an
     acceptance criterion can reference it. -->

- **INV-<tag>:** <e.g. `email` is unique across all instances.>
- **INV-<tag>:** <e.g. `total` equals the sum of line subtotals (derived, never negative).>
- **INV-<tag>:** <e.g. `status` is one of the allowed enum values.>
- **INV-<tag>:** <e.g. a non-empty `lines` list is required (>= 1 line).>

# Derivation note

<!-- Fixed, mandatory note: this spec is the single source for two derived
     artifacts. Keep this section; adjust only the file paths/ids in <...>. -->

This entity spec is the **single source of truth** for two derived artifacts; both
are generated from the fields, relations and invariants above and are **never
hand-authored independently** (conventions §1, §2):

1. **Code entity** at `<source path>` — derived by `code-implementer`; field types
   map to the target language via `.sdd/target.md`; invariants become construction
   /mutation validation.
2. **DB schema change** — owned by **MOD-build**, derived from this spec. NOT NULL /
   UNIQUE / CHECK / FK constraints come straight from the Fields, Relations and
   Invariants tables. If this entity changes, the schema change is re-derived; the spec
   is corrected first and the schema change follows (spec wins).

Constraint/validation tests are derived from the invariants by `test-writer`
(conventions §1: `tests/` constraints from entities).

# Acceptance criteria

<!-- Each ACn is a constraint/validation check in Given/When/Then form, tied to a
     field constraint or an INV-<tag> above. Stable ids ACn (conventions §2).
     Cover every invariant and every non-trivial constraint. -->

- **AC1** — Given <a valid instance>, When <constructed/validated>, Then <it is
  accepted and INV-<tag> holds>.
- **AC2** — Given <an instance violating INV-<tag>>, When <constructed/validated>,
  Then <construction fails with `<NamedError>`>.

---

## Filled example

<!-- A complete, real entity spec produced from this template. ENT-order, with a
     relation to ENT-user and concrete invariants. This is what a finished
     specs/model/ENT-order.spec.md looks like (front-matter included). -->

```markdown
---
id: ENT-order
name: Order
kind: entity
module: MOD-domain
status: draft
depends_on: [ENT-user]
requirements: [REQ-014]
source: [src/domain/order/Order.ts]
owns_sections: []
---

# Purpose

An Order represents a customer's confirmed purchase: who placed it, the line items
purchased, the monetary total, and its progression through the fulfilment lifecycle.
It is the aggregate root for everything billed and shipped as one transaction.

# Public interface

- **Identity:** `id: UUID` — server-assigned, stable for the order's lifetime.
- **Inputs (to construct a valid instance):** `customerId`, `currency`, and a
  non-empty `lines` list; `placedAt` defaults to creation time.
- **Outputs / exposed:** all fields below; `total` is read-only (derived).
- **Errors (validation):** `EmptyOrder` (no lines), `InvalidCurrency` (not ISO-4217),
  `NegativeQuantity` (a line quantity < 1), `TotalMismatch` (`total` ≠ sum of lines).

# Fields

| Field          | Type                | Constraints                                  | Description                                         |
|----------------|---------------------|----------------------------------------------|-----------------------------------------------------|
| `id`           | `UUID`              | required, unique                             | Stable order identity (primary key).                |
| `customerId`   | `UUID`              | required, format=UUID                        | FK to the owning user (see Relations).              |
| `lines`        | `List<OrderLine>`   | required, size >= 1                          | Purchased line items (sku, quantity, unitPrice).    |
| `currency`     | `String`            | required, format=ISO-4217 (3 letters)        | Currency of all monetary amounts on the order.      |
| `total`        | `Decimal`           | derived, range >= 0.00, scale=2              | Sum of line subtotals; never negative.              |
| `status`       | `OrderStatus`       | required, default=`PENDING`                  | Lifecycle: PENDING, PAID, SHIPPED, CANCELLED.       |
| `placedAt`     | `DateTime`          | required, default=now                        | When the order was placed (UTC).                    |
| `note`         | `Option<String>`    | optional, maxLength=500                       | Optional customer note.                             |

# Relations

| Relation     | Target (id) | Cardinality (this → target) | Via field    | Rule                                                  |
|--------------|-------------|-----------------------------|--------------|-------------------------------------------------------|
| `placedBy`   | `ENT-user`  | `1` (many orders → one user)| `customerId` | Required; restrict delete — a user with orders cannot be hard-deleted. |

# Invariants & rules

- **INV-nonEmpty:** `lines` always has at least one element (size >= 1).
- **INV-totalDerived:** `total` equals the sum of each line's `quantity * unitPrice`,
  rounded to 2 decimal places; it is read-only and always >= 0.00.
- **INV-positiveQty:** every line's `quantity` is >= 1.
- **INV-currencyFormat:** `currency` is a valid ISO-4217 3-letter code, upper-case.
- **INV-statusEnum:** `status` is one of PENDING, PAID, SHIPPED, CANCELLED.
- **INV-customerExists:** `customerId` references an existing `ENT-user`.

# Derivation note

This entity spec is the **single source of truth** for two derived artifacts; both
are generated from the fields, relations and invariants above and are **never
hand-authored independently** (conventions §1, §2):

1. **Code entity** at `src/domain/order/Order.ts` — derived by `code-implementer`;
   field types map to the target language via `.sdd/target.md`; invariants become
   construction/mutation validation (e.g. `EmptyOrder`, `TotalMismatch`).
2. **DB schema change** — owned by **MOD-build**, derived from this spec. NOT NULL comes
   from required fields, UNIQUE from `id`, the FK on `customerId` from the `placedBy`
   relation (ON DELETE RESTRICT), and CHECK constraints from INV-positiveQty,
   INV-totalDerived (`total >= 0`) and INV-statusEnum. If Order changes, the
   schema change is re-derived; the spec is corrected first and the schema change follows.

Constraint/validation tests are derived from the invariants by `test-writer`.

# Acceptance criteria

- **AC1** — Given a payload with `customerId`, `currency="USD"`, and one line
  `{ sku, quantity: 2, unitPrice: 10.00 }`, When the Order is constructed, Then it is
  accepted, `total` = `20.00` (INV-totalDerived) and `status` = `PENDING`.
- **AC2** — Given a payload with an empty `lines` list, When construction is attempted,
  Then it fails with `EmptyOrder` (INV-nonEmpty).
- **AC3** — Given a line with `quantity: 0`, When construction is attempted, Then it
  fails with `NegativeQuantity` (INV-positiveQty).
- **AC4** — Given `currency="US"`, When construction is attempted, Then it fails with
  `InvalidCurrency` (INV-currencyFormat).
- **AC5** — Given a `customerId` that no `ENT-user` matches, When the Order is
  persisted, Then it is rejected by the `placedBy` FK (INV-customerExists).
```
