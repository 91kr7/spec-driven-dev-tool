<!--
  TEMPLATE — FEATURE / USE-CASE spec (kind: use-case, behavioral orchestration).
  Copy this file to specs/features/<FEAT-nnn>.spec.md and fill every field.
  Authority: .claude/sdd/conventions.md (ids §2, front-matter §3, sections §3, status §5).
  Body form for kind:use-case is SCoT — grammar in .claude/sdd/scot.md (orchestration:
  cross-class CALLs only). Markdown is the source of truth (authority); reuse over
  repetition (DRY) — name collaborators by id, never re-describe them here.
  The "## Filled example" at the bottom is a complete sample — delete it from a real spec.
-->
---
id: FEAT-<nnn>                       # required — matches the filename and the features.index row
name: <HumanReadableFeatureName>     # required — human name for the use-case
kind: use-case                       # required — fixed for this template (behavioral orchestration)
module: MOD-<kebab>                  # required — the feature's home module
status: draft                        # draft|reviewed|approved — MIRRORS features.index (index is canonical)
depends_on: [CLS-<lowerCamel>, ENT-<kebab>, SHR-<lowerCamel>]  # ids of every collaborator, topological order
requirements: [REQ-<nnn>]            # requirement id(s) this feature serves (back-link)
source: []                           # [] if purely compositional (no coordinator code → integration tests only);
                                     # else a single coordinator file, e.g. [src/features/<name>Flow.ts]
error_style: result                  # result|raise — declares the SCoT error style (.claude/sdd/scot.md §6)
---

# Purpose
<!-- One paragraph: the end-to-end user scenario this feature realizes and the
     business value it delivers. State the actor, the goal, and the outcome.
     Describe WHAT happens across collaborators, never HOW any one is built. -->
<Describe the use-case scenario here.>

# Public interface
<!-- The feature's entry point as seen by a caller (a screen/controller). Inputs,
     outputs, and the error cases (list them regardless of error_style — .claude/sdd/scot.md §6). -->

- **Inputs:** `<command/DTO name>` — `{ <field>: <NeutralType>, … }`
- **Outputs:** `<Result type>` — `<what success returns>`
- **Errors:**
  - `<ErrorCase1>` — `<when it occurs>`
  - `<ErrorCase2>` — `<when it occurs>`

# Collaborators
<!-- Reference every participant BY ID only (CLS-*/ENT-*/COMP-*/SHR-*). Do NOT
     re-describe their fields or behavior — that lives in their own spec. One row
     per collaborator; "Role in this flow" says why this feature calls it. Every id
     here MUST also appear in depends_on. (reuse over repetition / DRY) -->

| Id                  | Kind   | Role in this flow                          |
|---------------------|--------|--------------------------------------------|
| `CLS-<lowerCamel>`  | service / controller | <what it does for this use-case> |
| `ENT-<kebab>`       | entity | <the domain object created/loaded/mutated> |
| `SHR-<lowerCamel>`  | shared | <the shared abstraction used>              |
| `COMP-<lowerCamel>` | gui    | <the screen/component, if any, that triggers this> |

# Invariants & rules
<!-- Cross-class rules that hold for the whole orchestration (ordering guarantees,
     transactionality, idempotency, what must be true on success/failure). -->

- <Invariant 1, e.g. "No order row is persisted unless payment was authorized.">
- <Invariant 2.>

# Orchestration SCoT
<!-- HIGH-LEVEL sequence of CROSS-CLASS CALLs only (one FUNCTION, the entry point).
     Grammar = .claude/sdd/scot.md. Reference collaborators by id in every CALL. Each
     decision point carries a stable branch id [Bn]; arms get ids per .claude/sdd/scot.md §4
     (B1.then / B1.else / B6.catch:<Error> …). No library names, no language syntax,
     no per-method internals — those belong to the collaborators' own specs and to
     .sdd/impl-notes/. Keep ids stable across re-runs (never renumber). -->

```
error-style: <result|raise>          # MUST match the error_style front-matter

FUNCTION <entryPoint>(<param>: <Type>) -> <ReturnType>
  INPUT:  <param> — <meaning>
  OUTPUT: <what the return value means>
  PRECONDITION: <what must hold on entry>      # optional

  <step> <- CALL CLS-<lowerCamel>.<method>(<args>)

  [B1] IF <cross-class condition> THEN
    RETURN <Err(...) | failure outcome>        # arm B1.then
  ELSE
    # arm B1.else — continue
  END

  <result> <- CALL ENT-<kebab>.<factory-or-method>(<args>)
  RETURN <Ok(result) | success outcome>
END
```

# Integration acceptance criteria
<!-- End-to-end Given/When/Then, each with a stable ACn id (.claude/sdd/conventions.md §2).
     These drive the INTEGRATION tests (one per feature). Each AC should exercise a
     full cross-class path; together they should cover every orchestration branch arm
     above. State observable outcomes across collaborators, not internal calls. -->

- **AC1** — *Given* <initial cross-class state>, *When* `<entryPoint>` is invoked with `<input>`, *Then* <observable end-to-end outcome and persisted/emitted effects>.
- **AC2** — *Given* <state triggering a branch>, *When* `<entryPoint>` is invoked with `<input>`, *Then* <the alternate outcome, e.g. an error and no side effects>.

---

## Filled example
<!-- A complete, valid instance of this template. Everything below is real content
     (no placeholders). Note the second front-matter block belongs to the example. -->

```yaml
---
id: FEAT-010
name: Checkout
kind: use-case
module: MOD-shop
status: draft
depends_on: [CLS-cartService, ENT-order, CLS-paymentGateway, CLS-orderRepo]
requirements: [REQ-014]
source: [src/features/checkoutFlow.ts]
error_style: result
---
```

### Purpose
A shopper with a populated cart confirms their purchase. Checkout validates that
the cart is non-empty and in stock, captures payment for the cart total, and — only
if payment is authorized — creates and persists an order, then clears the cart. The
shopper receives a confirmed order or a precise failure reason, and no order is ever
left half-created.

### Public interface

- **Inputs:** `CheckoutCmd` — `{ cartId: UUID, paymentToken: String, shippingAddress: Address }`
- **Outputs:** `Result<Order, CheckoutError>` — `Ok(Order)` with a `confirmed` order on success
- **Errors:**
  - `EmptyCart` — the cart has no line items
  - `OutOfStock` — one or more line items are no longer purchasable
  - `PaymentDeclined` — the payment gateway rejected the charge

### Collaborators

| Id                  | Kind    | Role in this flow                                          |
|---------------------|---------|------------------------------------------------------------|
| `CLS-cartService`   | service | Loads the cart, checks stock, and clears it after success  |
| `ENT-order`         | entity  | The domain order created from the cart and total           |
| `CLS-paymentGateway`| service | Captures payment for the cart total against the token      |
| `CLS-orderRepo`     | service | Persists the confirmed order                               |

### Invariants & rules

- No order row is persisted unless payment for the full cart total was authorized.
- The cart is cleared **only** after the order is successfully persisted (so a
  persistence failure leaves the shopper's cart intact to retry).
- Checkout is a single logical transaction: on any failure there are zero side
  effects (no charge captured stands without an order, no order without a charge).

### Orchestration SCoT

```
error-style: result

FUNCTION checkout(cmd: CheckoutCmd) -> Result<Order, CheckoutError>
  INPUT:  cmd — { cartId, paymentToken, shippingAddress }
  OUTPUT: Ok(Order) when the order is confirmed and persisted; Err(CheckoutError) otherwise
  PRECONDITION: cmd is structurally valid (DTO-validated)

  cart <- CALL CLS-cartService.load(cmd.cartId)

  [B1] IF cart.isEmpty THEN
    RETURN Err(EmptyCart)                              # arm B1.then
  ELSE
    # arm B1.else — cart has items, continue
  END

  [B2] IF NOT CALL CLS-cartService.allInStock(cart) THEN
    RETURN Err(OutOfStock)                             # arm B2.then
  END
  # arm B2.else — all line items purchasable

  payment <- CALL CLS-paymentGateway.capture(cmd.paymentToken, cart.total)

  [B3] IF NOT payment.authorized THEN
    RETURN Err(PaymentDeclined)                        # arm B3.then
  ELSE
    # arm B3.else — payment captured
  END

  order <- CALL ENT-order.fromCart(cart, payment, cmd.shippingAddress)
  saved <- CALL CLS-orderRepo.save(order)
  CALL CLS-cartService.clear(cmd.cartId)

  EMIT OrderConfirmed({ orderId: saved.id, total: saved.total })
  LOG info "checkout confirmed"
  RETURN Ok(saved)
END
```

Coverage set for `checkout`: `B1.then`, `B1.else`, `B2.then`, `B2.else`,
`B3.then`, `B3.else`.

### Integration acceptance criteria

- **AC1** — *Given* a cart with two in-stock line items and a valid payment token,
  *When* `checkout` is invoked, *Then* the payment is captured for the cart total, a
  `confirmed` order is persisted via `CLS-orderRepo`, the cart is cleared, an
  `OrderConfirmed` event is emitted, and the result is `Ok(Order)`.
- **AC2** — *Given* a cart whose payment token is declined by `CLS-paymentGateway`,
  *When* `checkout` is invoked, *Then* the result is `Err(PaymentDeclined)`, no order
  is persisted, and the cart is left unchanged (zero side effects).
