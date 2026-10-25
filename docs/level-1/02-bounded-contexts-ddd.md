---
description: "Finding real service boundaries using bounded contexts from domain-driven design, applied to a concrete shop domain, not an abstract diagram."
---

# Bounded Contexts & Domain-Driven Design Basics

## What was run

No new service for this lesson — this one is about a design technique, applied directly to the `monolith.py` shop app
from [Lesson 1](01-monolith-to-microservices.md), whose actual code and data you already have running on
`localhost:8001`.

## The same word, two different things

Look at the monolith's two concepts: `Customer` and `Order`. Now imagine the business grows and adds a support team
that also talks about "customers" — but their idea of a customer includes a support-ticket history, a satisfaction
score, and a list of past complaints. The *sales* team's idea of a customer includes a lead source, a sales rep, and a
pipeline stage. Both teams say "customer." Neither means the same thing.

This is the core insight of domain-driven design's **bounded context**: a model is only consistent and only has one
meaning *within* a boundary. Outside that boundary, the same word can and should mean something else. A bounded
context is not a diagram box you draw first — it's the boundary around a linguistically consistent chunk of the
domain that you discover by listening to how people in the business actually talk, and noticing where the same word
splits into two different meanings or two different sets of rules.

## Finding the seams in the shop domain

Applying this to the shop app honestly, with just two entities, there's really only one plausible context boundary
worth naming yet: `Ordering` (orders, items, quantities) and `Customer Identity` (name, email, contact info). The
useful exercise isn't to force a split on a two-entity toy app — it's to notice the *question* that tells you a
boundary exists:

- Does `Ordering` need the customer's *email*, or does it only need a `customer_id` and the fact that the ID is valid?
- If support tickets were added, would "customer" there need `name`/`email` at all, or just enough to look the person
  up — and does that mean *support* is its own context with its own, much smaller, customer model?

The test for a real bounded-context boundary: **two parts of the domain can use the word "customer" and mean
genuinely different things, validated by genuinely different rules, without ever needing to reconcile those meanings
directly.** If they'd actually break if their models diverged, they're the same context. If they can happily diverge
forever, they're different contexts — and that's your service boundary candidate.

## How It Actually Works

A bounded context maps to a microservice boundary candidate because of a specific DDD rule: inside one bounded
context, there is exactly one model for each concept, and that model is internally consistent. Across a context
boundary, models can and do diverge, and that divergence is intentional, not a bug to be fixed with a shared schema.

This gives you a concrete decomposition heuristic that beats "one service per noun" (the subject of
[Lesson 3](03-service-boundaries-sizing.md)):

1. List the core nouns in the domain (`Customer`, `Order`, `Payment`, `Shipment`, ...).
2. For each noun, ask which teams or parts of the business use it, and whether their *rules* about it genuinely
   differ — not just their field names, their actual business rules (e.g., "a customer's email must be verified
   before checkout" vs. "a customer's email is just a free-text contact field for support").
3. Group nouns and rules that must stay mutually consistent into one context. A `Payment` context that needs to know
   an `Order`'s total and currency, and must never see a stale total, is tightly coupled to `Ordering` — maybe the
   same context, maybe a very chatty neighbor that needs the saga pattern from Level 2.
4. Each resulting context is a *candidate* service boundary — not yet a guaranteed one; [Lesson 3](03-service-boundaries-sizing.md)
   covers when a context is also the right *size* for one deployable service versus several, or part of a shared one.

A **context map** — a diagram of which bounded contexts exist and how they relate (shared model, translation layer,
one context being upstream of another) — is the deliverable DDD recommends before any code. It costs nothing to draw
and it is the single most common missing artifact in microservices projects that end up with chatty, tangled service
boundaries (the "distributed monolith" anti-pattern covered in [Level 4, Lesson 8](../level-4/08-antipatterns-distributed-monolith.md)).

## Common mistakes

- Drawing context boundaries around database tables instead of around *meaning and rules* — two tables can belong to
  the same context, and one table's concept can legitimately need to be split across two contexts with different
  views of it.
- Trying to find one universal `Customer` model that satisfies sales, support, and ordering simultaneously — this is
  exactly the shared-model trap DDD is designed to avoid; a translation layer between contexts is normal and healthy,
  not a failure.
- Doing this exercise once, in a kickoff meeting, and never revisiting it — bounded contexts shift as a business
  evolves, and a context map should be a living document, not a one-time artifact.

## Exercise

Take the shop domain and add a `Shipping` concern to it on paper only (no code yet): once an order is placed, it needs
to be packed, labeled, and tracked until delivered. Write down: (1) what a `Shipment` means to the `Shipping` context
versus what "shipping info" might mean if it stayed inside `Ordering` instead, and (2) one concrete business rule that
would only make sense in one of the two contexts, not both. Keep this answer — [Lesson 3](03-service-boundaries-sizing.md)
uses it directly to decide whether `Shipping` deserves its own service.
