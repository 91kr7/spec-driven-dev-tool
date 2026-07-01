<!--
  TEMPLATE — ENTITY spec (kind: entity, DECLARATIVE — no SCoT, no UI). Authority: conventions [§3](../conventions.md#s3)/[§4](../conventions.md#s4).
  Copy to .sdd/specs/<MOD>/model/<ENT-id>.spec.md. Entity = WHAT data it holds + the rules binding it;
  never HOW it is stored. Both the code entity AND the DB schema change DERIVE from this spec.
  Markdown = source of truth; reuse over repetition (DRY). Delete "## Filled example" before saving.
-->
---
id: ENT-<kebab>                # required — matches filename + a <MOD>.index.md row
name: <EntityName>            # required
kind: entity                  # required — declarative
module: MOD-<kebab>           # required — home module
depends_on: [ENT-<other>]     # only entity ids referenced by Relations; [] if none
requirements: [REQ-<nnn>]     # back-link
source: [<path/to/Entity.ext>] # authoritative spec→source map
owns_sections: []
---

# Purpose
<1–2 sentences: WHAT this entity models in the domain + why. No storage/behavior detail.>

# Public interface
- **Identity:** <field(s) forming the primary key, e.g. `id: UUID`>.
- **Inputs (to construct):** <required fields a caller supplies>.
- **Outputs / exposed:** <readable fields; note derived/read-only>.
- **Errors (validation):** <named failures, each tied to an invariant below>.

# Fields
<Neutral types (scot.md [§2](../scot.md#s2)). State each constraint as a declarative fact (required/unique/format/range/default/derived), not a storage directive.>

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `<field>` | `<NeutralType>` | `<required \| unique \| range \| format \| default \| derived>` | <meaning> |

# Relations
<List associations to other entities by id + cardinality (1, 0..1, 1..*, 0..*). Add each related id to depends_on.>

| Relation | Target (id) | Cardinality | Via field | Rule |
|----------|-------------|-------------|-----------|------|
| `<name>` | `ENT-<other>` | `<1 \| 0..1 \| 1..* \| 0..*>` | `<fkField>` | <e.g. required; cascade/restrict on delete> |

# Invariants & rules
<Truths that always hold; both the schema (NOT NULL/UNIQUE/CHECK/FK) and code validation derive from these. Make each testable; give it a short tag.>
- **INV-<tag>:** <e.g. `email` is unique.>
- **INV-<tag>:** <e.g. `total` = sum of line subtotals, never negative.>

# Derivation note
Single source for two derived artifacts, never hand-authored independently (items 1–2 below):
1. **Code entity** at `<source>` — by `code-implementer`; types map via `target.md`; invariants become construction/mutation validation.
2. **DB schema change** — owned by **MOD-schema**, derived here (NOT NULL/UNIQUE/CHECK/FK from the tables above). If the entity changes, correct the spec first, then the schema follows.

# Acceptance criteria
<Each `ACn` as Given/When/Then, tied to a constraint or INV-<tag>. Cover every invariant.>
- **AC1** — Given <a valid instance>, When constructed/validated, Then it is accepted and INV-<tag> holds.
- **AC2** — Given <an instance violating INV-<tag>>, When validated, Then it fails with `<NamedError>`.

---

## Filled example — `ENT-order`

```yaml
id: ENT-order · name: Order · kind: entity · module: MOD-domain
depends_on: [ENT-user] · requirements: [REQ-014] · source: [src/domain/order/Order.ts]
```

**Purpose** — A customer's confirmed purchase: who placed it, the line items, the total, and its fulfilment status. Aggregate root for what is billed and shipped as one transaction.

**Fields**

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | `UUID` | required, unique | Primary key |
| `customerId` | `UUID` | required, format=UUID | FK to the owning user |
| `lines` | `List<OrderLine>` | required, size>=1 | Purchased items |
| `total` | `Decimal` | derived, >=0.00, scale=2 | Sum of line subtotals |
| `status` | `OrderStatus` | required, default=`PENDING` | PENDING/PAID/SHIPPED/CANCELLED |

**Relations** — | `placedBy` | `ENT-user` | `1` (many→one) | `customerId` | required; restrict delete |

**Invariants** — INV-nonEmpty (`lines` size>=1); INV-totalDerived (`total` = Σ qty*unitPrice, >=0, read-only); INV-positiveQty (each line qty>=1); INV-statusEnum.

**Derivation** — code entity at `src/domain/order/Order.ts`; schema owned by MOD-schema (NOT NULL from required, UNIQUE from `id`, FK on `customerId` ON DELETE RESTRICT, CHECKs from INV-positiveQty/totalDerived/statusEnum).

- **AC1** — Given `customerId`, `currency="USD"`, one line `{qty:2, unitPrice:10.00}`, When constructed, Then accepted, `total`=`20.00`, `status`=`PENDING`.
- **AC2** — Given empty `lines`, When constructed, Then `EmptyOrder` (INV-nonEmpty).
- **AC3** — Given a line `qty:0`, When constructed, Then `NegativeQuantity` (INV-positiveQty).
