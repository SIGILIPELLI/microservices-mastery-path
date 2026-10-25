---
description: "A real fast dependency call measured at 1.719s when sharing a connection pool with a hammered slow dependency, versus 0.006s with an isolated pool."
---

# Bulkheads & Resource Isolation

## What was run

Two real services: `slow_dependency.py` (port 8131, a real 2-second `time.sleep` per request) and
`fast_dependency.py` (port 8132, responds instantly). A real `caller.py` script measured how long a call to the
*fast* dependency takes while five threads are simultaneously hammering the *slow* one with 10 requests each —
first using one shared `httpx.Client` (one shared connection pool) for both dependencies, then using two separate
clients (one pool each). All timings are real, measured with `time.perf_counter()`.

## The problem: one slow dependency starves an unrelated fast one

```python
client = httpx.Client(limits=httpx.Limits(max_connections=5, max_keepalive_connections=5))
# five threads hammering SLOW_URL using this same client...
measure_fast_latency(client, "fast call WHILE slow dependency is hammered, shared pool")
```

```
baseline (no contention): fast dependency call took 0.010s
fast call WHILE slow dependency is hammered, shared pool: fast dependency call took 1.719s
```

The fast dependency itself never got slower — `fast_dependency.py` still responds in milliseconds, as the baseline
confirms. What got slower is **getting a connection to make the call at all**: with `max_connections=5` and five
threads already occupying all five connections talking to the slow dependency, the fast call had to wait in the
same pool's queue until one of those slow, 2-second requests freed up a connection. The fast dependency became
collateral damage of the slow one, through a resource (the connection pool) they had no business sharing.

## The fix: a separate pool per dependency

```python
slow_client = httpx.Client(limits=httpx.Limits(max_connections=5, max_keepalive_connections=5))
fast_client = httpx.Client(limits=httpx.Limits(max_connections=5, max_keepalive_connections=5))
# five threads hammering SLOW_URL using slow_client...
measure_fast_latency(fast_client, "fast call WHILE slow dependency is hammered, isolated pool")
```

```
fast call WHILE slow dependency is hammered, isolated pool: fast dependency call took 0.006s
```

Same hammering of the slow dependency, happening at the exact same time — but the fast call now has its own,
completely separate pool of connections, with nothing competing for it. **0.006s, indistinguishable from the
baseline with zero contention**, versus **1.719s** when the pool was shared — roughly a **280x** difference, caused
entirely by which resource pool each dependency's calls drew from.

## How It Actually Works

A **bulkhead** (named after a ship's bulkheads — compartments that keep one section flooding from sinking the whole
vessel) isolates a resource — a connection pool, a thread pool, a queue, a rate limit — per dependency, so that one
dependency behaving badly (slow, failing, saturated) cannot exhaust a resource that a different, unrelated
dependency also needs. This demo isolated an HTTP connection pool, but the same principle applies to thread pools (a
slow dependency's calls blocking all the worker threads a fast dependency's calls also need), database connection
pools (a slow query type exhausting the pool a fast query type also needs), and queues (a slow consumer backing up a
shared queue that a fast consumer also reads from).

This complements, rather than replaces, the circuit breaker from [Lesson 1](01-circuit-breakers.md): a circuit
breaker decides whether to *keep calling* a failing dependency at all; a bulkhead decides how much shared
*capacity* any one dependency — failing or just slow — is allowed to consume while it's being called. A circuit
breaker alone would not have helped this demo's scenario at all: the slow dependency wasn't failing, it was just
slow, and a slow-but-technically-successful dependency can starve shared resources just as effectively as a failing
one, without ever tripping a breaker that only watches for failures.

## Common mistakes

- Sizing one shared connection or thread pool for an entire service's *total* outbound call volume, without
  considering that different downstream dependencies have very different latency profiles — a single slow
  dependency can dominate a shared pool's occupancy even at a modest request rate, exactly as shown above.
- Isolating pools per dependency but sizing every pool identically regardless of that dependency's actual expected
  latency and call volume — a bulkhead's size should reflect how long calls to that specific dependency are
  expected to hold a connection, not a one-size-fits-all default.
- Adding bulkheads everywhere reflexively — every isolated pool is also a fixed resource commitment (a minimum
  number of connections/threads reserved whether they're busy or not); bulkheading a dependency that's never shown
  any sign of being slow or unreliable is pure overhead with no real benefit yet.

## Exercise

Change `slow_dependency.py`'s sleep to a random value between 0.5 and 3 seconds per request (still real, just more
realistic than a fixed delay), and change the hammering load from 5 threads × 10 requests to 20 threads × 5
requests. Re-run both scenarios and confirm the isolated-pool version's fast-dependency latency stays close to the
no-contention baseline regardless of how much load is thrown at the slow dependency — then find the smallest
`max_connections` value for the isolated fast-dependency pool that still keeps its own latency low under a
concurrent burst of 20 real simultaneous fast-dependency calls.
