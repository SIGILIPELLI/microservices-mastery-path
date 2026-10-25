---
description: "Microservices Level 1: when to split, bounded contexts, service sizing, sync/async communication, REST/gRPC contracts, API gateway, service discovery."
---

# Level 1 · Decomposition & Service Boundaries

Level 1 builds the judgment you need before you write a single service: when splitting actually pays off, where the
boundary should go, and how services actually talk to each other once they're split. Every example here is backed by
two real small services you'll start, call, and kill on your own laptop.

!!! note "What was run, and what wasn't"
    This course runs small FastAPI (Python) and gRPC services locally, started with `uvicorn`/`python` and called with
    `curl`, `httpx`, and generated gRPC clients. Every piece of terminal output in this level is real, observed output
    from those services. Where a lesson describes something only a full org-scale rollout would show (a real multi-team
    migration taking months), it says so plainly.

**Prerequisites:** basic programming and command-line comfort, and some exposure to REST APIs. Python 3.10+ is used for
the runnable examples; no prior distributed-systems experience is assumed.

## Modules

1. **[From Monolith to Microservices: Why and When to Split](01-monolith-to-microservices.md)**: the real costs and
   real benefits, not the slide-deck version.
2. **[Bounded Contexts & Domain-Driven Design Basics](02-bounded-contexts-ddd.md)**: finding seams in a domain before
   you find seams in code.
3. **[Sizing a Service: Boundaries That Actually Hold](03-service-boundaries-sizing.md)**: why "one service per noun"
   fails, and what holds up instead.
4. **[Inter-Service Communication: Synchronous vs Asynchronous](04-sync-vs-async-communication.md)**: the real
   latency and failure-mode tradeoff, demonstrated with a killed service.
5. **[REST Between Services: Contracts & Versioning](05-rest-contracts-versioning.md)**: designing a contract that
   survives independent deploys.
6. **[gRPC for Service-to-Service Calls](06-grpc-service-calls.md)**: a real `.proto` contract, generated clients,
   and when gRPC beats REST internally.
7. **[The API Gateway Pattern](07-api-gateway-pattern.md)**: one real gateway routing to two real backend services.
8. **[Backend-for-Frontend (BFF)](08-backend-for-frontend.md)**: why one gateway shape doesn't fit every client.
9. **[Service Discovery](09-service-discovery.md)**: how a caller finds a callee when instances come and go.
10. **[Project: Decompose a Monolith Into Two Real Services](10-project-decompose-monolith.md)**: a real runnable
    split, with a real seam and a real contract.

## After Level 1

Level 2 moves from "services that can talk" to "services that agree on data" — database-per-service, sagas, the
outbox pattern, and what eventual consistency actually feels like when you break it on purpose.
