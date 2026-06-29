<!--
  TEMPLATE — FEATURE / USE-CASE spec (kind: use-case, behavioral orchestration).
  Copy to specs/features/<FEAT-nnn>.spec.md. Authority: conventions §2/§3/§5; body = SCoT (scot.md,
  cross-class CALLs only). Markdown is the source of truth; reuse over repetition — name collaborators
  by id, never re-describe them. Delete the "## Filled example" from a real spec.
-->
---
id: FEAT-<nnn>                  # required — matches filename + features.index row
name: <FeatureName>            # required
kind: use-case                 # required — behavioral orchestration
module: MOD-<kebab>            # required — home module
depends_on: [CLS-<id>, ENT-<id>]   # every collaborator id, topological
requirements: [REQ-<nnn>]      # back-link
source: []                     # [] if purely compositional (integration tests only); else one coordinator file
error_style: result            # result|raise (scot.md §6) — canonical home for the error style
---

# Purpose
<One paragraph: the end-to-end scenario (actor, goal, outcome). WHAT happens across collaborators, never HOW.>

# Public interface
- **Inputs:** `<Cmd/DTO>` — `{ <field>: <NeutralType>, … }`
- **Outputs:** `<Result type>` — `<what success returns>`
- **Errors:** `<ErrorCase>` — `<when>` (list all, regardless of error_style)

# Collaborators
<Every participant by id (also in depends_on); never re-describe its internals.>

| Id | Kind | Role in this flow |
|----|------|-------------------|
| `CLS-<id>` | service/controller | <what it does here> |
| `ENT-<id>` | entity | <object created/loaded/mutated> |

# Invariants & rules
<Cross-class rules for the whole orchestration: ordering, transactionality, idempotency, on-success/failure guarantees.>
- <Invariant 1>

# Orchestration SCoT
<One FUNCTION (the entry point), cross-class CALLs by id only. Grammar = scot.md; every branch a stable [Bn] with named arms; no library/syntax/internals.>

```
FUNCTION <entryPoint>(<param>: <Type>) -> <ReturnType>
  PRECONDITION: <holds on entry>      # optional
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
<End-to-end Given/When/Then, stable `ACn`, observable cross-class outcomes; together cover every orchestration arm. (GUI: the screen's e2e tests own the user-facing journey ACs.)>
- **AC1** — *Given* <state>, *When* `<entryPoint>` is invoked with `<input>`, *Then* <observable outcome + persisted/emitted effects>.
- **AC2** — *Given* <branch-triggering state>, *When* invoked, *Then* <alternate outcome, e.g. error + no side effects>.

---

## Filled example — `FEAT-010` Checkout

```yaml
id: FEAT-010 · name: Checkout · kind: use-case · module: MOD-shop
depends_on: [CLS-cartService, ENT-order, CLS-paymentGateway, CLS-orderRepo]
requirements: [REQ-014] · source: [src/features/checkoutFlow.ts] · error_style: result
```

**Purpose** — A shopper confirms a purchase: validate cart (non-empty, in stock), capture payment, and **only if authorized** create+persist the order and clear the cart. No order is ever half-created.

**Invariants** — no order persisted unless payment authorized; cart cleared only after the order is saved; checkout is one logical transaction (zero side effects on any failure).

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

- **AC1** — *Given* a cart with in-stock items + a valid token, *When* `checkout` runs, *Then* payment is captured, a confirmed order is persisted, the cart is cleared, `OrderConfirmed` is emitted, result `Ok(Order)`.
- **AC2** — *Given* a declined token, *When* `checkout` runs, *Then* `Err(PaymentDeclined)`, no order persisted, cart unchanged.
