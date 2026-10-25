---
description: "A real migration on a shared SQLite database breaks a second service's query with a real 'no such column' error, then the database-per-service fix."
---

# Database-per-Service & Why Shared Databases Break Down

## What was run

A real SQLite database, `shared_shop.db`, read directly by two independent scripts standing in for two services. One
script (`orders_reads_shared.py`) represents Orders Service reading a `customers` table it doesn't own. A real
migration was then applied directly against that same file — standing in for Customers Service deploying
independently — and the Orders Service script was re-run against the now-changed schema with no code changes of its
own.

## Before the migration, it works

```python
# orders_reads_shared.py — Orders Service, reading a table it doesn't own, directly
row = conn.execute("SELECT id, name, email FROM customers WHERE id = 1").fetchone()
```

```bash
python3 orders_reads_shared.py
# Orders Service read customer: id=1 name=Ada Lovelace email=ada@example.com
```

## Customers Service ships an unrelated, reasonable migration

```sql
ALTER TABLE customers DROP COLUMN email;
ALTER TABLE customers ADD COLUMN contact_info TEXT;  -- a JSON blob, to support phone + email + preferences later
```

This is, from Customers Service's point of view, an entirely internal, reasonable refactor: replacing one scalar
column with a more flexible structure, something any team should be free to do without asking permission.

```bash
python3 orders_reads_shared.py
# Orders Service query FAILED: no such column: email
```

That's a real `sqlite3.OperationalError`, not a simulated one. Orders Service didn't change a single line of its own
code. It broke because it depended directly on Customers Service's internal table structure — a dependency that was
invisible until the moment it broke in production.

## The fix: each service owns its own database, period

```python
# customers_own_db.py — a real FastAPI service with its own, private SQLite file
conn = sqlite3.connect("customers_only.db")
@app.get("/customers/{cid}")
def get_customer(cid: int):
    row = conn.execute("SELECT id, name, contact_info FROM customers WHERE id = ?", (cid,)).fetchone()
    return {"id": row[0], "name": row[1], "contact_info": row[2]}
```

```bash
curl -s localhost:8041/customers/1
# {"id":1,"name":"Ada Lovelace","contact_info":"{\"email\":\"ada@example.com\"}"}
```

Now Customers Service can rename, drop, or restructure any column in `customers_only.db` it wants, as often as it
wants, and the *only* thing any other service can depend on is the shape of the JSON this endpoint returns — a
contract Customers Service controls and can version deliberately, the way [Lesson 5 of Level 1](../level-1/05-rest-contracts-versioning.md)
covered. No other process even has a file handle open on `customers_only.db`.

## How It Actually Works

A shared database creates a dependency that bypasses every service boundary built in Level 1: it doesn't matter how
cleanly you drew the line between `Orders` and `Customers` in your API design if both processes also read and write
the same tables directly, because the *real* contract between them becomes the full SQL schema — every column, every
type, every constraint — not the deliberately small, versioned API either team intended to expose. This is why the
error above says "no such column," not "API contract violation": the dependency was never going through anything
that could enforce or version a contract in the first place.

**Database-per-service** means each service's data lives in storage only that service's code can access directly.
Every other service that needs that data must go through the owning service's API (synchronously, as a REST/gRPC
call, or asynchronously, as an event) — never a direct connection to its tables. This gives a service real freedom to
change its internal schema whenever it wants, as long as its public API contract stays stable (or is versioned
deliberately when it can't). The real cost, covered across the rest of this level, is that this removes the one
thing a shared database gave you for free: a single ACID transaction spanning both services' data. Lessons 4 and 5
build the saga and outbox patterns that replace that lost atomicity.

## Common mistakes

- Sharing one database "temporarily, just for this one urgent feature" — the coupling this creates is invisible
  until the exact moment shown above, and by then it's load-bearing production behavior, not a quick fix to revert.
- Giving a second service *read-only* access to the first service's tables, thinking read-only avoids the coupling —
  it doesn't; the schema dependency and the breakage shown above are identical whether the second service writes or
  only reads.
- Splitting services at the API layer while leaving their databases merged "to migrate the data later" — this is a
  distributed monolith in exactly the sense covered in [Level 4, Lesson 8](../level-4/08-antipatterns-distributed-monolith.md):
  two deployables, one database-level coupling.

## Exercise

Reproduce this yourself: run `setup_shared_db.py`, confirm `orders_reads_shared.py` succeeds, then apply the
`DROP COLUMN` / `ADD COLUMN` migration shown above and re-run it to see the real failure. Then write a second script,
`orders_reads_via_api.py`, that gets the same customer data by calling `http://localhost:8041/customers/1` instead of
touching SQLite directly, and confirm that a hypothetical future migration to Customers Service's *internal* schema
(renaming `contact_info` again, for instance) would no longer be able to break it, as long as the API response shape
stays the same.
