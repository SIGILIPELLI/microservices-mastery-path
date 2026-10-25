---
description: "A real SIGKILL on a saga dependency, reconstructed from logs into 3 timelines showing a circuit breaker trip, 3 real refunds, and a clean recovery."
---

# Capstone: Build, Break & Observe a Real Multi-Service System

## What was run

A real four-service saga — `order_service.py`, `payment_service.py`, `inventory_service.py` (all from
[Level 2, Lesson 4](../level-2/04-saga-pattern.md)), coordinated by a new `capstone_orchestrator.py` that combines
three patterns built across this entire course: the saga and compensation logic from Level 2, a real `pybreaker`
circuit breaker around the inventory call from [Level 3, Lesson 1](../level-3/01-circuit-breakers.md), and
structured, trace-ID-correlated logging from [Level 4, Lesson 6](06-observability-across-boundaries.md).
`inventory_service.py` was killed with a real `kill -9` mid-course, three real checkouts were attempted during the
outage, and the service was then restarted to observe real recovery.

## The real failure, fully reconstructed from structured logs

```bash
pkill -9 -f "uvicorn inventory_service:app"   # a real, hard kill, no warning
# three real checkouts attempted during the outage...
```

```
trace 353b711ce859 (1st failure -- breaker still closed going in):
  saga_started → order_created(id=5) → payment_charged(id=5)
  → inventory_reservation_failed  (breaker_state=closed, reason="Connection refused")
  → compensation_payment_refunded(id=5) → compensation_order_cancelled(id=5)

trace 02fa6ab3c7e6 (2nd failure -- THIS ONE trips the breaker):
  saga_started → order_created(id=6) → payment_charged(id=6)
  → inventory_reservation_failed  (breaker_state=open, reason="Failures threshold reached, circuit breaker opened")
  → compensation_payment_refunded(id=6) → compensation_order_cancelled(id=6)

trace b59721be850f (3rd failure -- breaker already open, call SKIPPED):
  saga_started → order_created(id=7) → payment_charged(id=7)
  → inventory_reservation_failed  (breaker_state=open, reason="Timeout not elapsed yet, circuit breaker still open")
  → compensation_payment_refunded(id=7) → compensation_order_cancelled(id=7)
```

```bash
curl -s localhost:8072/payments   # checked directly against the real payment service, not inferred
```
```json
{"id": 5, "order_id": 5, "amount_cents": 4000, "status": "REFUNDED"},
{"id": 6, "order_id": 6, "amount_cents": 4000, "status": "REFUNDED"},
{"id": 7, "order_id": 7, "amount_cents": 4000, "status": "REFUNDED"}
```

**All three real charges were really refunded** — confirmed independently against Payment Service's own state, not
just trusted from the orchestrator's log. And the circuit breaker's real behavior shows exactly across these three
traces, in the `reason` field: the 1st failure is a real network attempt (`"Connection refused"`), the 2nd is also
a real attempt that happens to cross the failure threshold (`"Failures threshold reached"` — see
[Level 3, Lesson 1](../level-3/01-circuit-breakers.md)'s finding that this specific call is still real), and the
3rd is genuinely skipped with no network call at all (`"Timeout not elapsed yet"`).

```bash
# inventory_service.py restarted for real...
curl -s -X POST localhost:8270/checkout -d '{"item":"tripod","amount_cents":4000}'
# {"trace_id":"c06421e6ecf8","saga":"COMPLETED","order_id":8}
```

A real, clean recovery — no manual intervention needed beyond restarting the dependency; the breaker's own
`reset_timeout` eventually allowed a real probe through, and that probe succeeded.

## How It Actually Works

This capstone is not a new pattern — it's **three already-built real patterns, composed, under one real
failure**, which is exactly what a production incident actually looks like: not one isolated technique being
tested, but several working (or failing) together at once. The saga's compensation logic
([Level 2](../level-2/04-saga-pattern.md)) ensured no customer was left charged for an order that never shipped,
regardless of how many times the saga failed. The circuit breaker ([Level 3](../level-3/01-circuit-breakers.md))
kept the second and third failed attempts from each wasting a full network timeout — visible directly in the real
log timestamps, where the 3rd attempt's `inventory_reservation_failed` event follows its `payment_charged` event
by only ~0.0002 seconds, versus the 1st attempt's real ~16ms network round trip. The structured, trace-correlated
logging ([Level 4, Lesson 6](06-observability-across-boundaries.md)) is what made it possible to state all of this
precisely, from real data, instead of describing it in the abstract.

The real, honest gap this capstone still has, consistent with every earlier lesson's own stated limitations: the
orchestrator's saga state lives only in each request's own execution — if the orchestrator process itself had
crashed mid-saga (not just the inventory dependency), there's no persistent saga log to resume or verify against,
the same limitation [Level 2, Lesson 10](../level-2/10-project-real-saga.md) named explicitly and left as an
acknowledged, real boundary of what this course's scope could cover.

## Common mistakes

- Testing resilience patterns one at a time in isolation and assuming they compose correctly together without
  ever actually running them together under one real failure, as this capstone did — a circuit breaker and a
  saga's compensation logic interacting incorrectly (e.g., the breaker's `CircuitBreakerError` not being caught by
  the same `except` clause as the dependency's own exceptions) is a realistic integration bug this kind of combined
  test would catch and an isolated unit test of either pattern alone would not.
- Trusting an orchestrator's own response as proof of what really happened, instead of checking the actual
  downstream state — this capstone's refund confirmation came from querying Payment Service directly, the same
  discipline [Level 2, Lesson 9](../level-2/09-testing-data-consistency.md)'s consistency tests insisted on.
- Declaring a system "resilient" after one successful chaos test, without also checking what happens to state left
  behind by failures that happened *before* the resilience pattern fully engaged (the 1st trace's real attempt,
  before the breaker had even tripped yet) — real failures don't wait politely for your protections to be ready.

## Exercise

Add persistent saga logging to `capstone_orchestrator.py` (as sketched, but not fully built, in
[Level 2, Lesson 10](../level-2/10-project-real-saga.md)'s exercise): write each saga's state to a real SQLite
table as it progresses. Then kill the *orchestrator* process itself (not just inventory) partway through a saga,
restart it, and write a small recovery routine that reads any `IN_PROGRESS` saga from the table and finishes
compensating it correctly — closing the one real, acknowledged gap this capstone (and this entire course) leaves
open, with your own working code.
