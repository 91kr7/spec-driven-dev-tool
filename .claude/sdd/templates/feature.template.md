<!--
<instructions>
TEMPLATE: FEATURE / USE-CASE spec (kind: use-case, behavioral orchestration).
Copy to `.sdd/specs/<MOD>/features/FEAT-<nnn>.spec.md`. Authority: conventions §2/§3/§5; body = SCoT (cross-class CALLs only).
DRY: name collaborators by id, never re-describe them. Delete the `<example>` block before saving.
</instructions>
-->
---
id: FEAT-<nnn>                  # required
name: <FeatureName>             # required
kind: use-case                  # required — behavioral orchestration
module: MOD-<kebab>             # required
depends_on: [CLS-<id>, ENT-<id>] # every collaborator id, topological
requirements: [REQ-<nnn>]
source: []                      # [] if purely compositional; else one coordinator file
error_style: <result|raise>     # behavioral only
---

# Purpose
<instruction>The end-to-end scenario (actor, goal, outcome). WHAT happens across collaborators, never HOW.</instruction>

# Public interface
<instruction>Inputs (Cmd/DTO), Outputs (Result type), Errors (ErrorCase - list all regardless of error_style).</instruction>

# Collaborators
<instruction>Every participant by id (also in depends_on); never re-describe its internals.</instruction>
| Id | Kind | Role in this flow |
|----|------|-------------------|
| `CLS-<id>` | service/controller | <what it does here> |
| `ENT-<id>` | entity | <object created/loaded/mutated> |

# Invariants & rules
<instruction>Cross-class rules for the whole orchestration: ordering, transactionality, idempotency, on-success/failure guarantees.</instruction>
- <rule>

# Orchestration SCoT
<instruction>One FUNCTION (entry point), cross-class CALLs by id only. scot.md grammar; stable [Bn] with named arms; no library/syntax/internals.</instruction>
```
FUNCTION <entryPoint>(<param>: <Type>) -> <ReturnType>
  PRECONDITION: <holds on entry>
  <step> <- CALL CLS-<id>.<method>(<args>)
  [B1] IF <cross-class condition> THEN
    RETURN <Err(...)>                  # B1.then
  ELSE
    # B1.else — continue
  END
  <result> <- CALL ENT-<id>.<factory>(<args>)
  RETURN <Ok(result)>
END
```

# Integration acceptance criteria
<instruction>End-to-end Given/When/Then, stable ACn, observable cross-class outcomes; cover every orchestration arm. (GUI screens own journey ACs).</instruction>
- **AC1** — *Given* <state>, *When* `<entryPoint>` is invoked with `<input>`, *Then* <observable outcome + persisted/emitted effects>.
- **AC2** — *Given* <branch-triggering state>, *When* invoked, *Then* <alternate outcome, error, no side effects>.

---
<example name="FEAT-010 Checkout">
```yaml
id: FEAT-010
name: Checkout
kind: use-case
module: MOD-shop
depends_on: [CLS-cartService, ENT-order, CLS-paymentGateway, CLS-orderRepo]
requirements: [REQ-014]
source: [src/features/checkoutFlow.ts]
error_style: result
```

**Purpose**
A shopper confirms a purchase: validate cart, capture payment, and only if authorized create order and clear cart.

**Invariants**
- No order persisted unless payment authorized
- Cart cleared only after order saved
- Checkout is one logical transaction (zero side effects on failure)

**Orchestration SCoT**
```
FUNCTION checkout(cmd: CheckoutCmd) -> Result<Order, CheckoutError>
  PRECONDITION: cmd is DTO-valid
  cart <- CALL CLS-cartService.load(cmd.cartId)
  [B1] IF cart.isEmpty THEN RETURN Err(EmptyCart)   # B1.then
  ELSE END                                          # B1.else
  [B2] IF NOT CALL CLS-cartService.allInStock(cart) THEN RETURN Err(OutOfStock) END  # B2.then / B2.else
  payment <- CALL CLS-paymentGateway.capture(cmd.paymentToken, cart.total)
  [B3] IF NOT payment.authorized THEN RETURN Err(PaymentDeclined)   # B3.then
  ELSE END                                                          # B3.else
  order <- CALL ENT-order.fromCart(cart, payment, cmd.shippingAddress)
  saved <- CALL CLS-orderRepo.save(order)
  CALL CLS-cartService.clear(cmd.cartId)
  EMIT OrderConfirmed({ orderId: saved.id })
  RETURN Ok(saved)
END
```
Coverage set: `B1.then/else`, `B2.then/else`, `B3.then/else`.

- **AC1** — *Given* a cart with in-stock items + valid token, *When* checkout, *Then* payment captured, order persisted, cart cleared, Ok(Order).
- **AC2** — *Given* a declined token, *When* checkout, *Then* Err(PaymentDeclined), no order persisted, cart unchanged.
</example>
