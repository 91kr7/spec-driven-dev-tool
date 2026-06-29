<!--
  TEMPLATE — class/service/controller spec (behavioral, SCoT). Copy to .sdd/specs/classes/CLS-<lowerCamel>.spec.md.
  Authority: conventions §3 + scot.md (the ONLY behavioral grammar — branch ids [Bn], arm enumeration, §6 error style).
  Discover before create: read classes.index.md first; reuse SHR-*/CLS-* by id.
  Markdown is the source of truth; reuse over repetition (DRY). Delete the "## Filled example".
-->
---
id: CLS-<lowerCamel>          # required — matches filename
name: <ClassName>            # required
kind: <service|controller>   # required — behavioral → SCoT body
module: MOD-<kebab>          # required — home module
depends_on: [<CLS-*|SHR-*|ENT-* ids, deps first>]
requirements: [REQ-<nnn>]    # back-link
source: [<src/… from target.md conventions>]
owns_sections: []
error_style: <result|raise>  # behavioral only (scot.md §6) — canonical home for the error style
---

# Purpose
<One paragraph: WHAT this class is responsible for and WHY. Responsibility/behavior, never implementation/library/framework.>

# Public interface
<Every public method; neutral types (scot.md §2); errors listed regardless of error_style.>

| Method | Inputs | Output | Errors |
|--------|--------|--------|--------|
| `<method>(<p>: <Type>)` | `<p>` — <meaning> | `<ReturnType>` | `<ErrorCase>` — <when> |

# Invariants & rules
<Properties that always hold + business rules enforced; reference deps by id.>
- <Invariant / rule, traceable to a requirement>

# SCoT
<One FUNCTION per public method. Grammar = scot.md. Every branch a stable [Bn] with ALL arms enumerated. No library/syntax/SQL — those go to impl-notes.>

```
FUNCTION <method>(<param>: <Type>) -> <ReturnType>
  PRECONDITION: <holds on entry>      # optional
  [B1] IF <condition> THEN
    <statement>                       # B1.then
  ELSE
    <statement>                       # B1.else
  END
  <result> <- CALL <SHR-*|CLS-*|ENT-* id>.<method>(<args>)
  RETURN <value or Ok(...)/Err(...)>
END
```

# Acceptance criteria
<Each `ACn` Given/When/Then; cover the happy path AND every branch arm (≥1 test per ACn + per arm — scot.md §7).>
- **AC1** — Given <state>, When <call>, Then <observable outcome>.
- **AC2** — Given <state>, When <branch-triggering call>, Then <expected error/result>.

---

## Filled example — `CLS-cartService.addItem`

```yaml
id: CLS-cartService · name: CartService · kind: service · module: MOD-checkout
depends_on: [CLS-cartRepo, ENT-cart, ENT-product] · requirements: [REQ-014]
source: [src/checkout/CartService.ts] · error_style: result
```

**Public interface** — | `addItem(cartId: UUID, productId: UUID, qty: Int)` | ids + units(≥1) | `Result<Cart, CartError>` | `CartNotFound`, `InvalidQuantity`, `InsufficientStock` |

**Invariants** — no line with qty<1; adding an existing product increments its line; resulting qty must not exceed `ENT-product.stock` (else reject, cart unchanged); save only after all rules pass.

```
FUNCTION addItem(cartId: UUID, productId: UUID, qty: Int) -> Result<Cart, CartError>
  PRECONDITION: cartId, productId are well-formed UUIDs
  [B1] IF qty < 1 THEN RETURN Err(InvalidQuantity)   # B1.then
  ELSE END                                           # B1.else
  cart <- CALL CLS-cartRepo.findById(cartId)
  [B2] IF cart IS null THEN RETURN Err(CartNotFound) # B2.then
  ELSE END                                           # B2.else
  existing <- CALL cart.lineFor(productId)
  desired  <- ((existing IS null) ? 0 : existing.quantity) + qty
  [B3] IF desired > CALL ENT-product.stockOf(productId) THEN RETURN Err(InsufficientStock)  # B3.then
  ELSE END                                                                                  # B3.else
  CALL cart.setQuantity(productId, desired)
  saved <- CALL CLS-cartRepo.save(cart)
  RETURN Ok(saved)
END
```
Coverage set: `B1.then/else`, `B2.then/else`, `B3.then/else` + the ACs.

- **AC1** — Given an empty cart + product stock 10, When `addItem(cartId, productId, 3)`, Then `Ok(Cart)` with one line qty 3. *(B1.else, B2.else, B3.else)*
- **AC2** — Given `qty=0`, When `addItem(...)`, Then `Err(InvalidQuantity)`, cart unchanged. *(B1.then)*
