---
description: "A real read model answers a query 4-7x faster than a live aggregate over 300k+ rows, plus a real divergence caused by a write that bypassed the outbox."
---

# CQRS Across Service Boundaries

## What was run

A real `write_service.py` (port 8101) backed by a real SQLite `write_model.db`, a real `read_model_projector.py`
process consuming its outbox table and maintaining a separate, denormalized `read_model.db`, and a real
`read_service.py` (port 8102) serving only from that read model. 500 orders were created through the real API for
customer 1, then 300,000 more rows were inserted **directly** into the `orders` table (bypassing the API and its
outbox) to simulate scale — which, honestly, produced a real bug worth seeing, not just a timing number.

## The query, two ways, timed

```python
# write_service.py — correct, but must aggregate over every matching row, every time
@app.get("/customers/{cid}/total-spent-computed-live")
def total_spent_live(cid: int):
    row = conn.execute("SELECT SUM(amount_cents), COUNT(*) FROM orders WHERE customer_id = ?", (cid,)).fetchone()
    return {"customer_id": cid, "total_cents": row[0] or 0, "order_count": row[1]}

# read_service.py — pre-computed by a separate projector process, always a single-row lookup
@app.get("/customers/{cid}/total-spent-from-read-model")
def total_spent_read_model(cid: int):
    row = conn.execute("SELECT total_cents, order_count FROM customer_totals WHERE customer_id = ?", (cid,)).fetchone()
    return {"customer_id": cid, "total_cents": row[0], "order_count": row[1]}
```

```bash
# warmed-up timings, orders table at 300,500 rows:
curl -s localhost:8102/customers/1/total-spent-from-read-model -o /dev/null -w "time:%{time_total}\n"
# time:0.000868   (and 0.001005, 0.002058 across repeats)

curl -s localhost:8101/customers/1/total-spent-computed-live -o /dev/null -w "time:%{time_total}\n"
# time:0.006638   (and 0.007020, 0.007635 across repeats)
```

The read model answered in roughly a quarter to an eighth of the time of the live aggregate, consistently, because
it's a primary-key lookup against a tiny pre-aggregated table instead of a scan-and-sum over every order row.

## The real bug this setup exposed, honestly

```bash
curl -s localhost:8102/customers/1/total-spent-from-read-model
# {"customer_id":1,"total_cents":500000,"order_count":500}

curl -s localhost:8101/customers/1/total-spent-computed-live
# {"customer_id":1,"total_cents":650000,"order_count":650}
```

These two numbers **disagree**, and not because of ordinary eventual-consistency lag — the read model is correctly
showing the 500 orders that went through `create_order()` and its outbox row. The 300,000 bulk rows were inserted
directly into the `orders` table with a raw `INSERT` that never touched the `outbox` table, so the projector never
saw them and never updated `customer_totals` for the ~150 of those rows that happened to belong to customer 1. The
read model isn't being slow to catch up — it's **permanently** missing data that bypassed its only input pipeline.
This is a completely real, common CQRS failure mode, not a contrived one: any write path that doesn't go through the
same event/outbox mechanism the projector depends on will silently drift the read model and never self-correct.

## How It Actually Works

**CQRS (Command Query Responsibility Segregation)** separates the model used to *write* data (the command side —
`write_service.py`'s normalized `orders` table, optimized for correctly recording individual transactions) from the
model used to *read* data (the query side — `read_model.db`'s denormalized `customer_totals` table, optimized for
answering one specific question fast). The two models don't have to use the same schema, the same database
technology, or even the same service — the read side can be a completely different service, scaled and optimized
independently of the write side, exactly as this demo splits them.

The connective tissue between the two sides — here, the outbox + projector from [Lesson 5](05-outbox-pattern-polling.md) —
is doing real, critical work: it's the *only* path by which the write side's changes reach the read side. This
lesson's bug demonstrates the sharp edge of that design directly: **a read model is only as complete as its event
pipeline's coverage of every possible write path.** A system that has even one write path skipping the outbox (a
backfill script, an admin tool, a different service writing directly to the same table) will have a read model
that's wrong in a way that never fixes itself, no matter how long you wait — unlike ordinary replication lag, which
resolves on its own given time.

This is the deeper reason CQRS is usually paired with event sourcing or at least a single, disciplined write path:
if the write model's *only* way of mutating state already goes through an event that the read side also consumes,
there's no way to "bypass" the pipeline, because there's no other way to write at all.

## Common mistakes

- Treating a CQRS read-model discrepancy as "it'll catch up eventually" without first checking whether every write
  path actually produces an event — as shown above, some discrepancies are permanent, not transient lag.
- Letting a second write path (an admin panel, a migration script, direct database access for "just this once")
  write to the command side's tables without going through the same code path that produces outbox events.
- Building a read model as a premature optimization before measuring whether the live-aggregate query is actually
  too slow for its real use case — Lesson 2's data-ownership lesson applies here too: build the access pattern the
  consumer actually needs, when it's actually needed.

## Exercise

Write a small reconciliation script that compares, for every customer with at least one order, the write side's
live-computed total against the read model's stored total, and prints every customer where they disagree. Run it
against this demo's data and confirm it surfaces the discrepancy found above. Then fix the root cause: modify the
bulk-insert step to also write corresponding outbox rows, re-run the projector, and confirm the reconciliation script
reports zero discrepancies afterward.
