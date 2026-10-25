---
description: "Measuring synchronous vs asynchronous service calls for real by killing a dependency, showing a real 502 versus a real queued retry that recovers."
---

# Inter-Service Communication: Synchronous vs Asynchronous

## What was run

Three real local services: `shipping_service.py` (port 8011), a synchronous `order_service_sync.py` (port 8012) that
calls Shipping and waits, and an asynchronous `order_service_async.py` (port 8013) that queues the shipment request
in a background thread and returns immediately. All three were started with `uvicorn`, and Shipping was killed with
`pkill` mid-course to observe real failure behavior in each.

## The synchronous version, with Shipping alive, then killed

```python
@app.post("/orders")
def create_order(o: OrderIn):
    oid = next(order_id_seq)
    orders[oid] = {"id": oid, "item": o.item}
    resp = httpx.post(f"{SHIPPING_URL}/shipments", json={"order_id": oid, "address": o.address}, timeout=2.0)
    resp.raise_for_status()
    return orders[oid]
```

```bash
curl -s -X POST localhost:8012/orders -H 'Content-Type: application/json' \
  -d '{"item":"desk lamp","address":"221B Baker St"}'
# {"id":1,"item":"desk lamp"}

pkill -f "uvicorn shipping_service:app"

curl -s -w "\nHTTP:%{http_code} time:%{time_total}\n" -X POST localhost:8012/orders \
  -H 'Content-Type: application/json' -d '{"item":"office chair","address":"10 Downing St"}'
# {"detail":"order created but shipment call failed: [Errno 61] Connection refused"}
# HTTP:502 time:0.021181
```

That `502` is real, and it hides a genuinely awkward fact: `orders[oid] = {...}` already ran *before* the failed
call. The order exists in Order Service's memory with `id: 2`, but the client received an error and has no way to
know the order was actually created. This exact gap — a side effect that happened, paired with a response that says
it didn't — is precisely the problem the saga pattern in [Level 2, Lesson 4](../level-2/04-saga-pattern.md) exists to
solve properly.

## The asynchronous version, with Shipping dead the whole time

```python
@app.post("/orders")
def create_order(o: OrderIn):
    oid = next(order_id_seq)
    orders[oid] = {"id": oid, "item": o.item}
    pending.put({"order_id": oid, "address": o.address})   # hand off, don't wait
    return orders[oid]

def worker():
    while True:
        job = pending.get()
        while True:
            try:
                httpx.post(f"{SHIPPING_URL}/shipments", json=job, timeout=2.0).raise_for_status()
                break
            except httpx.HTTPError:
                time.sleep(1)   # keep retrying; the original request already succeeded
```

```bash
curl -s -w "\nHTTP:%{http_code} time:%{time_total}\n" -X POST localhost:8013/orders \
  -H 'Content-Type: application/json' -d '{"item":"standing desk","address":"1600 Pennsylvania Ave"}'
# {"id":1,"item":"standing desk"}
# HTTP:200 time:0.015813

curl -s localhost:8013/queue-depth
# {"pending":0}
```

The request succeeded with a real `200` in ~16ms, even though Shipping was dead at that exact moment — the opposite
of the sync case. One genuinely surprising thing showed up here: `queue-depth` already read `0` *before* Shipping was
restarted. That's not a bug in this demo, it's a real, useful fact about Python's `queue.Queue`: `pending.get()`
removes the item from the queue the moment the worker thread picks it up, even while that worker is still stuck
inside the retry loop's `time.sleep(1)`. "Queue depth" as measured here tells you how many jobs haven't been *picked
up* yet, not how many are still in flight — a distinction that matters a lot once you're debugging a real backlog.
After restarting Shipping:

```bash
curl -s localhost:8011/shipments
# [{"order_id":1,"address":"1600 Pennsylvania Ave"}]
```

The shipment that was queued while Shipping was down arrived automatically once it came back, with no client retry
needed.

## How It Actually Works

**Synchronous (request/response)** communication means the caller's request does not complete until the callee
responds. This is simple to reason about — one function call, one return value, errors propagate immediately — and
that simplicity is real and valuable. Its cost is availability coupling: Order Service's effective uptime for
`POST /orders` becomes the *intersection* of its own uptime and Shipping's uptime, even though nothing about placing
an order logically requires shipping to happen in the same instant.

**Asynchronous (fire-and-forget, or message-queue-based)** communication means the caller hands off a unit of work
and returns without waiting for it to be fully processed. This decouples availability — Order Service stays up even
if Shipping is down — at the cost of a much harder question: *where does that queued work live, and does it survive
a crash of the process holding it?* The demo above uses an in-memory `queue.Queue`, which is intentionally fragile:
if `order_service_async.py` crashes before the worker drains the queue, that shipment request is gone forever, with
no record it ever existed. A production system needs the durable version of this pattern — the outbox pattern in
[Level 2, Lesson 5](../level-2/05-outbox-pattern-polling.md) persists the pending work to a real database table
*in the same transaction* as the order itself, so a crash can't lose it.

The deciding question for choosing between them, per call, is: **does the caller need the result right now to decide
what to do next?** Checking whether an item is in stock before confirming an order — synchronous, the caller needs
the answer immediately. Kicking off a shipping label print after an order is confirmed — asynchronous, the order's
success doesn't depend on it.

## Common mistakes

- Making every inter-service call asynchronous "for resilience," even when the caller genuinely cannot proceed
  without the answer (e.g., checking payment authorization before shipping) — this just moves the failure later and
  makes it harder to attribute.
- Making every inter-service call synchronous because it's simpler to write, then bolting on circuit breakers and
  retries (Level 3) to paper over an availability coupling that an async design would have avoided entirely.
- Using an in-memory queue (as this demo does, deliberately, to isolate the sync/async difference) in anything that
  needs to survive a restart — see the real crash-safety gap called out above, and its fix in Level 2.

## Exercise

Start the `order_service_async.py` + `shipping_service.py` pair yourself. Create an order, then immediately kill the
*Order* service process (not Shipping) with `kill -9` before the background worker has a chance to run. Restart Order
Service and check `GET /queue-depth`. Confirm for yourself that the queued shipment request is gone — then write one
sentence on what would need to change in the code for that request to survive the crash. Keep that answer; it's
exactly what [Level 2, Lesson 5](../level-2/05-outbox-pattern-polling.md) builds.
