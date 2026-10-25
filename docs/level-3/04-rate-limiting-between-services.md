---
description: "A real token-bucket limiter rejects exactly 20 of 30 simultaneous requests, protecting a downstream service from a real burst, per its own counter."
---

# Rate Limiting Between Services

## What was run

A real `protected_service.py` (port 8141) implementing a real token-bucket algorithm (capacity 10, refill rate 5
tokens/second) with its own internal counter of real work actually performed, separate from one unprotected
endpoint doing no limiting at all. A real burst of 30 simultaneous requests, from 30 real threads, was fired at
both endpoints.

## The token bucket, for real

```python
CAPACITY = 10
REFILL_RATE = 5  # tokens/second

def take_token():
    global tokens, last_refill
    with lock:
        elapsed = time.time() - last_refill
        tokens = min(CAPACITY, tokens + elapsed * REFILL_RATE)
        last_refill = time.time()
        if tokens >= 1:
            tokens -= 1
            return True
        return False

@app.get("/process-rate-limited")
def process_rate_limited():
    if not take_token():
        raise HTTPException(429, "rate limit exceeded, try again shortly")
    real_requests_received["count"] += 1
    return {"data": "processed"}
```

## The burst, with and without protection

```
=== NO RATE LIMITING (30 simultaneous requests) ===
client-observed: 30 accepted (200), 0 rejected (429)
service's own count of real work actually done: 30

=== TOKEN BUCKET RATE LIMITED (capacity=10, refill=5/s) (30 simultaneous requests) ===
client-observed: 10 accepted (200), 20 rejected (429)
service's own count of real work actually done: 10
```

With no limiting, all 30 requests hit real backend work simultaneously — fine for this toy endpoint, but exactly the
pattern that overwhelms a real downstream dependency (a database, a third-party API with its own rate limit, a
service with genuinely limited capacity) under a real traffic spike. With the token bucket in place, **exactly 10**
requests got through — matching the bucket's capacity precisely, confirmed independently by the service's own
`real_requests_processed` counter, not just the client-observed status codes — and the other 20 were rejected with a
real `429`, before any real backend work happened for them.

## How It Actually Works

A **token bucket** holds up to `CAPACITY` tokens, refilling at `REFILL_RATE` tokens per second; each request
consumes one token if available, and is rejected if the bucket is empty. This gives two independent, tunable
controls that matter for different reasons: **capacity** sets how large a sudden burst can be absorbed all at once
(here, 10 simultaneous requests succeed even though none of them "waited" for a refill), while **refill rate** sets
the sustained throughput the limiter allows over time (here, 5 requests/second indefinitely, once the initial burst
of 10 is spent). This is a meaningfully different shape of control than a simple "max N requests per second"
counter: a token bucket can absorb a legitimate short burst gracefully while still capping sustained load, which
matches how real traffic actually behaves — rarely perfectly smooth, often bursty.

The real value for inter-service communication specifically: this limiter protects the *service being called*,
regardless of how many different callers exist or how uncoordinated they are. [Lesson 2](02-retries-timeouts-backoff.md)'s
backoff and jitter reduce load *voluntarily*, from the caller's side, assuming the caller is well-behaved. Rate
limiting, placed on the *callee* side, enforces a hard cap regardless of caller behavior — a buggy client with no
backoff, a sudden legitimate spike in real users, or (in the inter-service case this lesson is really about) a
genuinely overloaded calling service all get the same protection, without requiring every caller to cooperate
correctly.

## Common mistakes

- Setting `REFILL_RATE` to match a dependency's absolute theoretical maximum throughput, leaving no margin — a
  downstream dependency usually needs some of its capacity for its *own* other callers or background work, not just
  the one caller being rate-limited.
- Rate limiting only at a single aggregate level when the real risk is one bad caller — a per-caller (or
  per-API-key) token bucket, rather than one global bucket shared by everyone, is often the more useful shape:
  one noisy client gets throttled without punishing every other well-behaved client sharing the same global bucket.
- Returning a bare `429` with no `Retry-After` header or guidance — a caller that doesn't know how long to wait
  before retrying is likely to retry immediately, recreating the exact retry-storm problem from Lesson 2.

## Exercise

Add a `Retry-After` response header to the `429` response, computed from how long until at least one token will be
available again (`(1 - tokens) / REFILL_RATE` seconds, roughly). Then write a small real client that reads this
header on a `429` and sleeps for exactly that long before retrying once, and confirm with a real `curl` or Python
script that its retry succeeds on the first attempt after waiting, rather than needing several blind retries.
