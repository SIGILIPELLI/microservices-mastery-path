---
description: "A single checkout measured 4.1x slower with 8 sequential synchronous calls to a sibling service than with 1 batched call -- the real cost of chattiness."
---

# Anti-Patterns: the Distributed Monolith & Chatty Services

## What was run

A real `pricing_service.py` (port 8261) and a real client script, `chatty_checkout.py`, computing the exact same
checkout total two ways: eight separate, sequential, synchronous HTTP calls (one per cart item) versus one single
batched call. Both were timed with `time.perf_counter()` against the same live service.

## The real cost of chattiness, measured per request

```python
def chatty_checkout():
    total = 0
    for item_id in ITEMS:                       # 8 items
        r = c.get(f"{PRICING_URL}/price/{item_id}")   # 8 separate round trips, one per item
        total += r.json()["price_cents"]
    return total

def batched_checkout():
    r = c.post(f"{PRICING_URL}/price/batch", json=ITEMS)   # 1 round trip for all 8
    return sum(p["price_cents"] for p in r.json())
```

```
chatty checkout  (8 sequential round trips): 40.15ms, total=5332
batched checkout (1 round trip):              9.70ms, total=5332
single-request latency penalty from chattiness: 4.1x
```

Both return the identical total (5332 cents) — this is purely a latency cost, not a correctness difference. A
single user's checkout takes **4.1x longer** purely because the service boundary between checkout and pricing was
drawn in a way that needs eight sequential round trips to answer one logical question ("what's my cart's total?"),
instead of one. This is distinct from [Level 1, Lesson 2](../level-1/02-bounded-contexts-ddd.md)'s N+1 measurement
(which was about *throughput under load* from redundant lookups across many different requests) — this is the
*same single request's own latency*, inflated by how many hops its own logic needs.

## The second symptom: services that can't actually deploy independently

[Level 4, Lesson 4](04-shared-libraries-vs-duplication.md) already measured this directly: upgrading one shared
package (`shop-common`) silently changed both `orders_service.py`'s and `newsletter_service.py`'s real API
responses at the exact same moment, even though they're two separate deployables with no reason to otherwise
coordinate. That's the other defining symptom of a distributed monolith — not excessive chattiness, but **services
that are deployed separately on paper while actually being unable to change independently in practice**.

## How It Actually Works

A **distributed monolith** is a system split into multiple deployable services that, despite the split, behaves
like a monolith in all the ways that matter: changes still need to be coordinated across services, a single
logical operation still needs many of them to cooperate closely and synchronously (as measured above), and the
system has paid every real cost of being distributed (network latency, partial failure, operational complexity —
all covered across [Level 1](../level-1/04-sync-vs-async-communication.md) and
[Level 3](../level-3/01-circuit-breakers.md)) without gaining the benefits a real microservices split is supposed
to provide: independent deployability, independent scaling, and clear ownership boundaries.

The two symptoms demonstrated above are the two most common, concrete ways this shows up: **chattiness**
(measured here — a single operation needs many synchronous hops, each one compounding real latency, exactly the
problem [Level 1, Lesson 3](../level-1/03-service-boundaries-sizing.md)'s sizing heuristics are meant to prevent at
design time) and **deployment coupling** (measured in Lesson 4 — services that are nominally separate but can't
actually ship independent changes safely). Either symptom alone is a strong signal the service boundary is drawn
wrong, not that "microservices are inherently this way" — a correctly-sized boundary (per
[Level 1, Lesson 3](../level-1/03-service-boundaries-sizing.md)'s heuristics) should need far fewer synchronous hops
for its most common operations, and a correctly-isolated boundary (database-per-service from
[Level 2, Lesson 1](../level-2/01-database-per-service.md), no unnecessarily shared libraries) should allow real
independent deploys.

The fix for chattiness specifically, demonstrated directly above, is the same batch-endpoint approach
[Level 2, Lesson 2](../level-2/02-data-ownership.md) built for a different scenario: the *owning* service exposes
the access pattern its consumers actually need, rather than forcing consumers to assemble it themselves out of many
small calls.

## Common mistakes

- Treating "we have multiple services" as proof of a successful microservices migration, without checking whether
  any single common operation still requires many synchronous cross-service hops, or whether any two "independent"
  services actually have to be redeployed together to stay correct.
- Fixing chattiness by making the many calls asynchronous or parallel instead of reducing the call count — parallel
  calls reduce wall-clock latency somewhat, but still pay the overhead of N network round trips, N connection
  setups, and N places a partial failure can occur, compared to one well-designed batched call.
- Measuring only aggregate system throughput (requests/second) and never a single request's own latency breakdown —
  this lesson's 4.1x finding is specifically about one user's wait time, which an aggregate throughput number alone
  can hide.

## Exercise

Change `chatty_checkout.py` to use `httpx`'s async client and fire all 8 price lookups concurrently instead of
sequentially, then re-measure. Confirm the latency gap to the batched version narrows but doesn't disappear
entirely — and write down, from the real numbers you measure, how much of the original 4.1x penalty was pure
sequential-round-trip latency (fixed by concurrency) versus how much was the fixed per-call overhead of making 8
separate HTTP connections at all (not fixed by concurrency, only by batching).
