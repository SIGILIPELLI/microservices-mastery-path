---
description: "A real Envoy weighted split measured at 88.8%/11.2% across 500 requests against a configured 90/10 target, with zero application code in the split."
---

# Traffic Management: Canary Releases at the Service Level

## What was run

Two real, trivially different services — `service_v1.py` (port 8161, returns `"v1-stable"`) and `service_v2.py`
(port 8162, returns `"v2-canary"`) — with a second real Envoy instance (port 10001) configured with a
`weighted_clusters` route sending 90% of traffic to v1 and 10% to v2. 500 real HTTP requests were sent through the
proxy and each response's reported version was counted.

## The weighted route, and the measured real split

```yaml
route:
  weighted_clusters:
    clusters:
      - name: service_v1_cluster
        weight: 90
      - name: service_v2_cluster
        weight: 10
    total_weight: 100
```

```python
for _ in range(500):
    r = httpx.get('http://localhost:10001/version')
    counts[r.json()['version']] += 1
```

```
v1-stable: 444/500 = 88.8%
v2-canary: 56/500 = 11.2%
```

Across 500 real requests, Envoy's weighted routing landed at **88.8%/11.2%**, close to but not exactly the
configured 90/10 — a real, expected amount of sampling variance at this sample size, not a configuration error (the
law of large numbers, not a bug; a much larger sample would converge closer to exactly 90/10). Neither
`service_v1.py` nor `service_v2.py` contains a single line of routing logic — each only knows how to answer its own
`/version` endpoint; the entire decision of *which* service receives a given request lives in Envoy's configuration.

## How It Actually Works

A **canary release** sends a small percentage of real production traffic to a new version of a service while the
rest continues to the proven, stable version — letting you observe the new version's real behavior under real
traffic, with real users, at a scale small enough that a problem affects few people and is cheap to roll back,
before committing to it fully. This is meaningfully different from testing in a staging environment: a canary sees
*actual* production traffic patterns, actual production data shapes, and actual production load — things a
pre-production environment can approximate but never perfectly replicate.

Implementing this at the **service mesh / proxy layer**, as this demo does, rather than in application code, has a
specific advantage directly connected to [Lesson 5](05-service-mesh-sidecar.md): the percentage split is pure
configuration, changeable without redeploying either service. A rollout in practice usually starts at a small
percentage (1-5%), watches real error rates and latency on the canary cluster specifically (something the proxy
layer can report independently per-cluster, since it already tracks which cluster handled which request), and
ramps the weight up gradually — 5% → 25% → 50% → 100% — rolling back to 0% immediately if the canary's real metrics
look worse than the stable version's, with no code deploy needed to make that change, only a config push.

The alternative, feature flags evaluated inside application code, puts this same decision in each service's own
logic instead of the network layer — workable, but it means every service reimplements percentage-based rollout
logic itself, and the decision isn't visible or controllable from one central place the way a mesh's routing config
is.

## Common mistakes

- Treating a small sample's measured percentage (88.8% here, not exactly 90%) as evidence the configuration is
  wrong — always check with a large enough sample, or trust the configured weight directly, before assuming
  something is broken.
- Canarying a change with no plan for what metric decides success or failure — a canary without a clear, specific
  "if X gets worse by Y%, roll back" rule just delays the decision rather than making it safer.
- Forgetting that a canary needs genuinely comparable traffic to be meaningful — if the 10% isn't a representative
  random sample (e.g., it's always the same specific users, or always low-traffic hours), the canary's observed
  behavior may not predict what happens at 100%.

## Exercise

Change the weights to 50/50, restart Envoy, and re-run the 500-request measurement — confirm the real observed
split moves close to 50/50, proportionally. Then write a small script that also records each response's latency,
and compute the average latency for v1 versus v2 separately from the same 500-request run, simulating how you'd
monitor a real canary's performance against the stable version's, side by side, from the same traffic sample.
