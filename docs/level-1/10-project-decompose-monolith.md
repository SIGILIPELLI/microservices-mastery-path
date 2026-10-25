---
description: "Splitting the Lesson 1 monolith into two real services, measuring a real 13x latency increase for the same join, and two real new failure modes."
---

# Project: Decompose a Monolith Into Two Real Services

## What was run

The `monolith.py` shop app from [Lesson 1](01-monolith-to-microservices.md) was split into two real, independently
running services — `customers_service.py` (port 8031) and `orders_service.py` (port 8032) — with the former
in-process dict lookup replaced by a real `httpx` call between them. Every number and error below is real, observed
output from these two running processes.

## The split, and what it actually cost

```python
# orders_service.py — the join that was one line in monolith.py is now a network call
@app.get("/orders/{oid}")
def get_order(oid: int):
    order = orders[oid]
    r = httpx.get(f"{CUSTOMERS_URL}/customers/{order['customer_id']}", timeout=2.0)
    customer = r.json()
    return {**order, "customer_name": customer["name"], "customer_email": customer["email"]}
```

```bash
curl -s -X POST localhost:8031/customers -H 'Content-Type: application/json' \
  -d '{"name":"Grace Hopper","email":"grace@example.com"}'
# {"id":1,"name":"Grace Hopper","email":"grace@example.com"}

curl -s -X POST localhost:8032/orders -H 'Content-Type: application/json' \
  -d '{"customer_id":1,"item":"compiler manual","quantity":1}'
# {"id":1,"customer_id":1,"item":"compiler manual","quantity":1}

curl -s localhost:8032/orders/1 -w "\ntime:%{time_total}\n"
# {"id":1,"customer_id":1,"item":"compiler manual","quantity":1,"customer_name":"Grace Hopper","customer_email":"grace@example.com"}
# time:0.011994
```

The monolith's equivalent call in Lesson 1 measured `time_total: 0.000911`. This split version measured `0.011994` —
roughly **13x slower** for the exact same logical operation, entirely because a dict lookup became an HTTP round
trip. That cost was predicted in Lesson 1's discussion; here it's measured directly, on the same two pieces of
business logic, before and after the split.

## Two new failure modes that didn't exist in the monolith

```bash
curl -s -w "\nHTTP:%{http_code}\n" -X POST localhost:8032/orders \
  -H 'Content-Type: application/json' -d '{"customer_id":999,"item":"ghost item","quantity":1}'
# {"detail":"unknown customer_id"}
# HTTP:400
```

This one is a correct, intentional validation — Orders Service explicitly checks with Customers Service before
creating an order, something the monolith got for free from a single shared dict.

```bash
pkill -f "uvicorn customers_service:app"

curl -s -w "\nHTTP:%{http_code}\n" localhost:8032/orders/1
# {"detail":"order found, but customers service is unreachable to enrich it"}
# HTTP:503

curl -s -w "\nHTTP:%{http_code}\n" -X POST localhost:8032/orders \
  -H 'Content-Type: application/json' -d '{"customer_id":1,"item":"another item","quantity":1}'
# {"detail":"customers service unreachable"}
# HTTP:503
```

Both of these failure modes are genuinely new. In the monolith, `customers` could never be "unreachable" relative to
`orders` — they were the same process. Now, Orders Service's ability to create *or even just read* an order is
coupled to Customers Service's availability, exactly as [Lesson 1](01-monolith-to-microservices.md) warned in the
abstract. Notice also that the two `503`s have different, honest messages — `get_order` says the order *was* found
but couldn't be enriched, while `create_order` never got far enough to create anything. That distinction matters: a
client seeing the first error knows the order exists; a client seeing the second knows it doesn't.

## How It Actually Works

This project makes concrete every tradeoff discussed across Lessons 1–9: the bounded-context split ( `Customer` vs
`Order`, Lesson 2) was sized correctly enough that each service has a clear, independent reason to exist (Lesson 3);
the communication between them is synchronous REST (Lesson 4, Lesson 5) because `create_order` genuinely cannot
proceed without knowing the customer exists; and the new `503` failure paths are exactly the kind of partial failure
that a production system would want wrapped in a circuit breaker (Level 3) rather than left as a raw, immediate
error on every single request while Customers Service is down.

What this project does *not* yet have, and what the next two levels build: no saga or compensation logic if a
multi-step operation partially fails (Level 2), no resilience pattern softening the all-or-nothing `503` into a
degraded-but-functional response (Level 3), and no story for how Orders Service finds Customers Service other than a
hardcoded URL (solved for a single hardcoded case already, but see [Lesson 9](09-service-discovery.md) for why that
doesn't scale).

## Common mistakes

- Treating "the split works" (requests succeed) as the finish line without also measuring the real cost (the 13x
  latency increase shown above) — a split that's never actually necessary for this system's real scale or team
  structure just paid that cost for nothing.
- Writing the cross-service validation (`unknown customer_id`) but forgetting the cross-service *unavailability* case
  (`customers service unreachable`) — both `httpx.ConnectError` and a `404` response are real things `orders_service.py`
  had to handle that `monolith.py` never needed to think about.
- Giving the two new `503` error paths the same generic message, which would recreate the exact gateway mistake from
  [Lesson 7](07-api-gateway-pattern.md) — this project's two distinct, honest error messages are intentional.

## Exercise

Add a third endpoint to `orders_service.py`, `GET /orders/{oid}/summary`, that returns just `{"item": ..., "quantity": ...}`
with **no** call to Customers Service at all. Confirm with `curl -w "time:%{time_total}\n"` that this endpoint's
latency is close to the original monolith's ~1ms, not the ~12ms of the enriched `/orders/{oid}` endpoint — proving
that the 13x cost measured above is specifically the price of *that one join*, not an inherent property of Orders
Service as a whole.
