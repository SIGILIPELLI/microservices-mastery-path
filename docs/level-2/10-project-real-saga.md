---
description: "A real saga orchestrator combining idempotency keys and durable SQLite state, surviving a real SIGKILL and retry with exactly one real charge."
---

# Project: A Real Saga With a Real Compensating Transaction

## What was run

A real `orchestrator_durable.py` (port 8080) combining three ideas from this level into one service: the saga from
[Lesson 4](04-saga-pattern.md), the idempotency-key pattern from [Lesson 6](06-idempotency-application-layer.md),
and durable SQLite storage like [Lesson 5](05-outbox-pattern-polling.md)'s outbox. A checkout was run, retried with
the same key, then the orchestrator process was killed with `kill -9` and restarted, and the same checkout was
retried a third time — all against the real `order_service.py`, `payment_service.py`, and `inventory_service.py`
from Lesson 4.

## The saga, run once, for real

```bash
curl -s -X POST localhost:8080/checkout -H 'Idempotency-Key: ckt-001' \
  -d '{"item":"mechanical keyboard","amount_cents":9000}'
# {"saga":"COMPLETED","replayed":false,
#  "order":{"id":11,"item":"mechanical keyboard","amount_cents":9000,"status":"CONFIRMED"},
#  "payment":{"id":8,"order_id":11,"amount_cents":9000,"status":"CHARGED"},
#  "reservation":{"id":4,"order_id":11,"item":"mechanical keyboard","status":"RESERVED"}}
```

```python
existing = conn.execute("SELECT result FROM saga_runs WHERE idempotency_key = ?", (idempotency_key,)).fetchone()
if existing:
    result = json.loads(existing[0])
    result["replayed"] = True
    return result
# ... only reached if no existing result: run the real saga, then persist the result before returning it
conn.execute("INSERT INTO saga_runs (idempotency_key, result) VALUES (?, ?)", (idempotency_key, json.dumps(result)))
```

## Retried, then the process is really killed, then retried again

```bash
curl -s -X POST localhost:8080/checkout -H 'Idempotency-Key: ckt-001' -d '{...same body...}'
# {"saga":"COMPLETED","replayed":true, ...same order/payment/reservation ids as before...}

pkill -9 -f "uvicorn orchestrator_durable:app"     # a real, hard crash
uvicorn orchestrator_durable:app --port 8080 &      # a fresh process, fresh memory, nothing cached in RAM

curl -s -X POST localhost:8080/checkout -H 'Idempotency-Key: ckt-001' -d '{...same body...}'
# {"saga":"COMPLETED","replayed":true, ...identical order id 11, payment id 8, reservation id 4...}

curl -s localhost:8072/payments
# [...,{"id":8,"order_id":11,"amount_cents":9000,"status":"CHARGED"}]   <- exactly one payment, id 8
```

Three real checkout requests with the same `Idempotency-Key`, spanning a real process crash in between, produced
**exactly one** real order, one real payment, and one real inventory reservation — confirmed directly against
`payment_service.py`'s own records, not just trusted from the orchestrator's response. The crash in the middle
changed nothing, because the fact "this idempotency key already has a result" lives in `saga_log.db` on disk, not
in the orchestrator process's memory.

## How It Actually Works

This project directly closes the gap that [Lesson 4](04-saga-pattern.md)'s own exercise raised: an orchestrator
that only tracks saga progress in memory loses all of it on a crash, leaving no record that a saga was ever in
flight. This version fixes the *retry* half of that problem — a client that doesn't know whether its request made
it through (because the orchestrator crashed, or the network dropped the response) can safely retry with the same
key and get the real, authoritative final answer, without risking a duplicate charge, exactly the property
[Lesson 6](06-idempotency-application-layer.md) built in isolation and this project now applies to an entire
multi-step saga, not just one endpoint.

What this project does **not** yet fully solve, and what's worth being honest about: it only persists the saga's
*final* result, not its in-progress state. If the orchestrator crashes *mid-saga* — after charging payment but
before reserving inventory — a retry with the same key would find no `saga_runs` row yet and **re-run the entire
saga from scratch**, potentially double-charging the customer. A fully crash-safe saga needs to persist *each
step's* completion, not just the end result, so a resumed saga can pick up exactly where it left off rather than
restarting — the natural next step beyond this project, and close to what dedicated saga/workflow engines (like
Temporal or AWS Step Functions) are built specifically to handle.

## Common mistakes

- Persisting only the final saga result (as this project deliberately does, to keep the example reachable) and
  mistaking that for full crash safety — the honest limitation above (a mid-saga crash can still double-run) is a
  real gap, not a hypothetical one.
- Using the idempotency key as the saga's only identifier without also being able to look up a saga by its
  resulting order ID, payment ID, etc. — in production you'll need both directions of lookup for support and
  debugging.
- Never expiring old `saga_runs` rows — an idempotency log that grows forever is a real, if slow-growing,
  operational problem; production systems typically expire keys after a bounded window (e.g., 24 hours), long
  enough to cover realistic retries, short enough not to grow unbounded.

## Exercise

Close the gap named above for real: add a `status` column to `saga_runs` (`IN_PROGRESS`, `COMPLETED`, `FAILED`),
write a row with `IN_PROGRESS` immediately after creating the order (before calling Payment), and update it as each
step completes. Then simulate a mid-saga crash by adding a `time.sleep(10)` right after the payment call and
killing the process during that sleep. Restart, retry with the same key, and observe what your current
implementation does (hint: it will likely re-run everything, since it only checks for a `COMPLETED` row) — then
write, in a comment, what check you'd need to add to detect and safely resume (or safely compensate) an
`IN_PROGRESS` saga instead of blindly re-running it.
