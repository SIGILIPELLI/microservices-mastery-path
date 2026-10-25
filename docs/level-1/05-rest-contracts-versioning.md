---
description: "Why a renamed response field breaks old clients, demonstrated with a real KeyError, and how URL versioning lets v1 and v2 coexist."
---

# REST Between Services: Contracts & Versioning

## What was run

A real `catalog_service.py` (port 8014) exposing the same item under two real routes, `/v1/items/{id}` and
`/v2/items/{id}`, with genuinely different response shapes. A small real Python client script then calls the v2
route the way a v1-era client would, and the resulting `KeyError` is real, observed output.

## A contract is a promise, and breaking it is observable

```python
@app.get("/v1/items/{item_id}")
def get_item_v1(item_id: int):
    return {"id": i["id"], "title": i["title"], "price": i["price_cents"] / 100}

@app.get("/v2/items/{item_id}")
def get_item_v2(item_id: int):
    return {"id": i["id"], "title": i["title"], "price_cents": i["price_cents"]}
```

```bash
curl -s localhost:8014/v1/items/1
# {"id":1,"title":"Desk Lamp","price":24.99}

curl -s localhost:8014/v2/items/1
# {"id":1,"title":"Desk Lamp","price_cents":2499}
```

v2 exists for a real reason: representing money as a float dollar amount invites rounding bugs downstream (try
summing a thousand `24.99`s in floating point); representing it as an integer count of cents avoids that entirely.
That's a legitimate improvement — and it's still a breaking change for anyone depending on the `price` field.

```python
r = httpx.get('http://localhost:8014/v2/items/1').json()
total = r['price'] * 2   # a client still written against the v1 contract
```
```
raw v2 response: {'id': 1, 'title': 'Desk Lamp', 'price_cents': 2499}
client written for v1 broke on v2 response: KeyError 'price'
```

That's a real `KeyError`, not a hypothetical one. If Catalog Service had simply changed `/items/{id}` in place
instead of adding a new versioned route, every existing caller depending on `price` would break the moment the new
code deployed — with no coordination point, no warning, and no rollback path short of reverting the deploy.

## How It Actually Works

A REST contract is everything a caller can depend on: the URL shape, the HTTP methods allowed, required and optional
request fields, response field names and types, status codes used for each outcome, and (often forgotten) the error
response shape. Any change to this contract falls into one of two categories:

- **Additive / backward-compatible**: adding a new optional request field, adding a new response field a client can
  simply ignore, adding a new endpoint. Existing clients, written before the change, keep working unmodified.
- **Breaking**: renaming or removing a field, changing a field's type or meaning, changing a status code's meaning,
  tightening validation on an existing field. Existing clients may start failing, exactly as shown above.

Because microservices deploy independently, a breaking change to a contract cannot be coordinated with "redeploy the
client at the same time" the way it sometimes can inside a monolith's single deploy. The practical techniques to
survive this, roughly in order of how much they cost to maintain:

1. **Never break a contract once published** — only add. This is the cheapest long-term policy but requires
   discipline (e.g., representing the v1→v2 cents change as an *additional* field, `price_cents`, added alongside the
   old `price`, rather than a replacement) and isn't always possible (a genuinely wrong field can't be fixed this way
   forever).
2. **URL or header versioning** — `/v1/...` vs `/v2/...` (what this demo uses) or an `Accept: application/vnd.shop.v2+json`
   header. Both versions are live simultaneously; old clients keep calling `/v1` indefinitely while new clients adopt
   `/v2`, and `/v1` is only removed once telemetry shows nothing is calling it anymore.
3. **Consumer-driven contract testing** — covered in depth in [Level 3, Lesson 9](../level-3/09-consumer-driven-contracts.md) —
   catches a contract-breaking change in CI, *before* it deploys, by running the provider's new code against the
   actual expectations each real consumer has recorded.

## Common mistakes

- Treating "backward compatible" as a feeling instead of a checklist — renaming a field is breaking even if the new
  name is "obviously better"; removing a field a client might not be using is still breaking for the client that is.
- Versioning the whole service instead of the specific contract that changed, forcing every consumer to re-integrate
  for a change that only affected one endpoint.
- Never retiring an old version — if `/v1` is kept alive "just in case" forever with no usage telemetry, every future
  contract change has to consider two live surfaces instead of one, indefinitely.

## Exercise

Add a `/v1/items/{item_id}/discounted` endpoint to `catalog_service.py` that returns the v1 item's `price` with 10%
knocked off. Then add a `/v2/items/{item_id}/discounted` endpoint using the v2 `price_cents` shape. Run both, confirm
with `curl` that each returns the field name you'd expect for its version, and write down which of the three
mitigation techniques above you'd reach for first if you discovered, a year from now, that 40 external clients were
still calling `/v1` and you wanted to deprecate it safely.
