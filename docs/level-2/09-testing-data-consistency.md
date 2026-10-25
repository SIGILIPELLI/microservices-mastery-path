---
description: "A real cross-service consistency test passes against a correct saga, then fails with a real AssertionError against one missing a compensation call."
---

# Testing Data Consistency Across Services

## What was run

A real Python test script, `test_saga_consistency.py`, calling the live saga from [Lesson 4](04-saga-pattern.md)
over real HTTP and asserting a cross-service invariant: if the saga fails at the inventory step, the payment it
already charged must end up `REFUNDED`, not left `CHARGED`. It was run twice: once against the correct
`orchestrator.py`, and once against a deliberately broken copy, `orchestrator_buggy.py`, with the refund
compensation call removed.

## The invariant that matters, stated as a real assertion

```python
def test_out_of_stock_item_refunds_the_payment():
    r = httpx.post(f"{ORCHESTRATOR}/checkout", json={"item": "discontinued-widget", "amount_cents": 1500}, timeout=10)
    body = r.json()
    assert body["saga"] == "FAILED"
    assert body["failed_step"] == "inventory"

    latest = [p for p in httpx.get(f"{PAYMENT}/payments").json() if p["amount_cents"] == 1500][-1]
    assert latest["status"] == "REFUNDED", (
        f"consistency violation: payment {latest['id']} is '{latest['status']}', "
        f"but the saga it belongs to FAILED at the inventory step -- it should have been refunded"
    )
```

```bash
python3 test_saga_consistency.py
# PASS: test_successful_checkout_confirms_order
# PASS: test_out_of_stock_item_refunds_the_payment
```

## A one-line regression, and the test that actually catches it

```python
# orchestrator_buggy.py -- a "simplification" that deletes the refund call
inv_resp = http.post(f"{INVENTORY_URL}/reservations", json={"order_id": order["id"], "item": c.item})
if inv_resp.status_code != 200:
    http.post(f"{ORDER_URL}/orders/{order['id']}/cancel")   # <- refund call silently missing here
    return {"saga": "FAILED", "failed_step": "inventory", ...}
```

```bash
curl -s -X POST localhost:8075/checkout -H 'Content-Type: application/json' \
  -d '{"item":"discontinued-widget","amount_cents":1800}'
# {"saga":"FAILED","failed_step":"inventory",
#  "order":{"id":9,"item":"discontinued-widget","amount_cents":1800,"status":"CANCELLED"},
#  "payment":{"id":6,"order_id":9,"amount_cents":1800,"status":"CHARGED"}}
```

The order correctly shows `CANCELLED` — that part of the compensation still runs. The payment, real money charged
in this simulation, is left `CHARGED` forever. Running the same consistency test against this buggy version:

```bash
python3 test_saga_consistency_buggy.py
# AssertionError: consistency violation: payment 7 is 'CHARGED', but the saga it belongs to
# FAILED at the inventory step -- it should have been refunded
```

A real `AssertionError`, on a real bug, caught by a test that calls real running services over real HTTP — not a
unit test of any single service's internal logic, because no single service's internal logic is wrong here.
Payment Service correctly charges when asked, correctly refunds when asked; Order Service correctly cancels when
asked. The bug is entirely in the *interaction* — the orchestrator forgetting to ask for a refund — which is
precisely the class of bug that disappears from view if you only ever test services in isolation.

## How It Actually Works

Unit tests, run against one service in isolation (often with its dependencies mocked), can tell you a service does
what it claims given a specific input. They cannot tell you whether the *system* of services, working together,
upholds a cross-service invariant — "a cancelled order never leaves a charged payment behind" is a property of the
whole saga, not of any one service's code. Testing this requires either:

- **A real integration test**, as built here: start the real services (or realistic stand-ins), drive them through
  a real scenario over the network, and assert on the *combined* resulting state across services — exactly what
  this lesson's test does.
- **Consumer-driven contract tests** (covered in [Level 3, Lesson 9](../level-3/09-consumer-driven-contracts.md)),
  which catch a different but related class of problem: one service breaking another's expectations of its API,
  without needing every service running at once.

The specific pattern worth generalizing from this test: identify the saga's **failure-path invariants** — not just
"the happy path works," but "every compensation that should run for every real failure mode, actually runs" — and
write one assertion per invariant, checked against real post-failure state in every service that participated.
Lesson 4's saga has two failure paths (payment declined, inventory out of stock); a thorough test suite needs an
invariant check for each, not just the happy path shown as `test_successful_checkout_confirms_order` above.

## Common mistakes

- Testing only the saga's happy path and never deliberately forcing each failure branch — the bug caught above
  lives entirely inside a failure branch that a happy-path-only test suite would never execute.
- Asserting only on the orchestrator's immediate HTTP response ("saga": "FAILED") without also checking the actual
  downstream state in each participating service — the buggy orchestrator's response here doesn't even mention the
  payment at all in some code paths, so a test that only reads the orchestrator's reply could easily miss this.
- Running this kind of test against shared, stateful services without resetting state between runs — this demo's
  services accumulate orders and payments across test runs (note the real IDs climbing: `payment 6`, `payment 7`),
  which is fine for a quick manual check but would need a reset or unique-per-test identifiers in a real CI suite.

## Exercise

Add a third invariant test for the payment-declined failure path from Lesson 4: assert that when a payment is
declined (amount over the fraud limit), **no** inventory reservation was ever created for that order. Then
introduce a *different* deliberate bug — have the buggy orchestrator reserve inventory *before* checking the
payment response — and confirm your new test catches that ordering bug too.
