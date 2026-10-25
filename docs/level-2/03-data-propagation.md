---
description: "A real local cache serves stale data for 3 seconds after a live update, versus a live lookup that's instantly correct but always coupled to the dependency."
---

# Propagating Data: Sync Lookups vs Async Events

## What was run

A real `customers_service.py` (port 8061) and a real `orders_with_local_cache.py` (port 8062) that maintains an
in-memory local copy of one customer record, refreshed by a background thread every 3 seconds. The customer's name
was changed live, and both a cached read and a live read were taken immediately afterward, then the cached read was
taken again after waiting past the refresh interval.

## Two ways to read the same data, and the real gap between them

```python
# Cached path: zero network calls on the request path, served from local memory
@app.get("/orders/sample-with-cached-name")
def sample_cached():
    cached = local_cache.get(1, {"name": "<not cached yet>"})
    return {"customer_name_from_cache": cached.get("name")}

# Live path: one network call to Customers Service, every single request
@app.get("/orders/sample-with-live-lookup")
def sample_live():
    r = httpx.get(f"{CUSTOMERS_URL}/customers/1", timeout=2.0)
    return {"customer_name_live": r.json()["name"]}
```

```bash
curl -s -X PUT localhost:8061/customers/1/name -H 'Content-Type: application/json' -d '{"name":"Ada Lovelace-Byron"}'
# {"id":1,"name":"Ada Lovelace-Byron","updated_at":1792950751.01}

# immediately after:
curl -s localhost:8062/orders/sample-with-live-lookup
# {"order_item":"keyboard","customer_name_live":"Ada Lovelace-Byron"}   <- correct immediately

curl -s localhost:8062/orders/sample-with-cached-name
# {"order_item":"keyboard","customer_name_from_cache":"Ada Lovelace"}   <- STALE, the old name

# after waiting 4 seconds (past the 3-second refresh interval):
curl -s localhost:8062/orders/sample-with-cached-name
# {"order_item":"keyboard","customer_name_from_cache":"Ada Lovelace-Byron"}   <- now correct
```

This is a real, measured staleness window, not a theoretical one: for up to `POLL_INTERVAL` seconds (3, in this
demo) after a real update, the cached read serves data that is simply wrong. The live-lookup path never has this
problem — but it also never stops needing Customers Service to be reachable, which is exactly the coupling
[Lesson 1 of Level 1](../level-1/04-sync-vs-async-communication.md) measured as a real cost in the opposite
direction.

## How It Actually Works

Every time one service needs another service's data, there are really only two strategies, and this lesson's demo
makes both concrete:

- **Pull it live, every time** (the `sample-with-live-lookup` path): always correct as of the moment of the call,
  at the cost of a network round trip on every single request and a hard availability coupling to the data owner.
- **Keep a local copy, propagated some other way** (the `sample-with-cached-name` path): no network call on the
  request path, no availability coupling — but the local copy is, by construction, *always* at risk of being stale
  by however long it's been since the last successful propagation.

This demo propagates by polling on an interval, which is simple but wasteful (most polls find nothing changed) and
caps the staleness window at the poll interval, no tighter. The two more scalable alternatives, both of which trade
polling's waste for more infrastructure:

- **Event-driven propagation**: the owning service publishes an event the moment something actually changes
  ("customer 1's name changed"), and consumers update their local copy only when an event arrives — the staleness
  window becomes "however long event delivery takes," typically milliseconds, not a fixed poll interval. This is
  the natural next step once you have a real message broker, covered in depth in the
  [Kafka Mastery Path](https://sigilipelli.github.io/kafka-mastery-path/).
- **Change-data-capture**: tailing the owning service's database transaction log directly to produce the same kind
  of change events, without the owning service's application code needing to publish anything itself.

The deciding question, mirroring Level 1's sync/async lesson: **can this read tolerate being stale, and if so, by how
much?** A dashboard showing yesterday's sales totals can tolerate a lot of staleness. A page showing whether an item
is currently in stock, right before checkout, usually cannot — that calls for the live path, or an event-driven
propagation path with a staleness window measured in milliseconds, not seconds.

## Common mistakes

- Choosing a cache without ever stating, out loud, how stale is acceptable — "it's cached" with no bound on
  staleness is a silent correctness risk, not a performance optimization.
- Polling on a fixed interval when most polls find nothing changed, wasting both the network call and the owning
  service's capacity for no benefit — this is exactly what event-driven propagation is designed to fix.
- Forgetting that a local cache needs an answer for "what if this value was never cached yet" (the
  `"<not cached yet>"` fallback above) — a cache's cold-start behavior is as much a design decision as its
  refresh strategy.

## Exercise

Change `POLL_INTERVAL` to 1 second, repeat the name-update experiment, and measure the real staleness window with
timestamps printed on both sides (when the `PUT` was sent, when the cached value actually changed). Then write down,
for a feature you're familiar with (from this course or elsewhere), whether a 1-second staleness window would be
acceptable for it, and why.
