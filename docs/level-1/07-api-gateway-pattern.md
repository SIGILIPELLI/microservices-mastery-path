---
description: "A real gateway fanning out to a REST and a gRPC backend, plus a genuine gotcha where it collapsed a real 500 into a misleading 'unavailable' message."
---

# The API Gateway Pattern

## What was run

A real `gateway.py` FastAPI service (port 8015) that calls two real downstream services for a single client request:
the REST `catalog_service.py` from [Lesson 5](05-rest-contracts-versioning.md) and the gRPC `Inventory` service from
[Lesson 6](06-grpc-service-calls.md). All three were running locally and called with real `curl` requests.

## One client call, two backends, one response — and a real gotcha

```python
@app.get("/api/items/{item_id}")
def get_item(item_id: int):
    catalog = httpx.get(f"{CATALOG_URL}/v2/items/{item_id}").raise_for_status().json()
    stock = inventory_stub.GetStock(StockRequest(item_id=item_id))  # gRPC, different protocol entirely
    return {"id": catalog["id"], "title": catalog["title"], "price_cents": catalog["price_cents"],
            "in_stock": stock.quantity_available > 0, "quantity_available": stock.quantity_available}
```

```bash
curl -s localhost:8015/api/items/1
# {"id":1,"title":"Desk Lamp","price_cents":2499,"in_stock":true,"quantity_available":42}

curl -s localhost:8015/api/items/2
# {"detail":"catalog service unavailable"}
```

Item 2 exists in the gRPC Inventory service (with quantity 0) but not in Catalog Service's `items` dict. The gateway's
response for it is `"catalog service unavailable"` — which is **not what actually happened**. Checking directly:

```bash
curl -s -w "\nHTTP:%{http_code}\n" localhost:8014/v2/items/2
# Internal Server Error
# HTTP:500
```

Catalog Service was fully up; it just has an unhandled `KeyError` for an item ID it doesn't know (`items[item_id]`
with no existence check), which FastAPI turns into a generic `500`. The gateway's `except httpx.HTTPError` catches
*both* a real connection failure and this unrelated `500` the same way, because `httpx.HTTPStatusError` (raised by
`raise_for_status()` on any 4xx/5xx) is itself a subclass of `httpx.HTTPError`. The gateway collapsed two genuinely
different failures — "the service is down" and "the service is up but this specific request is invalid" — into one
indistinguishable, misleading message. This is a real, common gateway bug, not a contrived one.

## How It Actually Works

An API Gateway sits between external callers and a set of internal services, and does some combination of: routing a
request to the right backend service, aggregating multiple backend calls into one response (as above), translating
between protocols (REST in, gRPC to a backend, as above), and cross-cutting concerns like authentication, rate
limiting, and request logging that would otherwise need to be duplicated in every service.

Its real value is hiding internal topology from external callers — a caller of `/api/items/1` doesn't need to know
Catalog is REST and Inventory is gRPC, or that they're two different processes at all. Its real cost, demonstrated
above, is that **the gateway becomes responsible for correctly interpreting every backend's failure modes**, and
getting that wrong (as this demo's gateway did) produces a worse debugging experience than calling each backend
directly would have — the client sees "unavailable" when the truth is "backend returned a 500 for this specific
input," two very different problems requiring different fixes.

The correct fix, which real gateways implement, is to distinguish failure types explicitly rather than catching one
broad exception class:

```python
try:
    catalog_resp = httpx.get(...)
    catalog_resp.raise_for_status()
except httpx.ConnectError:
    raise HTTPException(503, "catalog service unreachable")
except httpx.HTTPStatusError as e:
    raise HTTPException(e.response.status_code, f"catalog service returned {e.response.status_code}")
```

This is also the architectural argument for the [Backend-for-Frontend pattern](08-backend-for-frontend.md) (next
lesson): a single generic gateway trying to serve every client type and hide every backend's quirks accumulates
exactly this kind of lossy error-translation logic, multiplied across every route.

## Common mistakes

- Catching a broad exception type at the gateway and translating every case to the same generic error, exactly as
  this demo did — always distinguish "couldn't reach the backend" from "backend responded with an error" from
  "backend responded successfully with data the gateway can't parse."
- Putting business logic (not just routing/aggregation/translation) in the gateway — a gateway that decides discount
  rules or validates domain rules becomes a second place those rules live, and the two copies drift.
- Making the gateway a single point of failure with no redundancy — since literally every client request flows
  through it, it needs the same (or better) availability engineering as the services behind it.

## Exercise

Fix the gateway's error handling using the pattern shown above (distinguish `httpx.ConnectError` from
`httpx.HTTPStatusError`), then re-run both `curl localhost:8015/api/items/1` and
`curl localhost:8015/api/items/2` and confirm the second one now returns a status code and message that actually
reflects what Catalog Service did (a `500`), not a generic "unavailable." Separately, fix the root cause in
`catalog_service.py` by raising a proper `404` for an unknown `item_id` instead of letting the `KeyError` propagate.
