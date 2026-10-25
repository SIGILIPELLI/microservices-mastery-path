---
description: "A real client polling an order mid-saga, measuring a genuine 2.56-second window where the order is visibly PENDING before the saga confirms it."
---

# Handling Partial Failures & Eventual Consistency

## What was run

The real saga from [Lesson 4](04-saga-pattern.md) — `order_service.py`, `payment_service.py`,
`inventory_service.py`, `orchestrator.py` — with a real `time.sleep(2.5)` added to `payment_service.py`'s `/payments`
endpoint, standing in for a real card-network round trip that isn't instant. A real Python script then started a
checkout in a background thread and polled the order's status directly, with real timestamps, while the saga was
still running.

## The inconsistency window, measured with real timestamps

```python
t.start()                                      # kicks off POST /checkout in a background thread
time.sleep(0.3)
r = httpx.get(f'http://localhost:8071/orders/{last_oid}')
print(f"[t+{time.time()-t0:.2f}s] status while saga in flight: {r.json()['status']}")
```

```
tracking order id 6
[t+0.52s] status while saga in flight: PENDING
[t+2.06s] status while saga in flight: PENDING
[t+2.56s] saga finished: COMPLETED
[t+2.58s] final status: CONFIRMED
```

For a real, measured **2.56 seconds**, any client that looked up order 6 — a status page, a mobile app polling for
confirmation, a different service checking whether it's safe to proceed — would have seen `PENDING`, a status that
is truthful (the order genuinely isn't confirmed yet) but incomplete (the order is, in fact, in the middle of
successfully completing, not stuck or failed). Only after the full saga finished did the status flip to `CONFIRMED`.

## How It Actually Works

A monolith processing this same checkout inside one database transaction would show the client either "order
doesn't exist yet" or "order exists and is fully confirmed" — there's no observable in-between state, because the
whole operation is atomic from any outside observer's point of view. A saga, by construction, cannot offer that: it
is a sequence of independently committing local transactions, so there necessarily exists a real window, after step
1 commits and before the last step commits, where the system's state is **genuinely partial** — true at that
instant, but not the final answer.

This is what "eventual consistency" concretely means: the system *will* reach a consistent, correct final state
(`CONFIRMED` or `CANCELLED`, never stuck forever assuming the saga completes or is properly compensated), but not
*instantly*, and any reader during the window in between sees a real, valid, but temporary state. The practical
questions this raises, all of which need an explicit answer rather than an accidental one:

- **What should the client do with a `PENDING` status?** Often: show a "processing" state, not an error, and poll
  or subscribe for the final result rather than treating `PENDING` as a failure.
- **What should a *different service* do if it reads `PENDING` mid-saga?** If Shipping checked order 6's status
  during that window to decide whether to prepare a shipment, it needs to know `PENDING` means "don't act yet,"
  not "this order failed" — the states a saga can be in, and what's safe to do in each, need to be part of the
  contract other services design against.
- **How long is the window allowed to be?** 2.56 seconds here came from one slow payment call; a saga with more
  steps, or a slower step, has a proportionally longer window — worth measuring deliberately, not discovering in
  production.

## Common mistakes

- Treating `PENDING` (or any intermediate state) as an error case in client code, causing a user to see a failure
  message for an operation that's actually still succeeding, just not finished yet.
- Letting a different service make an irreversible decision based on a mid-saga intermediate state without
  understanding that the state might still change — e.g., reserving a shipping slot for an order that might still
  be compensated into `CANCELLED` a moment later.
- Assuming the inconsistency window is always short enough not to matter — measure it, as this lesson did, rather
  than guessing; a saga calling a genuinely slow external system (a real payment processor, not a 2.5-second
  simulation) can have a window measured in many seconds, not fractions of one.

## Exercise

Add a fourth, intermediate status to `order_service.py` — `AWAITING_PAYMENT` — and have the orchestrator set it
right after creating the order but before calling Payment Service, instead of leaving the status at the generic
`PENDING` for the whole saga. Repeat this lesson's timed polling experiment and confirm you can now distinguish,
from the outside, "waiting on payment" from "waiting on inventory" during the saga's real execution window, not just
"something is still pending."
