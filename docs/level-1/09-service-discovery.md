---
description: "A real service registry with heartbeats, a gateway that stops hardcoding URLs, and a real stale-registration gap exposed by killing an instance."
---

# Service Discovery

## What was run

A real `registry.py` service (port 8019) that instances register with and heartbeat every 3 seconds, a real
`gateway_discovery.py` (port 8018) that looks up `catalog`'s current address instead of hardcoding it, and a real
`catalog_discoverable.py` instance moved from port 8020 to port 8021 mid-course, with the old instance actually
killed, to see exactly how the gateway reacts.

## Starting an instance, and watching it register itself

```python
def heartbeat():
    while True:
        httpx.post(f"{REGISTRY_URL}/register", json={"name": "catalog", "url": f"http://localhost:{PORT}"})
        time.sleep(3)
```

```bash
curl -s localhost:8019/instances
# {"catalog":{"url":"http://localhost:8020","last_heartbeat":1792950498.89}}

curl -s localhost:8018/items/1
# {"id":1,"title":"Desk Lamp","price_cents":2499}
```

The gateway never hardcodes `http://localhost:8020` — it asks the registry "where is `catalog` right now?" on every
request (`httpx.get(f"{REGISTRY_URL}/lookup/catalog")`), then calls whatever URL comes back.

## Killing the instance exposes a real gap

```bash
pkill -f "catalog_discoverable.py 8020"
curl -s -w "\nHTTP:%{http_code}\n" localhost:8018/items/1
# Internal Server Error
# HTTP:500
```

This `500` is real, and it reveals something this simple registry does *not* solve: the registry only learns an
instance is dead when its heartbeat stops arriving for `TTL_SECONDS` (10s here). In the gap between the kill and the
TTL expiry, `lookup/catalog` keeps returning the now-dead URL as if it were healthy, and the gateway's own
`httpx.get(catalog_url...)` call fails with an uncaught connection error, which FastAPI turns into a generic `500` —
the exact kind of swallowed, unhelpful error [Lesson 7](07-api-gateway-pattern.md) warned about.

Starting a *new* instance on a different port fixes it immediately, because registration is push-based, not
discovery-based from a fixed list:

```bash
python catalog_discoverable.py 8021   # new port, same service name
curl -s localhost:8019/instances
# {"catalog":{"url":"http://localhost:8021","last_heartbeat":1792950515.64}}
curl -s localhost:8018/items/1
# {"id":1,"title":"Desk Lamp","price_cents":2499}
```

No gateway code changed, no gateway restart happened — it picked up port 8021 automatically on the very next request,
because it never knew about 8020 in the first place; it only ever asked the registry.

## How It Actually Works

Hardcoding a downstream service's address (`CATALOG_URL = "http://localhost:8014"`, as every earlier lesson in this
level did) works fine with exactly one instance that never moves. It breaks down the moment you have more than one
instance of a service (for load distribution) or instances that come and go (deploys, autoscaling, crashes and
restarts) — which is normal, expected behavior in any real microservices deployment, not an edge case.

**Service discovery** solves this with one of two general approaches:

- **Client-side discovery** (what this demo built): each service instance registers itself with a shared registry on
  startup and re-registers (heartbeats) periodically; callers query the registry before each call, or cache the
  result briefly. This is roughly how tools like Consul or etcd-backed discovery work, and roughly what Kubernetes'
  own internal DNS-based service discovery abstracts away for you (a Kubernetes `Service` *is* a registry, with the
  kubelet handling the heartbeat-equivalent health checking).
- **Server-side discovery**: callers always talk to a fixed address (a load balancer), and the load balancer itself
  queries the registry and routes to a healthy instance. The caller never touches the registry directly.

The real gap this demo exposed — a TTL-based registry has a window where it confidently returns a dead address — is
not a bug unique to this toy implementation; it's the fundamental reason production discovery systems pair
registration with **active health checks** (the registry or load balancer itself pings each registered instance,
rather than only trusting self-reported heartbeats) and short TTLs, trading faster dead-instance detection against
more overhead. Kubernetes' readiness/liveness probes exist for exactly this reason.

## Common mistakes

- Trusting only self-reported heartbeats (as this demo does) in a system where an instance might hang rather than
  cleanly die — a hung process can keep its heartbeat thread alive while failing every real request.
- Setting the TTL too long "to reduce registry load," which widens the stale-address window demonstrated above; or
  too short, which can deregister a healthy instance during a brief GC pause or network blip.
- Building a custom registry like this one for production use instead of reaching for an existing, battle-tested
  system (Kubernetes Services, Consul, etcd) — this demo is deliberately minimal to show the mechanism, not something
  to actually deploy.

## Exercise

Add an active health check to `registry.py`: a background thread that, every 5 seconds, sends a real HTTP request to
each registered instance's `/items/1` endpoint and removes any registration that fails to respond, independent of
whether its heartbeat is still arriving on time. Kill a catalog instance with `kill -9` (not a graceful shutdown) and
measure, with real timestamps, how much faster this active check detects the dead instance compared to waiting for
the 10-second heartbeat TTL alone.
