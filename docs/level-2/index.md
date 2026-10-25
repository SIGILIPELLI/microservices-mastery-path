---
description: "Microservices Level 2: database-per-service, data ownership, sagas, the outbox pattern, idempotency, CQRS, eventual consistency."
---

# Level 2 · Data & Consistency Across Services

Level 2 is where microservices stop being a networking exercise and become a data-modeling one. Once each service owns
its own data, you lose the one thing a monolith gave you for free: a single transaction that touches everything. This
level shows, with a real database-backed saga, exactly what you do instead.

!!! note "What was run, and what wasn't"
    This level runs two real services, each with its own real SQLite database, coordinated through a real saga
    orchestrator service and a real outbox table polled on an interval. Every piece of output shown — including a
    deliberately failed step and its compensating transaction — is real, observed output. A full enterprise event bus
    (Kafka, RabbitMQ) is intentionally not required here; see the
    [Kafka Mastery Path](https://sigilipelli.github.io/kafka-mastery-path/) for that layer in depth.

**Prerequisites:** Level 1, plus basic SQL comfort.

## Modules

1. **[Database-per-Service & Why Shared Databases Break Down](01-database-per-service.md)**: the coupling a shared
   schema creates, shown with a real migration that breaks a second service.
2. **[Data Ownership & Avoiding Distributed Joins](02-data-ownership.md)**: who's allowed to write what, and how
   reads work without a join across services.
3. **[Propagating Data: Sync Lookups vs Async Events](03-data-propagation.md)**: keeping a local read-optimized copy
   versus calling out every time.
4. **[The Saga Pattern: Orchestration vs Choreography](04-saga-pattern.md)**: two working implementations of the same
   multi-step transaction.
5. **[The Outbox Pattern Without Kafka](05-outbox-pattern-polling.md)**: a real polling-based outbox that survives a
   crash between the write and the publish.
6. **[Idempotency at the Application Layer](06-idempotency-application-layer.md)**: making a retried request safe,
   demonstrated by firing the same request twice on purpose.
7. **[CQRS Across Service Boundaries](07-cqrs-across-services.md)**: separate read and write models, and why they
   drift on purpose.
8. **[Handling Partial Failures & Eventual Consistency](08-partial-failures-eventual-consistency.md)**: what a user
   actually sees during the inconsistency window, measured, not assumed.
9. **[Testing Data Consistency Across Services](09-testing-data-consistency.md)**: writing a test that actually
   catches a broken saga.
10. **[Project: A Real Saga With a Real Compensating Transaction](10-project-real-saga.md)**: an order-and-payment
    saga where the payment step is made to fail, on purpose, and the order gets rolled back for real.

## After Level 2

Level 3 assumes services will fail — not might, will — and builds the resilience patterns (circuit breakers,
bulkheads, service mesh) that keep one failing service from taking down everything that calls it.
