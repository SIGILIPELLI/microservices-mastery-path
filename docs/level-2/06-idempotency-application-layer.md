---
description: "A real double-charge from a retried request against a naive endpoint, versus an idempotency-key-protected endpoint producing exactly one real charge."
---

# Idempotency at the Application Layer

## What was run

A real `payment_service.py` (port 8091) with two endpoints against the exact same business operation: a naive one
with no protection, and one that checks an `Idempotency-Key` header before charging. Both were called multiple
times with `curl`, simulating a client that retries a request after not receiving a response — a real, common
failure mode whenever a network call times out and the client can't tell whether the server actually processed it
before the timeout.

## The naive endpoint, retried, charges twice for real

```python
@app.post("/charges-naive")
def charge_naive(c: ChargeIn):
    cid = next(seq)
    charges.append({"id": cid, "amount_cents": c.amount_cents})
    return {"id": cid, "amount_cents": c.amount_cents}
```

```bash
curl -s -X POST localhost:8091/charges-naive -H 'Content-Type: application/json' -d '{"amount_cents":5000}'
# {"id":1,"amount_cents":5000}
curl -s -X POST localhost:8091/charges-naive -H 'Content-Type: application/json' -d '{"amount_cents":5000}'
# {"id":2,"amount_cents":5000}

curl -s localhost:8091/charges-naive/all
# [{"id":1,"amount_cents":5000},{"id":2,"amount_cents":5000}]
```

Two real charge records, `id: 1` and `id: 2`, for what was meant to be one $50.00 charge. In a real payment system,
this is a customer getting billed twice because their client (or a retry layer, or a flaky mobile network) sent the
same logical request more than once — exactly the scenario [Level 1, Lesson 4](../level-1/04-sync-vs-async-communication.md)'s
async retry loop would create if it ever retried a *non*-idempotent call.

## The idempotent endpoint, retried, charges exactly once

```python
@app.post("/charges-idempotent")
def charge_idempotent(c: ChargeIn, idempotency_key: str = Header(...)):
    if idempotency_key in idempotency_store:
        return {**idempotency_store[idempotency_key], "replayed": True}   # same result, no new charge
    cid = next(seq)
    result = {"id": cid, "amount_cents": c.amount_cents, "replayed": False}
    idempotency_store[idempotency_key] = result
    return result
```

```bash
curl -s -X POST localhost:8091/charges-idempotent -H 'Idempotency-Key: req-abc-123' -d '{"amount_cents":5000}'
# {"id":3,"amount_cents":5000,"replayed":false}
curl -s -X POST localhost:8091/charges-idempotent -H 'Idempotency-Key: req-abc-123' -d '{"amount_cents":5000}'
# {"id":3,"amount_cents":5000,"replayed":true}
curl -s -X POST localhost:8091/charges-idempotent -H 'Idempotency-Key: req-abc-123' -d '{"amount_cents":5000}'
# {"id":3,"amount_cents":5000,"replayed":true}

curl -s localhost:8091/charges-idempotent/all
# [{"id":3,"amount_cents":5000,"replayed":false}]
```

Three real HTTP requests, the same `Idempotency-Key` on all three, and exactly one real charge record (`id: 3`) in
the final list. The second and third calls returned the *same* `id: 3` with `"replayed": true`, telling the caller
plainly that nothing new happened — they got back the original result, not a fresh charge.

## How It Actually Works

An idempotent operation is one that produces the same effect no matter how many times it's performed with the same
input. `GET` and `DELETE` are naturally idempotent (reading twice doesn't change anything; deleting an
already-deleted thing is a no-op). `POST` — "create a new charge" — is inherently *not* idempotent by default,
because its whole point is to create something new each time, which is exactly the naive endpoint's problem above.

The **idempotency key** pattern fixes this at the application layer by having the *caller* generate a unique key
per logical operation (a UUID generated once, before the first attempt) and send it on every retry of that same
logical operation. The server's job is then to remember, for each key, the result it already returned, and return
that same result again — without re-running the side-effecting logic — for any repeat of that key. This requires the
key store itself to be durable across the same kind of crash discussed in [Lesson 5](05-outbox-pattern-polling.md):
this demo's in-memory `idempotency_store` dict would lose every key on a restart, which would make every pending
retry look brand-new again — a real production implementation stores idempotency keys in the same durable database
as the charges themselves, ideally checked and written in the same transaction as the charge.

This is a necessary complement to the saga pattern from [Lesson 4](04-saga-pattern.md): a saga step that calls
Payment Service might itself be retried (by the orchestrator, after a timeout it couldn't distinguish from a real
failure) — without idempotency protection on that call, a saga retry could double-charge a customer even though the
saga's own logic was working exactly as designed.

## Common mistakes

- Generating a new idempotency key on every retry instead of reusing the same one generated at the start of the
  logical operation — this defeats the entire mechanism, since the server has no way to recognize a retry from a
  genuinely new request.
- Keying the idempotency store only by a short-lived in-memory structure (as this demo deliberately does, to keep
  the example small) — a real implementation needs the key store to survive a restart, or every pending retry after
  a deploy becomes a fresh, unprotected charge.
- Treating idempotency keys as a replacement for distinguishing genuinely different requests with the same
  parameters — two real, separate $50 charges by the same customer on the same day need two different keys; the key
  identifies the *request attempt*, not the payment amount.

## Exercise

Add a check to `charge_idempotent` that returns a real `409 Conflict` if the *same* `Idempotency-Key` is reused with
a *different* `amount_cents` than the first time it was seen — this catches a real bug class (a client reusing a key
across genuinely different logical operations) that the current implementation silently ignores by just replaying
the original amount. Test it with `curl` using the same key twice but a different amount the second time, and
confirm you get a real `409`, not a silently wrong replayed result.
