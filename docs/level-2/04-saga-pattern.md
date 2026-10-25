---
description: "A real 3-service orchestrated saga with two genuine failure points, each triggering real compensating transactions that undo prior steps in reverse order."
---

# The Saga Pattern: Orchestration vs Choreography

## What was run

Four real services: `order_service.py` (8071), `payment_service.py` (8072, which really declines payments over
$1000), `inventory_service.py` (8073, which really rejects one specific item as out of stock), and
`orchestrator.py` (8070) coordinating all three. Three real checkouts were run: one that succeeds completely, one
that fails at the payment step, and one that fails at the inventory step *after* payment already succeeded —
exercising a real multi-step compensation.

## A saga that completes

```bash
curl -s -X POST localhost:8070/checkout -H 'Content-Type: application/json' -d '{"item":"laptop","amount_cents":50000}'
# {"saga":"COMPLETED","order":{"id":1,"item":"laptop","amount_cents":50000,"status":"CONFIRMED"},
#  "payment":{"id":1,"order_id":1,"amount_cents":50000,"status":"CHARGED"},
#  "reservation":{"id":1,"order_id":1,"item":"laptop","status":"RESERVED"}}
```

Three real services, three real state changes, one logical transaction with no shared database underneath it.

## Failure at step 2: one compensation

```bash
curl -s -X POST localhost:8070/checkout -H 'Content-Type: application/json' -d '{"item":"yacht","amount_cents":500000}'
# {"saga":"FAILED","failed_step":"payment",
#  "reason":"payment declined: amount_cents 500000 exceeds fraud limit 100000",
#  "compensations_run":["order_cancelled"],
#  "order":{"id":2,"item":"yacht","amount_cents":500000,"status":"CANCELLED"}}
```

```python
pay_resp = http.post(f"{PAYMENT_URL}/payments", json={"order_id": order["id"], "amount_cents": c.amount_cents})
if pay_resp.status_code != 200:
    http.post(f"{ORDER_URL}/orders/{order['id']}/cancel")   # undo step 1
    return {"saga": "FAILED", "failed_step": "payment", "compensations_run": ["order_cancelled"], ...}
```

Order #2 really exists in `order_service.py`'s memory with status `CANCELLED`, not deleted — the saga's failure is
recorded, not erased. No payment record was created at all, confirmed directly: `curl -s localhost:8072/payments`
still shows only payment `id: 1` from the first, successful checkout.

## Failure at step 3: two compensations, in reverse order

```bash
curl -s -X POST localhost:8070/checkout -H 'Content-Type: application/json' -d '{"item":"discontinued-widget","amount_cents":2000}'
# {"saga":"FAILED","failed_step":"inventory",
#  "reason":"'discontinued-widget' is out of stock",
#  "compensations_run":["payment_refunded","order_cancelled"],
#  "order":{"id":3,"item":"discontinued-widget","amount_cents":2000,"status":"CANCELLED"},
#  "payment":{"id":2,"order_id":3,"amount_cents":2000,"status":"REFUNDED"}}
```

```python
inv_resp = http.post(f"{INVENTORY_URL}/reservations", json={"order_id": order["id"], "item": c.item})
if inv_resp.status_code != 200:
    http.post(f"{PAYMENT_URL}/payments/{payment['id']}/refund")    # undo step 2
    http.post(f"{ORDER_URL}/orders/{order['id']}/cancel")           # undo step 1, in reverse order
    return {"saga": "FAILED", "failed_step": "inventory", "compensations_run": ["payment_refunded", "order_cancelled"], ...}
```

By the time inventory rejected `discontinued-widget`, payment `id: 2` had *really* charged $20.00 against order 3.
The orchestrator's compensation logic ran a real `POST /payments/2/refund`, flipping its status to `REFUNDED` — a
second, separate, real state transition, not a rollback of the first transaction, because there is no shared
transaction to roll back.

## How It Actually Works

A saga is a sequence of local transactions, each in a different service, where each step either succeeds and moves
to the next step, or fails and triggers **compensating transactions** that semantically undo every step that already
succeeded — in reverse order, as shown above. This is fundamentally different from a database rollback: a
compensating transaction is a *new*, forward-moving operation (`REFUNDED`, `CANCELLED`) that happened at a real
point in time, not an erasure of history. This matters because other things may have already observed the
intermediate state — a dashboard might have already shown "payment charged" for order 3 before the refund ran, and
that was true at the time, even though the final state is "refunded."

This demo implements **orchestration**: one coordinator (`orchestrator.py`) explicitly calls each step in order and
explicitly decides what to compensate on failure. The alternative, **choreography**, has no central coordinator:
each service reacts to events from the previous step and emits its own event when done, with compensation triggered
by services reacting to a "step failed" event the same way they reacted to a "step succeeded" one. Concretely, a
choreographed version of this saga would have Order Service publish `OrderCreated`, Payment Service subscribe to it
and publish `PaymentCharged` or `PaymentFailed`, and Inventory Service subscribe to `PaymentCharged` — with Order
Service *also* subscribing to `PaymentFailed` to cancel itself, instead of a coordinator calling `cancel` directly.

The real tradeoff: orchestration keeps the whole saga's logic in one readable place (as this demo's `checkout`
function shows end-to-end) but creates a coordinator that knows about every participating service. Choreography
removes that central coordinator and lets each service stay fully independent, but the saga's actual logic — "what
happens when payment fails" — is now scattered across every service's event handlers, with no single place to read
the whole flow, which gets genuinely hard to reason about past 3–4 steps.

## Common mistakes

- Treating the orchestrator's `checkout` function as if it ran inside one transaction — it does not; a crash
  between step 2 and step 3 (payment charged, inventory call never made) leaves the system in `CHARGED`/no
  reservation until something retries or compensates it, which is exactly the durability gap
  [Level 1, Lesson 4](../level-1/04-sync-vs-async-communication.md) first exposed and
  [Lesson 5](05-outbox-pattern-polling.md) of this level addresses properly.
- Writing a compensating transaction that assumes the original action fully succeeded, with no handling for "what if
  the thing I'm compensating never actually finished" — a refund call to a payment that was never actually charged
  needs to be a safe no-op, not a crash.
- Choosing choreography for its decoupling benefits without first making sure every team involved can actually find
  and read every event handler across every service — the "no central coordinator" benefit becomes a real debugging
  cost the first time someone has to trace a failed saga across five services' logs instead of one orchestrator's.

## Exercise

Kill the `orchestrator.py` process with `kill -9` *between* the payment-charge call and the inventory-reservation
call (add a `time.sleep(10)` right there temporarily, and kill it during the sleep). Restart it, and confirm for
yourself, by querying `/payments` and `/reservations` directly, that order's payment is left `CHARGED` forever with
no reservation and no compensation — because the in-memory orchestrator has no record that this saga was ever in
progress. Write one sentence on what persistent state the orchestrator would need to recover and finish (or
compensate) that saga after a real restart.
