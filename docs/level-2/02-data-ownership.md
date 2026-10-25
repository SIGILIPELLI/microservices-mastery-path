---
description: "A real N+1 cross-service call pattern measured at 50 HTTP calls, fixed with a batch endpoint, measuring a real 5.7x speedup for 50 orders."
---

# Data Ownership & Avoiding Distributed Joins

## What was run

A real `customers_service.py` (port 8051) with 20 customers, and a real Python script, `n_plus_one_demo.py`, that
enriches 50 orders with customer names two different ways against that live service: one HTTP call per order (50
calls total), and one batch call for all unique customer IDs needed (1 call). Both timings are real, measured on this
machine.

## The distributed join, measured

```python
def enrich_n_plus_1():
    for o in orders:                                              # 50 orders
        r = c.get(f"{CUSTOMERS_URL}/customers/{o['customer_id']}")  # 50 separate HTTP calls
        result.append({**o, "customer_name": r.json()["name"]})
```

```bash
python3 n_plus_one_demo.py
# N+1 (50 separate calls for 50 orders): 0.0574s
# Batched (1 call for 20 unique customers): 0.0101s
# Speedup: 5.7x
```

Fifty orders referencing only 20 distinct customers means 30 of those 50 calls were redundant — the exact same
customer fetched multiple times across different orders. This is the distributed-systems version of the classic
"N+1 query" problem familiar from ORMs, except each redundant lookup here is a full network round trip, not a cheap
local query.

## The fix Customers Service had to provide

```python
@app.post("/customers/batch")
def get_customers_batch(ids: list[int]):
    return [customers[i] for i in ids if i in customers]
```

```python
def enrich_batched():
    ids = sorted({o["customer_id"] for o in orders})              # 20 unique IDs
    r = c.post(f"{CUSTOMERS_URL}/customers/batch", json=ids)       # 1 HTTP call
    by_id = {cust["id"]: cust for cust in r.json()}
    return [{**o, "customer_name": by_id[o["customer_id"]]["name"]} for o in orders]
```

Critically, **Orders Service cannot fix this problem by itself.** It can batch its own *outgoing* requests all it
wants, but without a `/customers/batch` endpoint on the other side, there's no single call to make. The fix required
Customers Service — the owner of that data — to decide how bulk access to its own data should work. This is the
core idea of data ownership: only the owning service gets to decide the efficient ways to access its data, because
only it understands its own storage, indexing, and caching.

## How It Actually Works

In a monolith, "give me these 20 customers" is one SQL query with a `WHERE id IN (...)` clause, regardless of how
many records the caller happens to need — the database engine already solves this problem. Across a service
boundary, every distinct access *pattern* a consumer needs (single lookup, batch lookup, "all customers updated
since timestamp X" for a sync job, etc.) has to be deliberately built as its own endpoint by the owning service,
because an owning service's whole point is to **not** expose its internal storage directly (the shared-database
anti-pattern from [Lesson 1](01-database-per-service.md)).

This creates a real, ongoing API design responsibility for the owning service that a monolith's shared database
never demanded: anticipate the real access patterns consumers need, and expose them explicitly — a batch endpoint, a
pagination scheme, a "changed since" endpoint for sync — rather than forcing every consumer to either hammer it with
N+1 calls (as measured above) or reach around it into its database (as Lesson 1 showed breaking). The data owner,
not the data consumer, carries this responsibility, because only the owner can change its efficient-access strategy
(add an index, add a cache) without needing every consumer to change too.

The alternative some systems reach for — giving each consumer a local, denormalized copy of the data it needs,
updated asynchronously via events — avoids the N+1 problem differently by not calling the owning service per-request
at all. That's the CQRS pattern covered in [Lesson 7](07-cqrs-across-services.md), and it trades the network cost
measured here for a staleness cost instead.

## Common mistakes

- Blaming the consuming service for "too many calls" when the real fix (a batch endpoint) can only be built by the
  owning service — data ownership means the fix for bad data-access ergonomics is the owner's job, not the
  consumer's workaround.
- Building a batch endpoint with no upper limit on how many IDs can be requested at once, turning a fix for N+1 into
  a new way to send one enormous, slow request instead.
- Exposing a generic query/filter endpoint ("send me any SQL-like filter") instead of a small number of deliberate,
  well-understood access patterns — this reintroduces the shared-database problem one layer up, because now any
  internal detail a filter can touch is effectively part of the public contract.

## Exercise

Add a limit to `/customers/batch` (reject requests for more than, say, 100 IDs with a `400`), then modify
`n_plus_one_demo.py` to generate 150 orders referencing 120 unique customers and confirm the batch call now correctly
fails with your new limit. Write down one real pagination or chunking strategy you'd use to still serve that request
efficiently without removing the limit.
