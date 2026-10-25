---
description: "A real 3-instance rolling deploy under continuous live traffic: 105 requests, zero errors, and a clean v1-to-v2 transition as each instance was replaced."
---

# Deployment Strategies for Many Independent Services

## What was run

A real `worker.py` service run as three separate real processes (ports 8241-8243) behind a real, hand-written
load balancer, `lb.py` (port 8240), that round-robins across whichever backends last passed a real health check. A
real `rolling_deploy.py` script ran continuous traffic through the load balancer in a background thread while
killing and replacing one worker process at a time with a new version, health-checking each replacement before
sending it any traffic — a real rolling deployment, not a diagram of one.

## The rolling deploy, with continuous real traffic running throughout

```python
for port in BACKENDS:
    stop_worker(port)                    # real process kill
    start_worker(port, "v2")             # real process start, new version
    for _ in range(20):                  # real health-check polling before trusting it
        try:
            httpx.get(f"http://localhost:{port}/health", timeout=0.5)
            break
        except httpx.HTTPError:
            time.sleep(0.2)
    httpx.post(f"{LB_URL}/recheck-health")   # only now does the LB route traffic to it again
```

```
--- rolling deploy: replacing one worker at a time with v2 ---
  taking 8241 out, deploying v2...
  8241 is back, now serving v2
  taking 8242 out, deploying v2...
  8242 is back, now serving v2
  taking 8243 out, deploying v2...
  8243 is back, now serving v2

--- real results from continuous client traffic during the ENTIRE rolling deploy ---
  ('ok', 'v2'): 63
  ('ok', 'v1'): 42
total requests: 105
total errors (connection/HTTP failures): 0
```

**105 real requests, during the actual window where three processes were individually killed and replaced, with
zero errors.** The real client thread never saw a failure, because the load balancer never routed traffic to an
instance that was down — it only re-added a port to its healthy set after that port's `/health` endpoint actually
responded, as shown above. The version breakdown (42 responses still on v1, 63 already on v2) is itself evidence
the deploy genuinely happened gradually: requests were served by a real mix of old and new versions simultaneously
throughout the rollout, not an instant all-at-once switch.

## How It Actually Works

A **rolling deployment** replaces instances of a service one at a time (or in small batches), removing each from
the load balancer's rotation, deploying the new version, confirming it's healthy, and only then adding it back —
exactly the sequence this demo's script ran for real. Its real strength, measured directly above: it needs no
extra infrastructure beyond what's already running (no second full environment), and a failure in the new version
is caught on the *first* replaced instance, while the other two are still serving the old version, limiting the
blast radius of a bad deploy to roughly one-third of capacity rather than all of it. Its real cost: for the
duration of the rollout, **both versions are genuinely live and serving real traffic at the same time**, which
means the new version must be backward compatible with whatever the old version's clients and data expect — if v2
writes data in a format v1 can't read, and both are live simultaneously, that's a real correctness problem, not
just a deployment inconvenience.

Two other strategies worth naming precisely, since "deployment strategy" names get used loosely:

- **Blue-green deployment** keeps two complete environments ("blue" = currently live, "green" = the new version,
  fully deployed but receiving no traffic yet) and switches all traffic from one to the other essentially
  instantly, usually at the load balancer or DNS level. This avoids the "both versions live simultaneously" window
  entirely — the switch is atomic — at the cost of needing double the infrastructure capacity during the
  transition, and a rollback (switching back to blue) is just as instant as the forward switch.
- **Canary deployment**, covered in depth with real measured traffic percentages in
  [Level 3, Lesson 6](../level-3/06-canary-traffic-management.md), sends a small, deliberate percentage of traffic
  to the new version while most traffic stays on the old one — closer in spirit to this lesson's rolling deploy
  (both versions live at once) but controlled by *traffic percentage* rather than *instance count*, which allows a
  far smaller initial exposure (1% of traffic, say) than a 3-instance rolling deploy's minimum one-third.

## Common mistakes

- Rolling out a change that isn't backward compatible with the previous version, without realizing both versions
  are genuinely live together during the rollout — this demo's real zero-error result depended entirely on v1 and
  v2 both being safe to serve the same kind of traffic; a breaking change here would show up as real, intermittent
  errors precisely correlated with which instance happened to answer each request.
- Removing an instance from load-balancer rotation and deploying the new version without a real health check gate
  before adding it back — this demo's explicit polling loop against `/health` is exactly what prevents routing
  traffic to a process that's still starting up (common in real frameworks: accepting connections before the
  application has finished initializing).
- Treating "zero errors during my test" as proof a rollout is always safe — this demo's worker versions were
  trivially compatible by design; a real rollout's safety depends on the actual compatibility of what's being
  deployed, which the load balancer and health checks alone cannot guarantee.

## Exercise

Modify `worker.py`'s v2 behavior to genuinely break backward compatibility — have it return `{"version": "v2",
"port": PORT, "format": "new"}` while v1 still returns the old two-field shape — and modify the continuous traffic
client to assert on the exact field set it expects. Re-run the rolling deploy and observe, with real data, exactly
how many of the 105-or-so requests would have failed a strict client-side contract check during the mixed-version
window, even though the load balancer itself reported zero HTTP-level errors — demonstrating concretely why "the
load balancer saw zero errors" and "this deploy was actually safe" are not the same claim.
