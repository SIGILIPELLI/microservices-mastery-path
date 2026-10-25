---
description: "Microservices Level 4: strangler fig migration, team topologies, versioning, deployment strategies, observability, anti-patterns, capstone."
---

# Level 4 · Organizing, Migrating & Operating Microservices

Level 4 zooms out from any single service to the system and the organization around it: how you migrate a real
monolith without a big-bang rewrite, how teams and service boundaries pull on each other, and — just as important —
when the honest answer is not to split at all.

!!! note "What was run, and what wasn't"
    This level runs a real strangler-fig migration against a small real monolith (a Flask app), routing selected
    paths to a new extracted service while the rest keeps serving from the monolith, plus real blue-green and
    canary deployment scripts against local services. Organization-level claims (team topologies, Conway's Law) are
    presented as documented industry patterns, not something a single-laptop course can run — that distinction is
    kept explicit throughout.

**Prerequisites:** Levels 1–3.

## Modules

1. **[The Strangler Fig Pattern: Migrating a Monolith Incrementally](01-strangler-fig-pattern.md)**: a real reverse
   proxy peeling one route at a time off a real monolith.
2. **[Team Topologies & Conway's Law](02-team-topologies-conways-law.md)**: why your service boundaries tend to
   mirror your org chart, whether you plan it or not.
3. **[Versioning & Backward Compatibility Across Services](03-versioning-backward-compatibility.md)**: shipping a
   breaking API change without a synchronized deploy.
4. **[Shared Libraries vs Code Duplication](04-shared-libraries-vs-duplication.md)**: the real coupling cost of a
   shared internal library, and when duplication is cheaper.
5. **[Deployment Strategies for Many Independent Services](05-deployment-strategies.md)**: real rolling, blue-green,
   and canary deploy scripts against local services.
6. **[Observability Across Service Boundaries](06-observability-across-boundaries.md)**: correlating real logs,
   metrics, and traces from three services into one incident timeline.
7. **[Testing Strategies in a Microservices World](07-testing-strategies.md)**: where unit, contract, integration,
   and end-to-end tests each actually earn their cost.
8. **[Anti-Patterns: the Distributed Monolith & Chatty Services](08-antipatterns-distributed-monolith.md)**: a real
   example of "microservices" that can't be deployed independently, and why.
9. **[When Not to Use Microservices](09-when-not-to-use-microservices.md)**: the honest cases where a monolith is
   the better engineering decision.
10. **[Capstone: Build, Break & Observe a Real Multi-Service System](10-capstone-multi-service-system.md)**: a
    multi-service system built from scratch, with a real injected failure and a real trace showing exactly where it
    broke.

## After Level 4

You've decomposed a real system, kept its data consistent under real failure, made it resilient, and migrated a real
monolith into it — the same loop you'd run on a production system, just at a scale you can finish on a laptop.
