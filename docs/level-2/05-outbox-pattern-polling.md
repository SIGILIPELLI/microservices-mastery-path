---
description: "A real outbox row that survives a SIGKILL crash of the service that wrote it, then gets delivered automatically once a separate poller process runs."
---

# The Outbox Pattern Without Kafka

## What was run

A real `order_service_outbox.py` (port 8081) writing to a real SQLite file, `orders_outbox.db`, with an `orders`
table and an `outbox` table written in the *same* transaction. The service was then killed with `kill -9` — a real
hard crash, not a graceful shutdown — and the outbox row's survival was confirmed by reading the SQLite file
directly with a separate Python process while the service was still dead. A separate `outbox_poller.py` process and
`shipping_service.py` (port 8082) were started afterward to show the surviving row get delivered automatically.

## Writing the order and its outbox entry atomically

```python
@app.post("/orders")
def create_order(o: OrderIn):
    cur = conn.cursor()
    cur.execute("INSERT INTO orders (item) VALUES (?)", (o.item,))
    oid = cur.lastrowid
    cur.execute("INSERT INTO outbox (order_id, payload) VALUES (?, ?)", (oid, json.dumps({"order_id": oid, "address": o.address})))
    conn.commit()   # both rows commit together, or neither does
```

```bash
curl -s -X POST localhost:8081/orders -H 'Content-Type: application/json' -d '{"item":"bookshelf","address":"42 Wallaby Way"}'
# {"id":1,"item":"bookshelf"}

curl -s localhost:8081/outbox
# [{"id":1,"order_id":1,"payload":"{\"order_id\": 1, \"address\": \"42 Wallaby Way\"}","published":false}]
```

## A real crash, and the real proof the row survives it

```bash
pkill -9 -f "uvicorn order_service_outbox:app"   # a real SIGKILL, no graceful shutdown

python3 -c "
import sqlite3
conn = sqlite3.connect('orders_outbox.db')
print(conn.execute('SELECT id, order_id, payload, published FROM outbox').fetchall())
"
# [(1, 1, '{"order_id": 1, "address": "42 Wallaby Way"}', 0)]
```

The order service process is dead. No Python object holding that outbox row exists anywhere in memory. The row is
still there, because SQLite's `conn.commit()` fsyncs it to disk as part of a real transaction — this is the entire
point of the pattern, and it's the thing [Level 1, Lesson 4](../level-1/04-sync-vs-async-communication.md)'s
in-memory `queue.Queue` fundamentally could not provide: that demo's queued job would have been gone forever after a
crash like this one. This one isn't.

## Recovery, with no special "recovery mode" needed

```bash
uvicorn shipping_service:app --port 8082 &      # Shipping comes up
python3 outbox_poller.py &                       # a separate process, polling the same SQLite file

# a few seconds later:
curl -s localhost:8082/shipments
# [{"order_id":1,"address":"42 Wallaby Way"}]

python3 -c "... SELECT published FROM outbox ..."
# [(1, 1, '...', 1)]   <- published flipped to 1
```

Nothing about the poller's logic knows or cares that the order service crashed in between. It just scans for
`published = 0` rows, same as it always does — crash recovery here isn't a special code path, it's the *absence* of
a special code path, because the durable state (the SQLite file) never depended on any one process staying alive.

## How It Actually Works

The core problem the outbox pattern solves is the "dual write" problem: a service needs to both change its own
state (create the order) and reliably notify another system about it (tell Shipping), and those two things cannot
be one atomic operation if the notification is a network call to a different service — exactly the gap
[Level 1, Lesson 4](../level-1/04-sync-vs-async-communication.md) demonstrated with its fragile in-memory queue. The
outbox pattern turns this into a single-database problem instead: write the state change *and* a durable record of
"this needs to be sent" in the same local transaction, which the database already knows how to make atomic. A
separate process (the poller here; in production, often a tool like Debezium reading the database's write-ahead log
instead of polling a table directly) is then responsible for reliably getting that durable record out to wherever it
needs to go, retrying as needed, with no risk of losing it because it only ever reads already-committed state.

This demo's `time.sleep(2)` polling loop is the simplest possible version of "a separate process delivers outbox
rows" and has a real, honest limitation: it adds up to 2 seconds of latency between an order being created and its
shipment request being sent, and it does real wasted work scanning for rows even when there's nothing new. A
message broker (Kafka, RabbitMQ) removes that polling delay by having the poller push to a topic the moment it finds
a new row, and lets consumers subscribe instead of the producer needing to know every downstream consumer's URL
directly, as this demo's poller does.

## Common mistakes

- Writing the business-state change and the outbox row in two separate transactions (even to the same database) —
  this reintroduces exactly the dual-write race the pattern exists to close; they must be one transaction.
- Deleting outbox rows after publishing instead of marking them published — losing the row removes your own audit
  trail of what was sent and when, which matters the first time you need to debug "did this shipment request ever
  actually get sent?"
- Running more than one poller instance against the same table with no locking — two pollers can both pick up the
  same unpublished row and send it twice, which is why real implementations add a `SELECT ... FOR UPDATE SKIP LOCKED`
  or equivalent claim step before publishing.

## Exercise

Modify `outbox_poller.py` to also record a `delivery_attempts` count and a `last_error` column on the outbox table,
incrementing it on every failed send. Stop `shipping_service.py`, create two new orders, let the poller retry for
about 10 seconds, then restart Shipping and confirm both orders are eventually delivered and `delivery_attempts`
shows a real count greater than zero for both.
