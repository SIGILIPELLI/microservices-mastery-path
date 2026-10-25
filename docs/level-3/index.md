---
description: "Microservices Level 3: circuit breakers, retries/backoff, bulkheads, rate limiting, service mesh sidecar pattern, mTLS, distributed tracing."
---

# Level 3 · Resilience & Service Mesh

Level 3 is about what happens the moment one service in your system gets slow or stops responding — which, given
enough time and enough services, will happen. This level builds and deliberately breaks real failure-isolation
mechanisms, then shows how a service mesh moves a lot of that logic out of application code entirely.

!!! note "What was run, and what wasn't"
    This level runs real services with a real circuit breaker library, real injected latency and real killed
    processes to trigger failures, and a real lightweight sidecar proxy (Envoy, run standalone) to demonstrate the
    mesh pattern and mTLS. A full production mesh control plane (Istio/Linkerd's control plane, multi-cluster mesh)
    was not run as part of this course; where that's the case, it's stated plainly and the underlying mechanism is
    shown with the standalone proxy instead.

**Prerequisites:** Levels 1–2.

## Modules

1. **[Circuit Breakers: Real Failure Isolation](01-circuit-breakers.md)**: tripping a real breaker against a real
   failing dependency, and watching it half-open.
2. **[Retries, Timeouts & Backoff Done Right](02-retries-timeouts-backoff.md)**: a real retry storm, caused on
   purpose, and the jittered backoff that fixes it.
3. **[Bulkheads & Resource Isolation](03-bulkheads-resource-isolation.md)**: isolating one slow dependency's
   connection pool so it can't starve the rest of the service.
4. **[Rate Limiting Between Services](04-rate-limiting-between-services.md)**: a real token-bucket limiter protecting
   a downstream service from a real burst.
5. **[Service Mesh Fundamentals: The Sidecar Pattern](05-service-mesh-sidecar.md)**: a real standalone Envoy proxy
   sitting in front of a service, with traffic routed through it.
6. **[Traffic Management: Canary Releases at the Service Level](06-canary-traffic-management.md)**: splitting real
   traffic between two versions of a service at the proxy layer.
7. **[mTLS & Service-to-Service Security](07-mtls-service-security.md)**: real self-signed certificates, a real
   handshake, and what breaks when one side doesn't trust the other's cert.
8. **[Distributed Tracing Across Services](08-distributed-tracing.md)**: a real trace ID propagated across three
   real service calls, visualized end to end.
9. **[Consumer-Driven Contract Testing](09-consumer-driven-contracts.md)**: a contract test that catches a breaking
   API change before it ships.
10. **[Project: Inject Real Failures and Watch Resilience Work](10-project-inject-failures.md)**: a small chaos
    script that kills and slows real services while the system keeps serving most requests.

## After Level 3

Level 4 steps back from any one service to the whole system: how teams are organized around services, how you migrate
an existing monolith incrementally instead of in one risky rewrite, and when microservices are the wrong answer.
