<!--
  TEMPLATE — class/service/controller spec (behavioral, SCoT).
  Copy this file to specs/classes/CLS-<lowerCamel>.spec.md and fill every <placeholder>.
  Authority: .claude/sdd/conventions.md (ids, front-matter §3, status, ROLE block, change policy),
             .claude/sdd/scot.md (the ONLY behavioral grammar — branch ids [Bn], arm enumeration, §6 error styles).
  Values that bind this artifact: Markdown is the source of truth (authority); reuse over repetition (DRY).
  Discover before create: read specs/indexes/classes.index.md first and reuse SHR-*/CLS-* by id.
  Placeholders below are intentional. The "## Filled example" section at the bottom is a complete,
  copy-correct sample — keep it as a reference, delete it from a real spec.
-->
---
id: CLS-<lowerCamel>            # required — matches filename; kind drives FORM=SCoT (conventions §3)
name: <ClassName>              # required — human/PascalCase name of the class
kind: <service|controller>    # required — behavioral → SCoT body (conventions §3 table)
module: MOD-<kebab>           # required — the home module of this class
status: draft                 # draft|reviewed|approved — MIRRORS the index row (index is canonical)
depends_on: [<CLS-*|SHR-*|ENT-* ids in topological order, deps first>]
requirements: [REQ-<nnn>]     # requirement id(s) this spec serves (back-link)
source: [<path proposed from .sdd/target.md conventions, e.g. src/...>]   # authoritative spec→source map
owns_sections: []             # only for co-owned aggregator files; else leave []
error_style: <result|raise>   # behavioral specs ONLY — must equal the SCoT `error-style:` below (scot.md §6)
---

# Purpose

<!-- One short paragraph: WHAT this class is responsible for and WHY it exists.
     Describe responsibility/behavior, never implementation, library, or framework. -->
<Single-responsibility statement for this class — what it does, for whom, and the boundary it owns.>

# Public interface

<!-- Every publicly callable method. Inputs/outputs use neutral SCoT type names (scot.md §2).
     The errors column lists error cases REGARDLESS of error_style (scot.md §6). -->

| Method                      | Inputs                         | Output            | Errors                                  |
|-----------------------------|--------------------------------|-------------------|-----------------------------------------|
| `<methodName>(<p>: <Type>)` | `<p>` — <meaning>              | `<ReturnType>`    | `<ErrorCase>` — <when it occurs>        |
| `<otherMethod>(...)`        | <named inputs and meaning>     | `<Type>`          | `<ErrorCase>`, `<ErrorCase2>`           |

# Invariants & rules

<!-- Properties that always hold, plus business rules the methods must enforce.
     Reference dependencies by id (e.g. ENT-user, SHR-passwordHasher). -->
- <Invariant 1 — a condition that is always true before/after every public method.>
- <Rule 1 — a domain/business rule this class enforces, traceable to a requirement.>
- <Rule 2 — preconditions on inputs; what is rejected and how.>

# SCoT

<!-- Behavioral body. Grammar = .claude/sdd/scot.md ONLY. State error-style first (must match front-matter).
     One FUNCTION per public method. Every branching construct carries a stable [Bn] id and ALL its
     arms must be enumerated (IF → Bn.then/Bn.elifk/Bn.else; loops → Bn.body + Bn.empty|skip; etc.).
     No library/framework names, no language syntax, no SQL — those go to .sdd/impl-notes/<id>.md. -->

```
error-style: <result|raise>

FUNCTION <methodName>(<param>: <Type>) -> <ReturnType>
  INPUT:  <param> — <meaning / shape>
  OUTPUT: <what the return value means on success and on failure>
  PRECONDITION: <what must hold on entry>        # optional

  [B1] IF <condition> THEN
    <statement>                                  # arm B1.then
  ELSE
    <statement>                                  # arm B1.else
  END

  <result> <- CALL <SHR-*|CLS-*|ENT-* id>.<method>(<args>)
  RETURN <value or Ok(...)/Err(...) per error-style>
END
```

# Acceptance criteria

<!-- Each ACn in Given/When/Then. Cover the happy path AND every branch arm above.
     Test-writer derives at least one test per ACn and per SCoT arm id (scot.md §7). -->

- **AC1** — Given <precondition/state>, When <action/call>, Then <observable outcome>.
- **AC2** — Given <precondition/state>, When <action that triggers a branch>, Then <expected error/result>.

<!-- ===================================================================== -->
<!-- The blank template ends here. Below is a complete, copy-correct       -->
<!-- example. Delete it when authoring a real spec.                        -->
<!-- ===================================================================== -->

## Filled example

```yaml
---
id: CLS-cartService
name: CartService
kind: service
module: MOD-checkout
status: draft
depends_on: [CLS-cartRepo, ENT-cart, ENT-cartItem, ENT-product, SHR-money]
requirements: [REQ-014]
source: [src/checkout/CartService.ts]
owns_sections: []
error_style: result
---
```

### Purpose

`CartService` owns the shopping-cart use cases for a shopper: adding a product to
a cart, adjusting quantities, and computing the cart total. It enforces the
quantity and stock rules so that no caller can place an invalid line item, and it
keeps the persisted cart consistent with the domain invariants.

### Public interface

| Method                                                 | Inputs                                                                 | Output                  | Errors                                                                 |
|--------------------------------------------------------|------------------------------------------------------------------------|-------------------------|------------------------------------------------------------------------|
| `addItem(cartId: UUID, productId: UUID, qty: Int)`     | `cartId` — target cart; `productId` — product to add; `qty` — units (≥1) | `Result<Cart, CartError>` | `CartNotFound` — no cart for `cartId`; `InvalidQuantity` — `qty < 1`; `InsufficientStock` — `qty` exceeds available stock |

### Invariants & rules

- A cart never contains a line item with `quantity < 1`; removing the last unit
  removes the line item entirely (enforced elsewhere; `addItem` only adds).
- Adding a `productId` already present increments the existing line item's
  quantity rather than creating a duplicate line.
- The requested resulting quantity must not exceed `ENT-product.stock`; otherwise
  the operation is rejected and the cart is left unchanged (`InsufficientStock`).
- `addItem` is atomic with respect to persistence: the cart is saved only after
  all rules pass.

### SCoT

```
error-style: result

FUNCTION addItem(cartId: UUID, productId: UUID, qty: Int) -> Result<Cart, CartError>
  INPUT:  cartId    — id of the cart to mutate
          productId — id of the product to add
          qty       — number of units to add (must be >= 1)
  OUTPUT: Ok(Cart) with the updated cart on success;
          Err(CartError) when a rule is violated, cart unchanged
  PRECONDITION: cartId, productId are well-formed UUIDs

  [B1] IF qty < 1 THEN
    RETURN Err(InvalidQuantity)                    # arm B1.then
  ELSE
    # arm B1.else — quantity is valid, continue
  END

  cart <- CALL CLS-cartRepo.findById(cartId)

  [B2] IF cart IS null THEN
    RETURN Err(CartNotFound)                        # arm B2.then
  ELSE
    # arm B2.else — cart exists, continue
  END

  existing <- CALL cart.lineFor(productId)
  current  <- ( existing IS null ) ? 0 : existing.quantity
  desired  <- current + qty
  stock    <- CALL ENT-product.stockOf(productId)

  [B3] IF desired > stock THEN
    RETURN Err(InsufficientStock)                   # arm B3.then
  ELSE
    # arm B3.else — enough stock, apply the change
  END

  CALL cart.setQuantity(productId, desired)
  saved <- CALL CLS-cartRepo.save(cart)
  EMIT CartItemAdded({ cartId: saved.id, productId: productId, quantity: desired })
  LOG info "cart item added"
  RETURN Ok(saved)
END
```

Coverage set for `addItem`: `B1.then`, `B1.else`, `B2.then`, `B2.else`,
`B3.then`, `B3.else` — plus the acceptance criteria below.

### Acceptance criteria

- **AC1** — Given an existing empty cart and a product with stock 10, When
  `addItem(cartId, productId, 3)` is called, Then the result is `Ok(Cart)` and the
  cart contains one line item for `productId` with quantity 3. *(covers B1.else,
  B2.else, B3.else)*
- **AC2** — Given an existing cart and `qty = 0`, When `addItem(cartId, productId, 0)`
  is called, Then the result is `Err(InvalidQuantity)` and the cart is unchanged.
  *(covers B1.then)*
