---
title: "Microservices Interview Questions and Answers (179 Questions)"
description: "179 Microservices interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Microservices Mastery Path."
---

# Microservices Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Microservices Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Decomposition & Service Boundaries

### [From Monolith to Microservices: Why and When to Split](level-1/01-monolith-to-microservices.md)

**Q1.** What is a key reason to move from a monolith to microservices?

**Answer:** Independent deployability and scaling of parts owned by different teams

It costs complexity.

**Q2.** What new problem do microservices introduce?

**Answer:** Distributed-system failure modes: network calls, partial failure and data consistency

Plan for it.

**Q3.** What is a modular monolith?

**Answer:** One deployable with strong internal module boundaries

Often the right start.

**Q4.** When should you NOT split?

**Answer:** When the team is small and the domain not yet understood

Premature splitting hurts.

### [Bounded Contexts &amp; Domain-Driven Design Basics](level-1/02-bounded-contexts-ddd.md)

**Q5.** What is a bounded context?

**Answer:** A boundary within which a domain model and its language are consistent

DDD.

**Q6.** What is ubiquitous language?

**Answer:** Shared domain terms used by developers and business inside a context

Reduces translation.

**Q7.** The word 'Customer' means different things to Sales and Billing. What does DDD suggest?

**Answer:** Separate bounded contexts with explicit translation between them

Context mapping.

**Q8.** What is an anti-corruption layer?

**Answer:** A translation layer protecting a model from another context's concepts

Integration.

### [Sizing a Service: Boundaries That Actually Hold](level-1/03-service-boundaries-sizing.md)

**Q9.** What makes a good service boundary?

**Answer:** High cohesion, low coupling and aligned with a business capability

Not a layer.

**Q10.** What's wrong with a 'nano-service' per function?

**Answer:** Excess network chatter and operational overhead

Right-size.

**Q11.** What's a smell that boundaries are wrong?

**Answer:** Changes constantly require coordinated releases of several services

Distributed monolith.

**Q12.** What guides sizing?

**Answer:** Team ownership, change cadence and data cohesion

Not a fixed size.

### [Inter-Service Communication: Synchronous vs Asynchronous](level-1/04-sync-vs-async-communication.md)

**Q13.** When prefer asynchronous messaging?

**Answer:** When callers needn't wait and you want decoupling and resilience

Events and commands.

**Q14.** What's the cost of synchronous chains?

**Answer:** Latency adds up and availability multiplies down

Temporal coupling.

**Q15.** Three services in a synchronous chain are each 99.9% available. What's the chain's availability?

```python
print(round(0.999**3*100,3))
```

**Answer:** 99.7

Availabilities multiply. Verified by running it.

**Q16.** What's temporal coupling?

**Answer:** Services must be up at the same time to work

Async reduces it.

### [REST Between Services: Contracts &amp; Versioning](level-1/05-rest-contracts-versioning.md)

**Q17.** What makes a REST API change backward compatible?

**Answer:** Adding optional fields/endpoints without changing existing behavior

Be liberal in what you accept.

**Q18.** How version breaking changes?

**Answer:** New version (URL or media type) while supporting old consumers during migration

Deprecation policy.

**Q19.** What does an OpenAPI contract provide?

**Answer:** A machine-readable API description for docs, mocks and tests

Contract-first.

**Q20.** What's Postel's law in APIs?

**Answer:** Be conservative in what you send, liberal in what you accept

Tolerant readers.

### [gRPC for Service-to-Service Calls](level-1/06-grpc-service-calls.md)

**Q21.** What does gRPC use?

**Answer:** HTTP/2 with Protocol Buffers and generated clients

Typed contracts.

**Q22.** Why is gRPC good between services?

**Answer:** Efficient binary encoding, streaming and strict schemas

Internal APIs.

**Q23.** What must clients set on gRPC calls?

**Answer:** Deadlines (timeouts)

Avoid hangs.

**Q24.** How evolve protobuf safely?

**Answer:** Add fields with new numbers; never reuse or renumber

Compatibility.

### [The API Gateway Pattern](level-1/07-api-gateway-pattern.md)

**Q25.** What does an API gateway do?

**Answer:** Single entry point for routing, auth, rate limiting and aggregation

Edge concerns.

**Q26.** What's a gateway anti-pattern?

**Answer:** Putting business logic in the gateway

Keep it thin.

**Q27.** What does the gateway centralize?

**Answer:** Cross-cutting concerns like authentication and throttling

Consistency.

**Q28.** What's its risk?

**Answer:** A single point of failure and bottleneck if not scaled/redundant

HA.

### [Backend-for-Frontend (BFF)](level-1/08-backend-for-frontend.md)

**Q29.** What is a Backend for Frontend?

**Answer:** A dedicated API layer tailored to each client type (web, mobile)

Per-client needs.

**Q30.** Why not one generic API for all clients?

**Answer:** Different clients need different shapes, causing over/under-fetching

BFF fixes.

**Q31.** Who should own a BFF?

**Answer:** The frontend team that uses it

Autonomy.

**Q32.** What should a BFF avoid?

**Answer:** Duplicating business rules from domain services

Orchestrate, don't own.

### [Service Discovery](level-1/09-service-discovery.md)

**Q33.** Why service discovery?

**Answer:** Instances come and go, so clients need current addresses

Dynamic.

**Q34.** Client-side vs server-side discovery?

**Answer:** Client queries a registry and picks; server-side uses a load balancer/router

Trade-offs.

**Q35.** What does Kubernetes Service/DNS provide?

**Answer:** Stable names and load balancing over pod endpoints

Built-in.

**Q36.** What must registries handle?

**Answer:** Health checks and deregistration of dead instances

Stale entries.

### [Project: Decompose a Monolith Into Two Real Services](level-1/10-project-decompose-monolith.md)

**Q37.** What does the decompose-a-monolith project practice?

**Answer:** Identifying boundaries and extracting a service incrementally

Strangler.

**Q38.** What's the first extraction candidate?

**Answer:** A well-bounded capability with few dependencies and clear value

Low risk.

**Q39.** How keep the system working during extraction?

**Answer:** Route traffic gradually and keep the old path as a fallback

Safe migration.

**Q40.** What do you measure?

**Answer:** Deploy frequency, failures and latency before and after

Evidence.

## Level 2 · Data & Consistency Across Services

### [Database-per-Service &amp; Why Shared Databases Break Down](level-2/01-database-per-service.md)

**Q41.** What does database-per-service mean?

**Answer:** Each service owns its data store; others access it only via the API

Autonomy.

**Q42.** What's the main problem with a shared database?

**Answer:** Hidden coupling: schema changes break other services

Distributed monolith.

**Q43.** Can services use the same DB server?

**Answer:** Yes if schemas/credentials are isolated and ownership is clear

Logical separation.

**Q44.** How do other services get your data?

**Answer:** Through your API or by consuming your published events

Contracts.

### [Data Ownership &amp; Avoiding Distributed Joins](level-2/02-data-ownership.md)

**Q45.** What does data ownership mean?

**Answer:** One service is the source of truth for each piece of data

Single writer.

**Q46.** How handle a query needing data from three services?

**Answer:** API composition, a read model or replicated data via events

No cross-DB joins.

**Q47.** What's a source of truth?

**Answer:** The authoritative record of a fact

Others hold copies.

**Q48.** What's an owner's responsibility?

**Answer:** Schema, quality, access and evolution of its data

Accountability.

### [Propagating Data: Sync Lookups vs Async Events](level-2/03-data-propagation.md)

**Q49.** How propagate changes to other services?

**Answer:** Publish domain events that interested services consume

Event-driven.

**Q50.** What's the difference between event notification and event-carried state transfer?

**Answer:** Notification says something changed; state transfer includes the data consumers need

Trade coupling vs chattiness.

**Q51.** What does a consumer's local copy risk?

**Answer:** Staleness (eventual consistency)

Design for it.

**Q52.** What identifies duplicate events?

**Answer:** Event IDs checked by idempotent consumers

At-least-once.

### [The Saga Pattern: Orchestration vs Choreography](level-2/04-saga-pattern.md)

**Q53.** What is a saga?

**Answer:** A sequence of local transactions with compensating actions for failure

Distributed workflow.

**Q54.** What's a compensating transaction?

**Answer:** An action that semantically undoes an earlier step (e.g. refund)

No distributed rollback.

**Q55.** Orchestration vs choreography?

**Answer:** Orchestration uses a coordinator; choreography has services react to events

Trade-offs.

**Q56.** Why not two-phase commit across services?

**Answer:** It blocks, couples availability and doesn't scale

Sagas instead.

### [The Outbox Pattern Without Kafka](level-2/05-outbox-pattern-polling.md)

**Q57.** What problem does the outbox pattern solve?

**Answer:** Atomically updating state and publishing an event (dual-write problem)

Same transaction.

**Q58.** How does the outbox work?

**Answer:** Write the event to an outbox table in the same transaction, then a relay publishes it

Polling or CDC.

**Q59.** What delivery guarantee results?

**Answer:** At-least-once, so consumers must be idempotent

Dedupe.

**Q60.** What cleans the outbox?

**Answer:** Deleting or marking rows after publish

Housekeeping.

### [Idempotency at the Application Layer](level-2/06-idempotency-application-layer.md)

**Q61.** What is idempotency?

**Answer:** Repeating an operation has the same effect as doing it once

Safe retries.

**Q62.** How implement idempotent POSTs?

**Answer:** Client-supplied idempotency keys stored with the result

Return the original.

**Q63.** Why do retries need it?

**Answer:** A timeout may hide a success, so a retry could double-apply

Payments.

**Q64.** Where store processed IDs?

**Answer:** In the same transaction as the side effect

Atomic dedupe.

### [CQRS Across Service Boundaries](level-2/07-cqrs-across-services.md)

**Q65.** What does CQRS separate?

**Answer:** The write model from one or more read models

Optimized reads.

**Q66.** How are read models updated across services?

**Answer:** By consuming events and building projections

Eventually consistent.

**Q67.** When is CQRS overkill?

**Answer:** Simple CRUD with similar read and write needs

Use judiciously.

**Q68.** What's the cost?

**Answer:** Complexity and eventual consistency

Trade-off.

### [Handling Partial Failures &amp; Eventual Consistency](level-2/08-partial-failures-eventual-consistency.md)

**Q69.** What's a partial failure?

**Answer:** Some operations succeed while others fail in a distributed call

Normal in distributed systems.

**Q70.** How design UX for eventual consistency?

**Answer:** Show pending states and reconcile when updates arrive

Honest UI.

**Q71.** What does 'read your writes' need?

**Answer:** Routing a user's reads to fresh data or returning the write result

Consistency guarantee.

**Q72.** What handles stuck workflows?

**Answer:** Timeouts, retries and manual intervention queues

Operational.

### [Testing Data Consistency Across Services](level-2/09-testing-data-consistency.md)

**Q73.** How test eventual consistency?

**Answer:** Assert eventually with polling/timeouts rather than immediately

Await conditions.

**Q74.** What should consistency tests cover?

**Answer:** Duplicates, reordering, delays and failures of messages

Chaos.

**Q75.** What's reconciliation?

**Answer:** Periodically comparing sources of truth with copies to repair drift

Safety net.

**Q76.** Why contract tests for events?

**Answer:** Consumers depend on event schemas

Schema registry.

### [Project: A Real Saga With a Real Compensating Transaction](level-2/10-project-real-saga.md)

**Q77.** What does the real saga project build?

**Answer:** An order workflow across services with compensations

Level 2 project.

**Q78.** What triggers compensation?

**Answer:** A failed step such as payment declined after inventory reserved

Undo reservations.

**Q79.** How track saga state?

**Answer:** Persisted saga state machine (orchestrator) or event history

Recoverable.

**Q80.** What proves correctness?

**Answer:** Tests for every failure path and idempotent steps

Cover compensations.

## Level 3 · Resilience & Service Mesh

### [Circuit Breakers: Real Failure Isolation](level-3/01-circuit-breakers.md)

**Q81.** What does a circuit breaker do?

**Answer:** Stops calling a failing dependency for a while to let it recover

Fail fast.

**Q82.** What are its states?

**Answer:** Closed, open and half-open

Half-open probes.

**Q83.** Why fail fast?

**Answer:** Prevent resource exhaustion and cascading failure

Protects callers.

**Q84.** What fallback might you provide?

**Answer:** Cached/default data or a degraded response

Graceful degradation.

### [Retries, Timeouts &amp; Backoff Done Right](level-3/02-retries-timeouts-backoff.md)

**Q85.** Why set timeouts on every remote call?

**Answer:** Without them threads hang and failures cascade

Always.

**Q86.** What is exponential backoff with jitter?

**Answer:** Increasing waits with randomness to avoid synchronized retries

Avoid thundering herd.

**Q87.** When must you NOT retry blindly?

**Answer:** Non-idempotent operations without idempotency keys

Duplicates.

**Q88.** What is a retry budget?

**Answer:** A cap on retry volume to avoid retry storms

Limit amplification.

### [Bulkheads &amp; Resource Isolation](level-3/03-bulkheads-resource-isolation.md)

**Q89.** What's a bulkhead?

**Answer:** Isolating resources (pools/threads) so one failure doesn't sink the rest

Ship compartments.

**Q90.** How implement in a service?

**Answer:** Separate thread/connection pools or concurrency limits per dependency

Isolation.

**Q91.** What does it prevent?

**Answer:** A slow dependency exhausting shared resources

Contain blast radius.

**Q92.** What else isolates?

**Answer:** Separate deployments/instances for critical vs batch traffic

Tiering.

### [Rate Limiting Between Services](level-3/04-rate-limiting-between-services.md)

**Q93.** Why rate limit between services?

**Answer:** Protect a service from overload by noisy callers

Fairness.

**Q94.** What's a token bucket?

**Answer:** A rate limiter allowing bursts up to bucket size with a steady refill

Common algorithm.

**Q95.** What should limited callers get?

**Answer:** 429 Too Many Requests with Retry-After

Clear signal.

**Q96.** Where enforce?

**Answer:** At the callee, gateway or mesh with shared counters

Distributed.

### [Service Mesh Fundamentals: The Sidecar Pattern](level-3/05-service-mesh-sidecar.md)

**Q97.** What does a service mesh sidecar do?

**Answer:** Handles networking (mTLS, retries, telemetry) outside application code

Envoy/Istio.

**Q98.** What's the control plane for?

**Answer:** Configuring sidecars and distributing policy and certificates

Management.

**Q99.** What's a cost of a mesh?

**Answer:** Operational complexity and added latency/resources

Adopt when needed.

**Q100.** What does the mesh give uniformly?

**Answer:** Cross-cutting behavior across languages

Consistency.

### [Traffic Management: Canary Releases at the Service Level](level-3/06-canary-traffic-management.md)

**Q101.** What is a canary release?

**Answer:** Routing a small share of traffic to the new version before full rollout

Reduce risk.

**Q102.** What drives promotion?

**Answer:** Automated analysis of error rate and latency against the baseline

Metrics.

**Q103.** What's weighted routing?

**Answer:** Splitting traffic by percentages between versions

Mesh feature.

**Q104.** What's the rollback trigger?

**Answer:** SLO violations during the canary

Automatic.

### [mTLS &amp; Service-to-Service Security](level-3/07-mtls-service-security.md)

**Q105.** What does mTLS provide?

**Answer:** Mutual authentication and encryption between services

Zero trust.

**Q106.** How are certificates managed in a mesh?

**Answer:** Automatically issued and rotated by the control plane

Short-lived.

**Q107.** What is zero trust here?

**Answer:** Never trust network location; authenticate and authorize every call

Least privilege.

**Q108.** What adds authorization?

**Answer:** Policies on service identity (who may call what)

SPIFFE ids.

### [Distributed Tracing Across Services](level-3/08-distributed-tracing.md)

**Q109.** What's distributed tracing for?

**Answer:** Following one request across services to find latency and errors

Spans.

**Q110.** What must propagate between services?

**Answer:** Trace context headers (traceparent)

W3C.

**Q111.** What's a span?

**Answer:** A timed operation within a trace

Parent-child.

**Q112.** What's sampling?

**Answer:** Recording only a fraction of traces to control cost

Tail sampling.

### [Consumer-Driven Contract Testing](level-3/09-consumer-driven-contracts.md)

**Q113.** What problem does consumer-driven contract testing solve?

**Answer:** Providers learn how consumers actually use their API before changing it

Pact.

**Q114.** Who writes the contract?

**Answer:** The consumer's expectations

CDC.

**Q115.** What do providers verify?

**Answer:** That their implementation satisfies all consumer contracts in CI

Can-I-deploy.

**Q116.** What does it replace?

**Answer:** Slow, flaky end-to-end integration tests for interface checks

Faster feedback.

### [Project: Inject Real Failures and Watch Resilience Work](level-3/10-project-inject-failures.md)

**Q117.** What does failure injection (chaos) test?

**Answer:** That resilience patterns work under realistic faults

Prove it.

**Q118.** What faults to inject?

**Answer:** Latency, errors, dependency outages and resource exhaustion

Real failures.

**Q119.** What should precede chaos in prod?

**Answer:** Observability, rollback and a limited blast radius

Safety.

**Q120.** What do you learn?

**Answer:** Whether timeouts, breakers and fallbacks behave as designed

Fix gaps.

## Level 4 · Organizing, Migrating & Operating Microservices

### [The Strangler Fig Pattern: Migrating a Monolith Incrementally](level-4/01-strangler-fig-pattern.md)

**Q121.** What is the strangler fig pattern?

**Answer:** Gradually replacing parts of a legacy system by routing functionality to new services

Incremental.

**Q122.** What sits in front during strangling?

**Answer:** A facade/proxy routing requests to old or new implementations

Switch per route.

**Q123.** When is the legacy removed?

**Answer:** After traffic and data are fully migrated and verified

Retire.

**Q124.** What's the main benefit?

**Answer:** Continuous delivery of value with reversible steps

Lower risk.

### [Team Topologies &amp; Conway's Law](level-4/02-team-topologies-conways-law.md)

**Q125.** What does Conway's law say?

**Answer:** System design mirrors the communication structure of the organization

Align teams and services.

**Q126.** What's the inverse Conway maneuver?

**Answer:** Shape teams to produce the desired architecture

Intentional.

**Q127.** What is a stream-aligned team?

**Answer:** A team owning a flow of business value end to end

Team Topologies.

**Q128.** What does a platform team provide?

**Answer:** Self-service capabilities to reduce others' cognitive load

Paved road.

### [Versioning &amp; Backward Compatibility Across Services](level-4/03-versioning-backward-compatibility.md)

**Q129.** How change an API without breaking consumers?

**Answer:** Expand and contract: add new, migrate consumers, then remove old

Parallel change.

**Q130.** What's a tolerant reader?

**Answer:** A consumer ignoring unknown fields

Compatibility.

**Q131.** How long support old versions?

**Answer:** A published deprecation window with usage monitoring

Sunset.

**Q132.** What detects who still uses v1?

**Answer:** Metrics/logs by version and consumer

Evidence.

### [Shared Libraries vs Code Duplication](level-4/04-shared-libraries-vs-duplication.md)

**Q133.** What's the risk of a shared library with domain logic?

**Answer:** Tight coupling: every service must upgrade together

Distributed monolith.

**Q134.** What's acceptable to share?

**Answer:** Stable cross-cutting utilities (logging, auth clients) with strict versioning

Careful.

**Q135.** What does duplication buy?

**Answer:** Independence at the price of repeated code

Trade-off.

**Q136.** What helps shared code at scale?

**Answer:** Versioned, backward-compatible packages and sidecar/mesh for infra

Governance.

### [Deployment Strategies for Many Independent Services](level-4/05-deployment-strategies.md)

**Q137.** What does blue-green deployment do?

**Answer:** Runs two environments and switches traffic between them

Fast rollback.

**Q138.** What does rolling update do?

**Answer:** Replaces instances gradually keeping capacity

Default.

**Q139.** What's a prerequisite for safe rollouts?

**Answer:** Backward-compatible DB and API changes

Expand/contract.

**Q140.** What enables instant rollback?

**Answer:** Keeping the previous version deployable and routing back

Immutable.

### [Observability Across Service Boundaries](level-4/06-observability-across-boundaries.md)

**Q141.** What's needed to debug across services?

**Answer:** Correlated logs, metrics and traces with shared IDs

Observability.

**Q142.** What's a golden signals set?

**Answer:** Latency, traffic, errors and saturation

Per service.

**Q143.** What's an SLO?

**Answer:** A reliability target for a user-facing indicator

Error budget.

**Q144.** Why define SLOs per service?

**Answer:** Ownership and prioritization of reliability work

Accountability.

### [Testing Strategies in a Microservices World](level-4/07-testing-strategies.md)

**Q145.** What is the test pyramid for microservices?

**Answer:** Many unit tests, fewer service/contract tests, very few end-to-end

Speed.

**Q146.** What do component tests do?

**Answer:** Test a service in isolation with its dependencies stubbed

Fast.

**Q147.** Why contract tests over e2e for interfaces?

**Answer:** Faster, more stable and pinpoint failures

Reliability.

**Q148.** What does testing in production use?

**Answer:** Canaries, feature flags and synthetic checks with strong observability

Controlled.

### [Anti-Patterns: the Distributed Monolith &amp; Chatty Services](level-4/08-antipatterns-distributed-monolith.md)

**Q149.** What's a distributed monolith?

**Answer:** Services so coupled they must be deployed and changed together

All the cost, none of the benefit.

**Q150.** Common causes?

**Answer:** Shared databases, synchronous chains and shared domain libraries

Avoid them.

**Q151.** What indicates it?

**Answer:** Lockstep releases and cascading failures

Smells.

**Q152.** How fix it?

**Answer:** Re-draw boundaries, own data per service and use async events

Refactor.

### [When Not to Use Microservices](level-4/09-when-not-to-use-microservices.md)

**Q153.** When are microservices a bad idea?

**Answer:** Small teams, unclear domain or limited operational maturity

Start simple.

**Q154.** What operational capabilities are required?

**Answer:** CI/CD, observability, automation and on-call maturity

Prerequisites.

**Q155.** What's a sensible default?

**Answer:** A well-structured modular monolith, splitting when needed

Evolve.

**Q156.** What costs grow with service count?

**Answer:** Operational overhead, latency and complexity

Pay-as-you-go.

### [Capstone: Build, Break &amp; Observe a Real Multi-Service System](level-4/10-capstone-multi-service-system.md)

**Q157.** What does the capstone combine?

**Answer:** Several services with sagas, resilience, mesh security and observability

End to end.

**Q158.** How prove it's resilient?

**Answer:** Failure injection and recovery drills

Evidence.

**Q159.** What documentation helps?

**Answer:** Context maps, API contracts and runbooks

Operate.

**Q160.** What sustains it?

**Answer:** Clear ownership, SLOs and continuous improvement

Lifecycle.

## Mixed questions across all levels

**Q161.** Why prefer a modular monolith first for a new product?

**Answer:** The domain is still being learned and distributed-system costs aren't yet justified

Split when boundaries and teams are clear.

**Q162.** 'Customer' means different things to Sales and Billing. What does DDD suggest?

**Answer:** Separate bounded contexts with explicit translation between them

Context mapping.

**Q163.** Three synchronous services at 99.9% each give what end-to-end availability?

```python
print(round(0.999**3*100,3))
```

**Answer:** 99.7

Availabilities multiply. Verified by running it.

**Q164.** Two services read and write the same tables. What anti-pattern is this?

**Answer:** A shared database creating hidden coupling (a distributed monolith)

Each service should own its data.

**Q165.** Checkout needs data from three services. What are good options?

**Answer:** API composition or a read model built from events (no cross-database joins)

Data ownership.

**Q166.** A payment fails after stock was reserved. What pattern restores consistency?

**Answer:** A saga with a compensating action (release stock)

No distributed rollback.

**Q167.** The service writes to its DB then publishes an event, and sometimes only one happens. What fixes it?

**Answer:** The transactional outbox pattern

Dual-write problem.

**Q168.** A client retries a payment POST after a timeout. How do you avoid double charging?

**Answer:** An idempotency key stored with the result

Idempotent endpoints.

**Q169.** Events can be delivered twice. What must consumers be?

**Answer:** Idempotent (deduplicate by event ID)

At-least-once delivery.

**Q170.** A downstream service is down and callers pile up threads. What pattern helps?

**Answer:** Circuit breaker with timeouts (and a fallback)

Fail fast.

**Q171.** Retries from 1,000 clients hit a recovering service at the same instant. What do you add?

**Answer:** Exponential backoff with jitter (and a retry budget)

Avoid retry storms.

**Q172.** One slow dependency exhausts the shared thread pool. What isolates it?

**Answer:** Bulkheads: separate pools/limits per dependency

Contain the blast radius.

**Q173.** What status should a rate-limited caller receive?

**Answer:** 429 Too Many Requests with Retry-After

Clear backpressure signal.

**Q174.** You want uniform mTLS and retries across languages without code changes. What do you use?

**Answer:** A service mesh with sidecar proxies

Control plane distributes policy.

**Q175.** How do you roll out a risky version safely?

**Answer:** Canary: shift a small percentage of traffic and promote based on error rate and latency

Automated analysis.

**Q176.** A request is slow and you don't know which service is to blame. What helps?

**Answer:** Distributed tracing with propagated trace context

Spans per hop.

**Q177.** A provider changes an API and breaks an unknown consumer. What prevents it?

**Answer:** Consumer-driven contract tests verified in the provider's CI

Pact.

**Q178.** How do you migrate a legacy system with minimal risk?

**Answer:** Strangler fig: route functionality gradually to new services behind a facade

Reversible steps.

**Q179.** Releases require coordinating five services. What is this symptom?

**Answer:** A distributed monolith: wrong boundaries or shared data/libraries

Re-draw boundaries.

## Practical coding questions

### A circuit breaker

Implement CircuitBreaker.call with closed, open and half-open states: open after N consecutive failures, fail fast while open, allow a probe after reset_after seconds (success closes, failure re-opens). A fake clock makes the simulation deterministic.

Starter code:

```
class Clock:
    def __init__(self): self.t = 0
    def now(self): return self.t
    def advance(self, s): self.t += s

class CircuitBreaker:
    """closed -> open after `threshold` consecutive failures; after `reset_after` seconds go half-open
    and allow ONE probe: success closes the breaker, failure re-opens it."""
    def __init__(self, clock, threshold=3, reset_after=10):
        self.clock, self.threshold, self.reset_after = clock, threshold, reset_after
        self.state, self.failures, self.opened_at = "closed", 0, None

    def call(self, fn):
        # open: reject immediately ("rejected (fail fast)") until reset_after seconds have passed, then half-open;
        # call fn(); on exception count the failure ("failed") and open when failures >= threshold
        # (or at once if half-open); on success close the breaker and reset the failure count; return the result
        return None

clock = Clock()
cb = CircuitBreaker(clock)
healthy = {"up": False}
def dependency():
    if not healthy["up"]:
        raise RuntimeError("boom")
    return "ok"

log = []
for _ in range(5):
    log.append(cb.call(dependency))
print(log, cb.state)
clock.advance(10)
print("probe while still down:", cb.call(dependency), cb.state)
clock.advance(10)
healthy["up"] = True
print("probe after recovery:", cb.call(dependency), cb.state)
print("normal call:", cb.call(dependency))
```

Expected output:

```text
['failed', 'failed', 'failed', 'rejected (fail fast)', 'rejected (fail fast)'] open
probe while still down: failed open
probe after recovery: ok closed
normal call: ok
```

??? success "Show a solution"

    ```
    class Clock:
        def __init__(self): self.t = 0
        def now(self): return self.t
        def advance(self, s): self.t += s

    class CircuitBreaker:
        """closed -> open after `threshold` consecutive failures; after `reset_after` seconds go half-open
        and allow ONE probe: success closes the breaker, failure re-opens it."""
        def __init__(self, clock, threshold=3, reset_after=10):
            self.clock, self.threshold, self.reset_after = clock, threshold, reset_after
            self.state, self.failures, self.opened_at = "closed", 0, None

        def call(self, fn):
            if self.state == "open":
                if self.clock.now() - self.opened_at >= self.reset_after:
                    self.state = "half-open"
                else:
                    return "rejected (fail fast)"
            try:
                result = fn()
            except Exception:
                self.failures += 1
                if self.state == "half-open" or self.failures >= self.threshold:
                    self.state, self.opened_at = "open", self.clock.now()
                return "failed"
            self.state, self.failures = "closed", 0
            return result

    clock = Clock()
    cb = CircuitBreaker(clock)
    healthy = {"up": False}
    def dependency():
        if not healthy["up"]:
            raise RuntimeError("boom")
        return "ok"

    log = []
    for _ in range(5):
        log.append(cb.call(dependency))
    print(log, cb.state)
    clock.advance(10)
    print("probe while still down:", cb.call(dependency), cb.state)
    clock.advance(10)
    healthy["up"] = True
    print("probe after recovery:", cb.call(dependency), cb.state)
    print("normal call:", cb.call(dependency))
    ```

    Failing fast while open protects callers and the struggling dependency; the half-open probe tests recovery without a flood. Verified by running the solution.

### A saga with compensation, plus an outbox

Implement run_saga: execute steps in order and, when one fails, undo the completed steps in reverse order, returning ROLLED_BACK (or COMPLETED). The provided outbox relay republishes an event after a crash, and an idempotent consumer applies each event once.

Starter code:

```
class Service:
    def __init__(self, name, fail_on=None):
        self.name, self.fail_on, self.ledger = name, fail_on, {}

    def do(self, order, action):
        if self.fail_on == action:
            raise RuntimeError(f"{self.name} failed")
        self.ledger[order] = action

    def undo(self, order):
        self.ledger.pop(order, None)

def run_saga(order, steps, log):
    # steps = [(service, action, compensation)]. Run each: svc.do(order, action) and log "<svc>:<action>".
    # On RuntimeError log "FAILED <message>", then undo the COMPLETED steps in reverse order (svc.undo(order))
    # logging "<svc>:<compensation>" for each, and return "ROLLED_BACK"; return "COMPLETED" when all succeed.
    return ""

class Outbox:
    """Events are written with the state change; a relay publishes them at-least-once; consumers dedupe by id."""
    def __init__(self): self.rows, self.next_id = [], 1
    def add(self, payload):
        self.rows.append({"id": self.next_id, "payload": payload, "sent": False}); self.next_id += 1
    def relay(self, bus, crash_after_publish=False):
        for r in self.rows:
            if not r["sent"]:
                bus.append((r["id"], r["payload"]))
                if crash_after_publish:
                    return                      # crashed before marking as sent -> will be published again
                r["sent"] = True

def consume(bus, processed, effects):
    for eid, payload in bus:
        if eid in processed:
            continue                            # idempotent consumer
        processed.add(eid); effects.append(payload)

def new_steps(fail_on=None):
    inv, pay, ship = Service("inventory"), Service("payment", fail_on), Service("shipping")
    return [(inv, "reserve", "release"), (pay, "charge", "refund"), (ship, "schedule", "cancel")], (inv, pay, ship)

log = []; steps, svcs = new_steps()
print(run_saga("o1", steps, log), log)
log = []; steps, svcs = new_steps(fail_on="charge")
print(run_saga("o2", steps, log), log, [s.ledger for s in svcs])

ob, bus, processed, effects = Outbox(), [], set(), []
ob.add("OrderPlaced:o1"); ob.add("OrderPaid:o1")
ob.relay(bus, crash_after_publish=True)          # relay crashes after publishing the first event
ob.relay(bus)                                    # restart: the first event is published again
consume(bus, processed, effects)
print("bus:", [e for e, _ in bus])
print("effects:", effects)
```

Expected output:

```text
COMPLETED ['inventory:reserve', 'payment:charge', 'shipping:schedule']
ROLLED_BACK ['inventory:reserve', 'FAILED payment failed', 'inventory:release'] [{}, {}, {}]
bus: [1, 1, 2]
effects: ['OrderPlaced:o1', 'OrderPaid:o1']
```

??? success "Show a solution"

    ```
    class Service:
        def __init__(self, name, fail_on=None):
            self.name, self.fail_on, self.ledger = name, fail_on, {}

        def do(self, order, action):
            if self.fail_on == action:
                raise RuntimeError(f"{self.name} failed")
            self.ledger[order] = action

        def undo(self, order):
            self.ledger.pop(order, None)

    def run_saga(order, steps, log):
        """steps = [(service, action, compensation)]. Run in order; on failure compensate completed steps in reverse."""
        done = []
        for svc, action, comp in steps:
            try:
                svc.do(order, action)
                log.append(f"{svc.name}:{action}")
                done.append((svc, comp))
            except RuntimeError as e:
                log.append(f"FAILED {e}")
                for s, c in reversed(done):
                    s.undo(order)
                    log.append(f"{s.name}:{c}")
                return "ROLLED_BACK"
        return "COMPLETED"

    class Outbox:
        """Events are written with the state change; a relay publishes them at-least-once; consumers dedupe by id."""
        def __init__(self): self.rows, self.next_id = [], 1
        def add(self, payload):
            self.rows.append({"id": self.next_id, "payload": payload, "sent": False}); self.next_id += 1
        def relay(self, bus, crash_after_publish=False):
            for r in self.rows:
                if not r["sent"]:
                    bus.append((r["id"], r["payload"]))
                    if crash_after_publish:
                        return                      # crashed before marking as sent -> will be published again
                    r["sent"] = True

    def consume(bus, processed, effects):
        for eid, payload in bus:
            if eid in processed:
                continue                            # idempotent consumer
            processed.add(eid); effects.append(payload)

    def new_steps(fail_on=None):
        inv, pay, ship = Service("inventory"), Service("payment", fail_on), Service("shipping")
        return [(inv, "reserve", "release"), (pay, "charge", "refund"), (ship, "schedule", "cancel")], (inv, pay, ship)

    log = []; steps, svcs = new_steps()
    print(run_saga("o1", steps, log), log)
    log = []; steps, svcs = new_steps(fail_on="charge")
    print(run_saga("o2", steps, log), log, [s.ledger for s in svcs])

    ob, bus, processed, effects = Outbox(), [], set(), []
    ob.add("OrderPlaced:o1"); ob.add("OrderPaid:o1")
    ob.relay(bus, crash_after_publish=True)          # relay crashes after publishing the first event
    ob.relay(bus)                                    # restart: the first event is published again
    consume(bus, processed, effects)
    print("bus:", [e for e, _ in bus])
    print("effects:", effects)
    ```

    Sagas replace distributed transactions with local steps plus compensations; the outbox gives at-least-once publishing and consumers dedupe by event id. This is a teaching model. Verified by running the solution.

---

More free practice: the [Microservices Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
