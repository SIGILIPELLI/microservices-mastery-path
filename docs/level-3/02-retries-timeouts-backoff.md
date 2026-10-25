---
description: "15 real clients retrying through a real 4-second outage: 465 requests with no backoff vs 83 with exponential backoff and jitter, measured directly."
---

# Retries, Timeouts & Backoff Done Right

## What was run

A real `target_service.py` (port 8121) that can simulate a timed outage and logs the real timestamp of every
request it actually receives. Fifteen real Python threads, each acting as an independent client, hammered it
through a real 4-second outage window using two different real retry strategies: a fixed 0.1-second interval with
no backoff, and exponential backoff with jitter starting at 0.1s and capped at 2.0s. The request counts below are
real, taken directly from the target service's own log — the same "trust the service's own log, not the client's
inference" lesson from the previous lesson, applied again.

## The retry storm, measured

```python
def naive_retry_client(client_id, results):
    while time.time() < deadline:
        try:
            r = http.get(TARGET)
            if r.status_code == 200: return
        except httpx.HTTPError:
            pass
        time.sleep(0.1)   # fixed interval, no backoff
```

```
=== NAIVE RETRY (fixed 0.1s interval, no backoff) ===
total real requests sent to target service: 465
clients that eventually succeeded: 15/15
attempts per client: max=31, sum=465
```

Fifteen clients, retrying every 0.1 seconds through a 4-second outage, sent **465 real requests** to a service that
was already struggling — every one of those requests, during the outage itself, was pure waste, hitting a service
that could not possibly serve it yet.

## The same outage, with backoff and jitter

```python
def backoff_retry_client(client_id, results):
    delay = 0.1
    while time.time() < deadline:
        try:
            r = http.get(TARGET)
            if r.status_code == 200: return
        except httpx.HTTPError:
            pass
        jitter = random.uniform(0, delay)
        time.sleep(delay + jitter)
        delay = min(delay * 2, 2.0)   # exponential, capped
```

```
=== EXPONENTIAL BACKOFF WITH JITTER ===
total real requests sent to target service: 83
clients that eventually succeeded: 8/15
attempts per client: max=6, sum=48
```

**83 requests instead of 465** — roughly a **5.6x reduction** in load on the recovering service, for the same real
outage. The honest, important caveat visible right in this number: only 8 of 15 clients succeeded within this test's
fixed cutoff window, not all 15. Backing off means some clients' next retry lands further in the future — a client
whose delay had already grown to, say, 1.6 seconds by the time the outage ended might not retry again until after
this test's cutoff, even though the service was already healthy again. This is the real, inherent tradeoff: less
load on the dependency, at the cost of some clients taking longer, individually, to notice recovery and succeed.

## How It Actually Works

Retrying a failed call is correct and necessary — transient failures (a brief network blip, a momentarily
overloaded instance) really do resolve on their own, and giving up immediately wastes a recoverable request. The
problem demonstrated above is specifically **synchronized, unthrottled retrying across many clients at once**: when
a dependency goes down, every client calling it fails at roughly the same time, and if every client then retries on
a fixed, short interval, the *retries themselves* become a sustained load spike on a service that is, by
definition, already having trouble — this is the "retry storm," and it can genuinely prevent a struggling service
from ever recovering, because it never gets a quiet moment to catch up.

**Exponential backoff** (doubling the wait between attempts, as shown, `0.1s → 0.2s → 0.4s → 0.8s → 1.6s → capped
at 2.0s`) spreads retries out over time instead of hammering at a fixed rate, which is most of the 5.6x reduction
measured above. **Jitter** (the `random.uniform(0, delay)` added on top) solves a second, subtler problem that
backoff alone doesn't: without jitter, many clients that failed at the same instant would retry at the same
computed delays, in near-synchronized waves — effectively a slower-motion version of the same retry storm, arriving
in bursts instead of continuously. Randomizing each client's actual wait time spreads those bursts out, which is
exactly why the demo includes it.

The caveat found above — not every client succeeds within a fixed observation window — is the real cost side of
this tradeoff, and it has a real-world answer: a well-designed backoff policy also sets a **maximum number of
retries or maximum total retry duration**, past which the caller gives up and surfaces a real failure to whatever
is above it, rather than retrying forever with an ever-growing delay. What that maximum should be is a product
decision (how long can a user reasonably wait?), not just an engineering one.

## Common mistakes

- Retrying with no backoff at all, "because the failure is probably transient" — exactly the 465-request storm
  measured above, which can make a real outage last longer, not shorter.
- Adding exponential backoff without jitter, which still allows large synchronized bursts of retries from clients
  that happened to fail at the same moment and compute the same delay sequence.
- Retrying indefinitely with no cap on either the delay or the total number of attempts — a caller that never gives
  up never surfaces a real, actionable failure to a human or a calling system, and can itself become a resource leak
  (threads, connections) that never resolves.
- Retrying a request that isn't idempotent (see [Level 2, Lesson 6](../level-2/06-idempotency-application-layer.md))
  without idempotency protection — retries and non-idempotent operations are a dangerous combination on their own,
  independent of backoff strategy.

## Exercise

Modify `backoff_retry_client` to also cap the total number of attempts at 5, and have it record (and print) a
clear `"gave up"` result for any client that exhausts its retries without success, instead of silently never
appearing in the results list (notice the current version has exactly this gap — 7 of the 15 clients above simply
vanish from the output with no indication of why). Re-run the full comparison and report how many clients now
explicitly "give up" versus how many succeed, for both strategies.
