<!--
<instructions>
TEMPLATE: class/service/controller spec (behavioral, SCoT). Copy to `.sdd/specs/<MOD>/classes/CLS-<lowerCamel>.spec.md`.
Authority: conventions §3 + scot.md (grammar: [Bn], arm enumeration, §6 error style).
Discover before create: read `<MOD>.index.md` first; reuse SHR-*/CLS-* by id.
DRY. Delete the `<example>` block before saving.
</instructions>
-->
---
id: CLS-<lowerCamel>          # required
name: <ClassName>             # required
kind: <service|controller>    # required
module: MOD-<kebab>           # required
depends_on: [<CLS-*|SHR-*|ENT-*>]
requirements: [REQ-<nnn>]
source: [<src/...>]
owns_sections: []
error_style: <result|raise>   # behavioral only
---

# Purpose
<instruction>WHAT this class is responsible for and WHY. Behavior only, no implementation details.</instruction>

# Public interface
<instruction>Every public method; neutral types (scot.md §2); errors listed regardless of error_style.</instruction>
| Method | Inputs | Output | Errors |
|--------|--------|--------|--------|
| `<method>(<p>: <Type>)` | `<p>` — <meaning> | `<ReturnType>` | `<ErrorCase>` — <when> |

# Invariants & rules
<instruction>Properties that always hold + business rules enforced; reference deps by id.</instruction>
- <rule>

# SCoT
<instruction>One FUNCTION per public method. scot.md grammar. Every branch a stable [Bn] with ALL arms enumerated. No library/syntax/SQL.</instruction>
```
FUNCTION <method>(<param>: <Type>) -> <ReturnType>
  PRECONDITION: <holds on entry>
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
<instruction>Each ACn Given/When/Then; cover happy path AND every branch arm (≥1 test per ACn + per arm).</instruction>
- **AC1** — Given <state>, When <call>, Then <observable outcome>.
- **AC2** — Given <state>, When <branch-triggering call>, Then <expected error/result>.

---
<example name="CLS-cartService.addItem">
```yaml
id: CLS-cartService
name: CartService
kind: service
module: MOD-checkout
depends_on: [CLS-cartRepo, ENT-cart, ENT-product]
requirements: [REQ-014]
source: [src/checkout/CartService.ts]
error_style: result
```

**Public interface**
| Method | Inputs | Output | Errors |
|--------|--------|--------|--------|
| `addItem(cartId: UUID, productId: UUID, qty: Int)` | ids + units(≥1) | `Result<Cart, CartError>` | `CartNotFound`, `InvalidQuantity`, `InsufficientStock` |

**Invariants**
- No line with qty<1
- Adding existing product increments its line
- Resulting qty must not exceed `ENT-product.stock` (else reject)
- Save only after all rules pass

**SCoT**
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
</example>
