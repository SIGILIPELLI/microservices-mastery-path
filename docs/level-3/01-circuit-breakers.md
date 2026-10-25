---
description: "A real circuit breaker's state transitions, plus a genuine gap between its own error labels and what the downstream service actually logged."
---

# Circuit Breakers: Real Failure Isolation

## What was run

A real `flaky_service.py` (port 8111) that can be toggled to return real `500`s on demand, and a real client using
`pybreaker` (a real, widely-used Python circuit breaker library, not a hand-rolled one) with `fail_max=3,
reset_timeout=5`. The demo script, `run_breaker_demo.py`, drives the breaker through closed → open → half-open →
closed in one run. Every number below was cross-checked against `flaky_service`'s own request log, which is what
caught a real, honest surprise in the breaker library's own error reporting.

## The expected part: closed, then open

```python
breaker = pybreaker.CircuitBreaker(fail_max=3, reset_timeout=5)

@breaker
def call_flaky_service():
    r = httpx.get("http://localhost:8111/data", timeout=2.0)
    if r.status_code != 200:
        raise DownstreamError(f"got {r.status_code}")
    return r.json()
```

```
--- phase 2: service goes down (real 500s) ---
call 1: real call failed (got 500). breaker_state=closed
call 2: real call failed (got 500). breaker_state=closed
call 3: CIRCUIT OPEN, real call SKIPPED entirely. breaker_state=open
call 4: CIRCUIT OPEN, real call SKIPPED entirely. breaker_state=open
call 5: CIRCUIT OPEN, real call SKIPPED entirely. breaker_state=open
call 6: CIRCUIT OPEN, real call SKIPPED entirely. breaker_state=open
```

## The real surprise: checking the service's own log

```bash
grep "GET /data" flaky.log
# 200, 200,                          <- phase 1
# 500, 500, 500, 500,                 <- phase 2 + phase 3 combined
# 200, 200                           <- phase 4
```

That's **four** real `500`s, not the three "real call failed" lines phase 2 printed. Instrumenting the breaker's
internal failure counter directly (`breaker.fail_counter`) resolved it precisely: **"call 3,"** the one printed as
"CIRCUIT OPEN, real call SKIPPED entirely," was actually the exact call that *tripped* the breaker — pybreaker
executes the real HTTP request first, and only after that real request fails does it check whether the failure
count has now reached `fail_max` and, if so, re-wrap that same real failure as a `CircuitBreakerError` instead of
the original exception. The fourth real `500` showed up later still: phase 3's "probe call (service still down)"
*also* made a real network request — because `reset_timeout` (5s) had elapsed by then, pybreaker moved the breaker
to half-open and let exactly one real trial call through, which failed for real and was, again, reported as a
`CircuitBreakerError` indistinguishable from a truly skipped call by its exception type alone. Two calls that
*did* reach the network and two different calls that genuinely did *not* all produce the identical
`pybreaker.CircuitBreakerError` from the caller's point of view.

## How It Actually Works

A circuit breaker wraps calls to a dependency with a state machine: **closed** (calls go through normally; failures
are counted), **open** (calls are rejected immediately, with no network request, once failures cross a threshold),
and **half-open** (after a cooldown, exactly one real trial call is let through to test whether the dependency has
recovered — success closes the breaker again, failure reopens it). This demo exercised all three states for real.

The genuinely important lesson from the discrepancy above isn't about `pybreaker` specifically — it's that **a
library's own exception type is not always sufficient to tell you whether a real network call happened.** The call
that trips the breaker, and the periodic half-open probe, both necessarily make a real call (that's the whole point
— the breaker needs real evidence to decide), and a library focused on giving callers one consistent error type
(`CircuitBreakerError`) for "this call did not succeed, for breaker-related reasons" can reasonably choose not to
distinguish those two very different situations in the exception alone. If you need to know definitively whether a
specific call reached the network, the reliable source of truth is the downstream service's own logs or metrics —
exactly how this discrepancy was actually caught here — not inference from the client-side exception type.

This matters practically: if you're debugging "why does my dependency's request count not match my circuit
breaker's open/closed transitions," the answer is often exactly this — trip calls and half-open probes are real
calls wearing the same exception label as a fully-skipped one.

## Common mistakes

- Assuming every `CircuitBreakerError` (or equivalent) means zero load reached the downstream service — as shown
  above, the trip call and every half-open probe are real load, and a downstream service "protected" by a breaker
  still receives periodic real requests the whole time it's unhealthy.
- Setting `reset_timeout` far longer than a dependency's typical real recovery time, leaving the breaker open (and
  rejecting legitimate traffic) long after the dependency actually healed — the fix is probing, not just waiting
  blindly, which is exactly what half-open does, but only as often as `reset_timeout` allows.
- Wrapping a call with a circuit breaker but no fallback behavior for the open state, just propagating the
  `CircuitBreakerError` straight to the end user — a breaker's value is partly in enabling a graceful degraded
  response (cached data, a "try again shortly" message) instead of a generic error, which the calling code still
  has to implement.

## Exercise

Add a real counter to `flaky_service.py` that increments on every `GET /data` request, exposed via a new
`GET /request-count` endpoint. Re-run `run_breaker_demo.py` and use that counter — ground truth from the service
itself, not the client's inferred labels — to confirm exactly which of the six "phase 2" calls and the one "phase 3"
probe actually reached the network. Write down the real total and compare it to what the client's print statements
implied.
