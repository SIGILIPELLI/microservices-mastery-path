---
description: "A real chaos script kills a dependency mid-traffic: the naive gateway loses A's data on 28 of 69 requests, the resilient one loses it on zero of 70."
---

# Project: Inject Real Failures and Watch Resilience Work

## What was run

A real `service_a.py` (port 8191, the critical dependency) and a real `service_b.py` (port 8192, a secondary
dependency), fronted by two real gateways built from the patterns in this level: `naive_gateway.py` (port 8193, no
resilience patterns at all) and `resilient_gateway.py` (port 8194, combining the circuit breaker from
[Lesson 1](01-circuit-breakers.md) and the bulkhead from [Lesson 3](03-bulkheads-resource-isolation.md)). A real
`chaos_test.py` script sent continuous real traffic to each gateway for 8 seconds while actually killing (`proc.kill()`,
a real `SIGKILL`) and later restarting Service B's process partway through — not a simulated failure, a real one.

## The naive gateway, under a real kill

```python
@app.get("/dashboard")
def dashboard():
    try:
        a = client.get("http://localhost:8191/a").json()
    except httpx.HTTPError:
        raise HTTPException(502, "service a unreachable")
    try:
        b = client.get("http://localhost:8192/b").json()   # one failure here kills the WHOLE response
    except httpx.HTTPError:
        raise HTTPException(502, "service b unreachable")
    return {"a": a, "b": b}
```

```
=== chaos test against naive gateway (http://localhost:8193) ===
[t+2.0s] killed service B for real
[t+5.0s] restarted service B for real
total requests sent: 69
dashboard returned 200: 41/69
requests where service A's data was still available to the user: 41/69
```

For **28 of 69 real requests**, a user hitting this dashboard got nothing at all — not even Service A's perfectly
healthy data — because the naive gateway treats any downstream failure as fatal to the entire response, even for a
dependency (B) that has nothing to do with A's own health.

## The resilient gateway, under the identical real kill

```python
client_a = httpx.Client(timeout=1.0)   # bulkhead: separate pool per dependency
client_b = httpx.Client(timeout=1.0)
b_breaker = pybreaker.CircuitBreaker(fail_max=3, reset_timeout=3)

@b_breaker
def call_b():
    return client_b.get("http://localhost:8192/b").json()

@app.get("/dashboard")
def dashboard():
    try:
        a = client_a.get("http://localhost:8191/a").json()
    except httpx.HTTPError:
        a = None
    try:
        b = call_b()
        degraded = False
    except (httpx.HTTPError, pybreaker.CircuitBreakerError):
        b = None             # degrade gracefully instead of failing the whole response
        degraded = True
    return {"a": a, "b": b, "degraded": degraded}
```

```
=== chaos test against resilient gateway (http://localhost:8194) ===
[t+2.0s] killed service B for real
[t+5.1s] restarted service B for real
total requests sent: 70
dashboard returned 200: 70/70
requests where service A's data was still available to the user: 70/70
```

**Every single one of 70 real requests** returned `200`, and every single one still included Service A's real
data — even during the window where Service B was genuinely, completely down. The difference between 41/69 and
70/70 is not a simulation or an approximation; it's the same real kill signal, sent to the same real process,
observed through two different gateway implementations handling the identical failure differently.

## How It Actually Works

This project is a direct, measured synthesis of two patterns from earlier in this level, and the combination is
what produces the result — neither alone would fully explain it:

- **Isolating the failure's blast radius** (the `try`/`except` structure that treats A and B independently, rather
  than letting B's exception propagate and abort the whole handler) is the single biggest contributor here: it's
  the direct fix for the naive gateway's core bug, that one dependency's failure takes down an entirely unrelated
  dependency's successful result.
- **The circuit breaker on `call_b`** adds a second real benefit beyond correctness: once it trips (after 3 real
  failures), subsequent calls during the outage are rejected immediately by pybreaker without even attempting a
  network call, which means the resilient gateway's `/dashboard` responses got *faster*, not just more complete,
  during the outage — each request no longer pays Service B's full 1-second timeout before giving up.

What makes this a genuine "watch resilience work" exercise rather than a reading exercise: the 28-request gap is a
directly observable consequence of eight total lines of difference between the two gateway implementations. No
mocking, no simulated failure injection framework — a real process was really killed while real traffic was really
flowing, and the measured outcome is the actual, concrete payoff of the patterns this level built one at a time.

## Common mistakes

- Building resilience patterns (breakers, bulkheads, retries) without a parallel "what should happen to the overall
  response if this one dependency fails" decision — the naive gateway's real bug wasn't a missing circuit breaker,
  it was the architectural choice to let any one failure abort everything, which no breaker alone would fix without
  also restructuring the failure handling as this project did.
- Testing resilience patterns only with simulated/mocked failures, never a real process kill — a real `SIGKILL`
  exercises real OS-level connection-refused behavior that a mocked `raise Exception()` in a unit test doesn't
  necessarily reproduce faithfully.
- Declaring a dependency "secondary" and degrading around it without the business actually agreeing it's acceptable
  to ship a dashboard missing that data — this is a product decision as much as an engineering one, and should be
  made explicitly, not assumed by whoever happens to be writing the gateway code.

## Exercise

Add a third dependency, `service_c.py`, that the resilient gateway also calls, and deliberately make it **not**
independently isolated — have it share `client_b`'s pool and have its failures propagate without a `try`/`except`
at all, as a regression. Re-run `chaos_test.py` against the resilient gateway while killing `service_c` instead of
`service_b`, and confirm you've reintroduced the exact failure mode this project fixed — proving that resilience
isn't a property of having *some* circuit breakers and bulkheads in a codebase, but of applying them consistently
to every dependency that needs them.
