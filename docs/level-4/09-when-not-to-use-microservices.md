---
description: "Every real cost measured across this course, tallied in one place, as the actual evidence for when a monolith is the better engineering decision."
---

# When Not to Use Microservices

## What was run

No new experiment — this lesson tallies the real, measured costs from every earlier lesson in this course into
one place, as the actual evidence base for the question this lesson is about, rather than restating it as abstract
advice.

## The real bill, so far

| Cost | Measured in | Real number |
|------|-------------|--------------|
| Latency for a simple cross-service join | [Level 1, Lesson 10](../level-1/10-project-decompose-monolith.md) | ~13x slower than the monolith's in-process equivalent |
| Redundant lookups without a batch endpoint | [Level 1, Lesson 2](../level-1/02-bounded-contexts-ddd.md) / [Level 2, Lesson 2](../level-2/02-data-ownership.md) | 5.7x more real HTTP calls than necessary |
| Chattiness in one logical operation | [Level 4, Lesson 8](08-antipatterns-distributed-monolith.md) | 4.1x single-request latency penalty |
| New failure modes that didn't exist in the monolith | [Level 1, Lesson 10](../level-1/10-project-decompose-monolith.md) | 2 new real `503` paths, requiring new handling code |
| Engineering effort to fake a cross-service transaction | [Level 2, Lessons 4-5](../level-2/04-saga-pattern.md) | A full saga + outbox implementation, replacing one SQL `COMMIT` |
| A real eventual-consistency window | [Level 2, Lesson 8](../level-2/08-partial-failures-eventual-consistency.md) | 2.56 real seconds where an order was visibly, correctly `PENDING` |
| Test suite cost per cross-service test | [Level 4, Lesson 7](07-testing-strategies.md) | ~9,300x slower per test than an equivalent unit test |
| Deployment coupling despite separate deployables | [Level 4, Lesson 4](04-shared-libraries-vs-duplication.md) | A shared library upgrade broke 2 "independent" services at once |

None of these numbers are made up to make a point — they're the actual measurements this course produced along
the way, each one a genuine cost of the specific microservices pattern it came from. And every one of them was
paired, in its own lesson, with a real benefit that pattern provides when the system actually needs it: independent
scaling, independent deployability, fault isolation, team autonomy.

## How It Actually Works

The honest, cumulative conclusion from this table is simple to state and easy to forget in practice: **every
pattern in this course is a trade, not a free upgrade.** A monolith that doesn't have a real, specific scaling
problem, a real, specific team-coordination problem, or a real, specific fault-isolation need is paying some
subset of the real costs above — slower cross-entity operations, new failure modes, eventual consistency where
atomicity used to be free, dramatically more expensive tests — for benefits it isn't actually using yet.

The questions worth asking honestly before splitting, each tied to a real measurement above:

- **Is there an actual, current scaling mismatch** between different parts of the system (one part needs 50x the
  resources of another), or is this hypothetical, "might need it someday" scaling? If hypothetical, the 13x
  latency cost and the new failure modes from Lesson 10 are being paid for nothing yet.
- **Is there an actual, current team-coordination problem** — multiple teams genuinely blocked on each other's
  release cadence — or is this one team, or a small team that communicates easily? If one team, Conway's Law
  ([Lesson 2](02-team-topologies-conways-law.md)) suggests the clean boundary a split promises won't hold anyway,
  since there's no organizational pressure keeping it clean.
- **Can the team afford the operational and testing overhead** — the 9,300x per-test cost difference, the real
  engineering effort of a saga and an outbox, the discipline of database-per-service — or is this a small team
  that would spend most of its capacity on this overhead instead of on the product?
- **Does the domain genuinely have the kind of bounded contexts** ([Level 1, Lesson 2](../level-1/02-bounded-contexts-ddd.md))
  that justify separate services, or is the "domain" actually small and tightly interrelated enough that forcing
  a split creates artificial seams with no real counterpart in how the business actually works?

A monolith is not a failure to "modernize" — it's the default, lower-cost starting point that this course's own
numbers argue for, right up until a *specific, measured* problem (not a guess about the future) justifies paying
one of the real costs above for a real, needed benefit.

## Common mistakes

- Splitting preemptively, based on anticipated future scale or team growth that hasn't happened yet — every cost
  in the table above starts accruing immediately, while the anticipated benefit might be months or years away, or
  might never materialize in the form expected.
- Treating "everyone else is doing microservices" as evidence for your own system, without checking whether your
  system has the same scaling, team-size, or fault-isolation needs that made it the right call elsewhere.
- Assuming a monolith can't be well-modularized — a monolith with clean internal module boundaries, enforced
  (not just suggested) by code structure, gets much of the "clear ownership" benefit of microservices without
  paying any of the costs in the table above, and is also the easiest starting point to split later if a real
  need appears.

## Exercise

Pick a real or hypothetical system you're familiar with. Walk through the four questions above for it, honestly,
one at a time, writing a specific answer (not just yes/no) for each — citing an actual current pain point if one
exists, or stating plainly "this is hypothetical, not measured" if it doesn't. Based on your four answers alone, not
on general enthusiasm for either approach, write one sentence stating whether this system should be a monolith, a
few large services, or many small services right now — and what specific, measurable signal would change that
answer in the future.
