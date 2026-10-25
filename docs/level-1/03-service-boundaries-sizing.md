---
description: "Why 'one service per noun' fails and what actually determines the right size for a service, using call-count and ownership heuristics."
---

# Sizing a Service: Boundaries That Actually Hold

## What was run

No new service — this lesson is a sizing exercise applied to the `Shipping` context you sketched in the
[Lesson 2](02-bounded-contexts-ddd.md) exercise, plus a call-graph thought experiment on the running `monolith.py`.

## Why "one service per noun" fails

A natural first instinct is: `Customer` is a noun, give it a service; `Order` is a noun, give it a service;
`Shipment` is a noun, give it a service. This fails in practice for a specific, measurable reason: it ignores how
often those nouns need to talk to *each other* to do anything useful.

Imagine `Order` and `OrderLineItem` as separate services. Nearly every operation on an order — add an item, compute a
total, cancel — needs to touch both. Every one of those becomes a network call where an in-process function call used
to be. You've paid the full cost of a service split (deploy complexity, network latency, partial-failure handling)
and received none of the benefit (independent deployability is fiction if the two "services" must always deploy
together to stay correct; independent scaling is fiction if they're always called together in a 1:1 ratio).

This failure mode has a name: a system split this way is often still a "distributed monolith" — many deployables,
but with the coupling (and none of the resilience) of one. It's covered directly as an anti-pattern in
[Level 4, Lesson 8](../level-4/08-antipatterns-distributed-monolith.md); this lesson is about avoiding it at design
time.

## How It Actually Works

The actual sizing heuristic that holds up is: **a service should be sized so that its most frequent, most
latency-sensitive operations don't need a network call to another service.** Concretely, three questions, applied to
each bounded context from Lesson 2:

1. **Call frequency and co-change.** Do these two concepts change together, in the same commit, most of the time?
   `Order` and `OrderLineItem` almost always change together — strong signal they're one service. `Order` and
   `Shipment` change independently (an order's items don't change because a shipment got delayed) — signal they can
   be separate.
2. **Transactional coupling.** Must two concepts be updated in the same atomic transaction to avoid an invalid state
   that matters? If yes, keeping them in one service with one database is usually simpler than building a saga
   (Level 2) to fake that atomicity across two services. If the "invalid state" in between is actually tolerable for
   a short window (an order exists before its shipment label is printed), that's a real signal they can split.
3. **Team ownership.** Does a different team, with a different release cadence and different on-call rotation, own
   this concept? This is often the deciding factor in practice even when the first two heuristics are ambiguous —
   see [Conway's Law](../level-4/02-team-topologies-conways-law.md) in Level 4.

Applied to the `Shipping` context from Lesson 2's exercise: shipments are created *after* an order exists, are
updated independently (carrier pickup, in-transit, delivered) on their own timeline, and a temporary inconsistency
("order placed, shipment not yet created") is completely normal and already how real shipping works. That's a strong
case for `Shipping` as its own service, talking to `Ordering` asynchronously rather than needing a shared transaction.

A second, orthogonal sizing failure mode is going too far the other way: a "service" so small it does nothing without
immediately calling three others (sometimes called a "nanoservice"). The same three questions catch this too — if a
service's *only* job is to wrap a single database table with no independent business rules, no independent release
cadence, and no team boundary justifying it, it's probably not earning its deployment overhead.

## Common mistakes

- Sizing services around your current org chart's smallest team unit rather than around real coupling — team
  boundaries matter (heuristic 3) but shouldn't override hard transactional coupling (heuristic 2).
- Treating "this could theoretically scale independently" as sufficient justification on its own, without checking
  whether the two concepts are actually called together constantly (heuristic 1) — theoretical independence doesn't
  help if every request needs both anyway.
- Refusing to ever revisit a size decision — a service that turns out too chatty with its neighbors (reaching for the
  circuit breakers and retries of Level 3 just to survive normal operation) is a strong, concrete signal to merge it
  back, not a problem to route around indefinitely.

## Exercise

Using the three heuristics above, decide whether `OrderLineItem` (the individual items within an order, with their
own quantity and price) should be its own service, separate from `Order`, or part of the same service. Write out your
reasoning against each of the three questions specifically — not just a one-line verdict — before moving to
[Lesson 4](04-sync-vs-async-communication.md), which assumes `Order` and `Shipping` are separate services and has you
build the call between them for real.
