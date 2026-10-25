---
description: "Why splitting a monolith into microservices costs real latency and complexity, measured with a real FastAPI monolith, and when that trade is worth it."
---

# From Monolith to Microservices: Why and When to Split

## What was run

A real two-table "shop" app (`customers`, `orders`) built as a single FastAPI process, started with `uvicorn` on
`localhost:8001`. Every request and timing number below is from that real running process, not a guess.

## The monolith, for real

```python
@app.get("/orders/{oid}")
def get_order(oid: int):
    order = orders[oid]
    customer = customers[order["customer_id"]]   # <- just a dict lookup, same process
    return {**order, "customer_name": customer["name"], "customer_email": customer["email"]}
```

```bash
curl -s -X POST localhost:8001/customers -H 'Content-Type: application/json' \
  -d '{"name":"Ada Lovelace","email":"ada@example.com"}'
# {"id":1,"name":"Ada Lovelace","email":"ada@example.com"}

curl -s -X POST localhost:8001/orders -H 'Content-Type: application/json' \
  -d '{"customer_id":1,"item":"keyboard","quantity":2}'
# {"id":1,"customer_id":1,"item":"keyboard","quantity":2}

curl -s localhost:8001/orders/1
# {"id":1,"customer_id":1,"item":"keyboard","quantity":2,"customer_name":"Ada Lovelace","customer_email":"ada@example.com"}

curl -s localhost:8001/orders/1 -o /dev/null -w "%{time_total}\n"
# 0.000911
```

Fetching an order and joining it to its customer took under a millisecond, because "joining" here is a Python dict
lookup in the same process. There is no network hop, no serialization, no partial-failure case where the order exists
but the customer service is down. This is the thing a monolith gives you for free, and it is also exactly the thing
you give up when you split.

## How It Actually Works

A monolith's single deployable unit means every function call between "modules" is an in-process call: same memory
space, same transaction if you're inside a database transaction, and it either all succeeds or the whole process
fails. A microservices architecture replaces some of those in-process calls with network calls between independently
deployable processes. That change is not free — it trades away:

- **Latency**: an in-process dict lookup is microseconds; a network call to another service, even on localhost, adds
  real serialization and I/O overhead (you'll measure this directly in [Lesson 4](04-sync-vs-async-communication.md)).
- **Atomicity**: a single SQL transaction across two tables in one database either commits or rolls back entirely. Two
  separate services with two separate databases cannot share one transaction — you need a saga instead (Level 2).
- **Operational surface**: one process to deploy, monitor, and log becomes N processes, N sets of dependencies, N
  places a bug can hide.

What splitting buys back, when a system actually needs it:

- **Independent deployability**: the team owning `orders` can ship a change without coordinating a release with the
  team owning `customers`, as long as the contract between them doesn't break.
- **Independent scaling**: if `orders` gets 100x the traffic of `customers`, you can run 20 instances of one and 2 of
  the other, instead of scaling the whole monolith to match its busiest part.
- **Fault isolation at the process level**: a memory leak or an infinite loop in one service can be killed and
  restarted without taking down everything else — *if* the resilience patterns in Level 3 are actually in place. Split
  without those patterns and you get the opposite: more places for a failure to start, not fewer places for it to
  spread.
- **Team-boundary alignment**: a service boundary can match an ownership boundary, so one team isn't blocked waiting
  on another team's code review to ship (see [Conway's Law](../level-4/02-team-topologies-conways-law.md) in Level 4).

None of these benefits require microservices as a *starting point*. A common, defensible path is: build the monolith
first, instrument it, find out where the actual scaling or team-coordination pain is, and split along that real seam —
not along a seam guessed on day one. The rest of this course assumes you've decided to split; it does not assume that
decision is always right.

## Common mistakes

- Splitting because "microservices are the modern way to build software," with no specific latency, scaling, or
  team-coordination problem the split is meant to solve — you pay all of the costs above for none of the benefits.
- Assuming a network call "is basically free on localhost" — it measurably is not, and the gap gets much worse across
  real network hops, retries, and TLS.
- Treating the split as one-way and permanent — a service boundary that turns out wrong should be mergeable back, and
  a monolith that's internally well-modularized (clear module boundaries, no circular imports) is far easier to split
  correctly later than one that isn't.

## Exercise

Add a third endpoint to `monolith.py`, `GET /customers/{cid}/order-count`, that returns how many orders a customer has
placed — again as a single in-process lookup. Time it with `curl -o /dev/null -w "%{time_total}\n"` the same way shown
above. Then, in your own words, write down one real feature of your own project (or a hypothetical one) where that
same in-process join would become a network call if `orders` and `customers` were split into separate services — and
whether you think that trade would be worth it.
