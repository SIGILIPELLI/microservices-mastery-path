---
description: "A real Envoy proxy running as a sidecar in front of a real backend, adding real headers and a real access log line, with no application code changed."
---

# Service Mesh Fundamentals: The Sidecar Pattern

## What was run

A real, standalone Envoy proxy (version 1.40.1, installed via Homebrew, not a container or mock) configured as a
sidecar in front of a real `backend_service.py` (port 8151). Envoy listened on port 10000 and proxied to the
backend on port 8151, adding request headers and writing a real access log line for every request. The exact same
endpoint was called twice: once directly, once through the proxy, with the backend itself reporting back which
headers it actually received.

## Direct call vs. through the sidecar

```bash
curl -s localhost:8151/hello   # straight to the backend, bypassing Envoy entirely
# {"message":"hello from the real backend","headers_seen":{"host":"localhost:8151"}}

curl -s localhost:10000/hello  # through the Envoy sidecar
# {"message":"hello from the real backend","headers_seen":{
#   "host":"localhost:10000",
#   "x-forwarded-proto":"http",
#   "x-request-id":"a0a5a79f-71a1-4406-8424-766278539483",
#   "x-envoy-expected-rq-timeout-ms":"15000",
#   "x-envoy-sidecar":"true",
#   "x-forwarded-by":"envoy-sidecar-demo"
# }}
```

`backend_service.py`'s code never changed between these two calls. The second call arrived with real headers the
backend never set itself: a generated `x-request-id` (useful for [distributed tracing](08-distributed-tracing.md),
covered later in this level), `x-envoy-expected-rq-timeout-ms` (Envoy's own bookkeeping), and the two custom headers
this demo's Envoy config explicitly adds (`x-envoy-sidecar`, `x-forwarded-by`) via a plain `request_headers_to_add`
rule — no code, just configuration.

```bash
tail -1 envoy.log
# ENVOY_ACCESS_LOG method=GET path=/hello response_code=200 upstream=127.0.0.1:8151
```

This log line was produced entirely by Envoy, from its own `access_log` configuration — `backend_service.py` has no
logging code for this request at all. Every request that goes through the sidecar gets this observability for free.

## How It Actually Works

A **sidecar** is a separate process (here, a standalone Envoy process; in Kubernetes, a second container in the
same pod) that sits directly in front of (or beside) a service and intercepts its network traffic, adding
cross-cutting behavior — routing, retries, timeouts, mTLS, observability, traffic shaping — without that behavior
living in the service's own application code. This demo's `envoy.yaml` defines a `listener` (where traffic arrives,
port 10000), a `cluster` (where it actually goes, the real backend at 127.0.0.1:8151), and a route that both sends
the traffic there and adds headers along the way — all declarative configuration, zero lines of Python changed in
`backend_service.py`.

A **service mesh** is what you get when every service in a system has its own sidecar, and those sidecars are
managed centrally by a control plane (Istio's istiod, Linkerd's control plane) that pushes configuration to every
sidecar at once — "every service's traffic should now use mTLS," or "route 10% of traffic for service X to the new
version," applied everywhere without touching any service's code or redeploying any service. This demo deliberately
stops short of that: it's one standalone Envoy instance with a static config file, which shows the sidecar
*mechanism* clearly without needing a full mesh control plane running locally — genuinely out of scope for what a
single laptop can reasonably run, and said plainly here rather than faked.

The real architectural point this demo makes concrete: **cross-cutting concerns move from inside every service's
code to configuration in front of every service**, uniformly. Without a mesh, giving every service real timeouts,
retries, and mTLS means every team re-implementing those correctly in their own language and framework (see
[Lesson 2](02-retries-timeouts-backoff.md) and [Lesson 1](01-circuit-breakers.md) for how much real logic that is).
With a mesh, that logic lives once, in the proxy layer, and applies uniformly regardless of what language or
framework any individual service happens to use.

## Common mistakes

- Treating the sidecar as a replacement for application-level resilience logic entirely — a sidecar can enforce
  timeouts and retries at the network level, but business-logic-aware decisions (what counts as a successful
  response for *this specific* operation, what a circuit breaker should fall back to) often still belong in the
  application.
- Underestimating the real latency and resource cost of adding a proxy hop to every single request — a sidecar
  adds a real, if usually small, amount of overhead to every call, which matters at very high request rates or very
  tight latency budgets.
- Assuming a mesh is "free" observability and security — someone still has to configure, upgrade, and debug the
  mesh's control plane itself, which is real operational surface, not a zero-cost abstraction.

## Exercise

Add a second route to `envoy.yaml` that matches `prefix: "/slow"` and routes to a *different* upstream cluster
pointing at the `slow_dependency.py` service from [Lesson 3](03-bulkheads-resource-isolation.md) (port 8131, if
still running, or restart it). Confirm with `curl localhost:10000/slow` that one Envoy instance can route different
paths to entirely different real backend services based on the request path alone — the same routing mechanism a
real API Gateway (back in [Level 1, Lesson 7](../level-1/07-api-gateway-pattern.md)) implements in application
code, now implemented as pure configuration instead.
