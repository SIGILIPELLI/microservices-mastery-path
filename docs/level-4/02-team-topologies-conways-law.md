---
description: "A real 30-directory lab structure as evidence for Conway's Law: session boundaries shaped the service boundaries, not an upfront architecture diagram."
---

# Team Topologies & Conway's Law

## What was run

No new service for this lesson — instead, a real, honest look at the actual directory structure this course's own
lab code has accumulated across the first three levels, as concrete evidence for the idea this lesson covers.

```bash
find ~/microservices-lab -maxdepth 2 -type d | sort
```

```
level1/discovery  level1/gateway  level1/grpc_demo  level1/project_decompose  level1/rest_contracts  level1/sync_async
level2/cqrs  level2/data_ownership  level2/data_propagation  level2/db_per_service  level2/idempotency
level2/outbox  level2/project_real_saga  level2/saga  level2/testing_consistency
level3/bulkheads  level3/canary  level3/circuit_breaker  level3/contracts  level3/mtls
level3/project_chaos  level3/rate_limiting  level3/retries  level3/service_mesh  level3/tracing
level4/strangler_fig
```

Thirty separate service directories accumulated in one continuous working session, each built to demonstrate one
specific lesson. Nothing here was planned as a system architecture up front — the boundaries are a direct byproduct
of how the work itself was organized, one lesson (one unit of work) at a time. That's a small, literal, concrete
instance of exactly the dynamic Conway's Law describes at a much larger, organizational scale.

## How It Actually Works

**Conway's Law**, stated by Melvin Conway in 1967, says that any system's design tends to mirror the communication
structure of the organization that built it. In microservices terms: if two teams rarely talk to each other, don't
review each other's code, and have separate release schedules, the services they own will tend to have a clean,
well-defined boundary between them — not because anyone drew that boundary on a whiteboard first, but because
every day-to-day friction of coordinating across that boundary pushes the design toward minimizing how much needs
to cross it. The reverse is just as real: two teams that talk constantly and ship interdependent changes together
will tend to produce services with a chatty, tangled boundary between them, regardless of what the architecture
diagram says — because the actual code reflects how people actually worked, not how they intended to.

The practical implication, formalized in the **Team Topologies** framework (Skelton & Pais), is to treat
organizational design as part of system design, not a separate concern that happens after the architecture is
decided: if you want services A and B to have a clean, low-coupling boundary, give them to teams that are
structured to have a clean, low-coupling relationship (clear APIs between them, infrequent need for synchronized
releases, limited need for the two teams to coordinate closely) — rather than designing a clean boundary on paper
and hoping two deeply intertwined teams respect it in practice. Team Topologies names four team types relevant
here: **stream-aligned** teams (own a service end-to-end, aligned to a business capability — most teams should be
this), **platform** teams (provide internal infrastructure, like a shared deployment pipeline, that stream-aligned
teams consume self-service), **enabling** teams (temporarily help stream-aligned teams adopt a new skill or
technology), and **complicated-subsystem** teams (own something that genuinely needs deep specialist knowledge,
like a real-time pricing engine). Matching a service boundary to the *wrong* team type — e.g., making a shared
platform concern "owned" by a stream-aligned team already busy with unrelated feature work — tends to produce
exactly the kind of chatty, badly-maintained boundary Conway's Law predicts.

This connects directly to [Level 1, Lesson 3](../level-1/03-service-boundaries-sizing.md)'s third sizing heuristic
("does a different team own this?") — that heuristic works precisely *because* of Conway's Law. A service boundary
that doesn't match any real team boundary is fighting the organization's natural communication structure, and
Conway's Law predicts it will erode over time back toward whatever boundary the actual communication patterns
support.

## Common mistakes

- Designing an "ideal" service architecture first, then assigning teams to it afterward, without checking whether
  the proposed boundaries match how those teams actually need to communicate day to day — Conway's Law suggests
  this ideal architecture will drift toward the org's real communication structure regardless, often at real cost
  in rework.
- Treating Conway's Law as fixed and unchangeable — the "inverse Conway maneuver" (deliberately restructuring teams
  first, to pull the system's architecture in a desired direction) is a real, used technique, not just an
  observation to passively accept.
- Giving one team ownership of too many unrelated services "for efficiency," which removes the natural
  coordination-cost pressure that would otherwise keep each service's boundary clean and well-scoped.

## Exercise

Pick two services from this course that you built in close succession, within the same short working session (for
example, `level3/circuit_breaker` and `level3/retries`, or any two you remember building back-to-back). Write down
one concrete way their *boundary* might have turned out differently — tighter or looser — if they had instead been
built by two different, loosely-coordinating people, each only seeing the other's work through a published API, with
no shared context about how the other one was implemented.
