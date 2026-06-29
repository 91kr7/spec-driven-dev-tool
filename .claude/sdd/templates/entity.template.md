<!--
<instructions>
TEMPLATE: ENTITY spec (kind: entity, DECLARATIVE — no SCoT, no UI). Authority: conventions §3/§4.
Copy to `.sdd/specs/<MOD>/model/ENT-<kebab>.spec.md`.
Describes WHAT data it holds + rules, never HOW it is stored. Both code entity AND DB schema change DERIVE from this.
DRY. Delete the `<example>` block before saving.
</instructions>
-->
---
id: ENT-<kebab>                # required
name: <EntityName>             # required
kind: entity                   # required — declarative
module: MOD-<kebab>            # required
depends_on: [ENT-<other>]      # referenced by Relations; [] if none
requirements: [REQ-<nnn>]
source: [<path/to/Entity.ext>] # authoritative spec→source map
owns_sections: []
---

# Purpose
<instruction>WHAT this entity models in the domain and why. No storage/behavior detail.</instruction>

# Public interface
<instruction>Identity (PK), Inputs (to construct), Outputs/exposed, Errors (validation).</instruction>

# Fields
<instruction>Neutral types (scot.md §2). Constraints are declarative facts, not storage directives.</instruction>
| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `<field>` | `<NeutralType>` | `<required \| unique \| range \| format \| default \| derived>` | <meaning> |

# Relations
<instruction>Associations to other entities by id + cardinality (1, 0..1, 1..*, 0..*); each related id also in depends_on.</instruction>
| Relation | Target (id) | Cardinality | Via field | Rule |
|----------|-------------|-------------|-----------|------|
| `<name>` | `ENT-<other>` | `<1 \| 0..1 \| 1..* \| 0..*>` | `<fkField>` | <e.g. required; cascade/restrict on delete> |

# Invariants & rules
<instruction>Truths that always hold; schema (NOT NULL/UNIQUE/CHECK/FK) and code validation derive from these. Testable.</instruction>
- **INV-<tag>:** <e.g. email is unique>

# Derivation note
<instruction>Spec is single source for: 1. Code entity (`code-implementer`); 2. DB schema change (`MOD-schema`). If entity changes, spec is corrected first.</instruction>

# Acceptance criteria
<instruction>Each ACn Given/When/Then, tied to a constraint or INV-<tag>; cover every invariant.</instruction>
- **AC1** — Given <a valid instance>, When constructed/validated, Then it is accepted and INV-<tag> holds.
- **AC2** — Given <an instance violating INV-<tag>>, When validated, Then it fails with `<NamedError>`.

---
<example name="ENT-order">
```yaml
id: ENT-order
name: Order
kind: entity
module: MOD-domain
depends_on: [ENT-user]
requirements: [REQ-014]
source: [src/domain/order/Order.ts]
```

**Purpose**
A customer's confirmed purchase: who placed it, the line items, the total, and its fulfilment status. Aggregate root.

**Fields**
| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | `UUID` | required, unique | Primary key |
| `customerId` | `UUID` | required, format=UUID | FK to the owning user |
| `lines` | `List<OrderLine>` | required, size>=1 | Purchased items |
| `total` | `Decimal` | derived, >=0.00, scale=2 | Sum of line subtotals |
| `status` | `OrderStatus` | required, default=`PENDING` | PENDING/PAID/SHIPPED/CANCELLED |

**Relations**
| Relation | Target (id) | Cardinality | Via field | Rule |
|----------|-------------|-------------|-----------|------|
| `placedBy` | `ENT-user` | `1` | `customerId` | required; restrict delete |

**Invariants**
- INV-nonEmpty (`lines` size>=1)
- INV-totalDerived (`total` = Σ qty*unitPrice, >=0, read-only)
- INV-positiveQty (each line qty>=1)
- INV-statusEnum

**Derivation**
Code entity at `src/domain/order/Order.ts`; schema owned by MOD-schema (NOT NULL from required, UNIQUE from `id`, FK on `customerId` ON DELETE RESTRICT, CHECKs from INV-positiveQty/totalDerived/statusEnum).

- **AC1** — Given `customerId`, `currency="USD"`, one line `{qty:2, unitPrice:10.00}`, When constructed, Then accepted, `total`=`20.00`, `status`=`PENDING`.
- **AC2** — Given empty `lines`, When constructed, Then `EmptyOrder` (INV-nonEmpty).
- **AC3** — Given a line `qty:0`, When constructed, Then `NegativeQuantity` (INV-positiveQty).
</example>
