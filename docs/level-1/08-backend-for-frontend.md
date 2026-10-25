---
description: "Two real BFFs over the same Catalog service, shaped for mobile and web, with a measured 2.6x payload-size difference for the same underlying item."
---

# Backend-for-Frontend (BFF)

## What was run

Two real FastAPI services, `mobile_bff.py` (port 8016) and `web_bff.py` (port 8017), each calling the same
`catalog_service.py` from [Lesson 5](05-rest-contracts-versioning.md) and shaping the response differently for its
client type. Response sizes below are real byte counts from `curl ... | wc -c`.

## Same data, two shapes

```python
# mobile_bff.py — compact card, no raw cents, truncated title
return {"id": i["id"], "t": i["title"][:20], "price_display": f"${i['price_cents']/100:.2f}"}

# web_bff.py — full title, raw cents for client formatting, plus SEO metadata mobile never needs
return {"id": i["id"], "title": i["title"], "price_cents": i["price_cents"],
        "seo": {"meta_title": f"{i['title']} | Shop", "meta_description": f"Buy {i['title']} today."}}
```

```bash
curl -s localhost:8016/mobile/items/1
# {"id":1,"t":"Desk Lamp","price_display":"$24.99"}
curl -s localhost:8017/web/items/1
# {"id":1,"title":"Desk Lamp","price_cents":2499,"seo":{"meta_title":"Desk Lamp | Shop","meta_description":"Buy Desk Lamp today."}}

curl -s localhost:8016/mobile/items/1 | wc -c   #  49
curl -s localhost:8017/web/items/1   | wc -c   # 129
```

The web response is **2.6x the byte size** of the mobile one for the exact same underlying item, because it carries
SEO metadata that only matters to a server-rendered web page, not a mobile app. On a slow mobile connection, that
difference is a real, user-visible latency cost if mobile had to download and discard the SEO fields it doesn't use.

## How It Actually Works

The single generic [API Gateway](07-api-gateway-pattern.md) from the previous lesson has a scaling problem that only
shows up once you have more than one kind of client: a mobile app, a web app, and maybe a third-party partner
integration each want *different shapes* of the same underlying data — different fields, different levels of detail,
different aggregations. A single gateway trying to serve all of them accumulates either:

- One bloated endpoint returning the union of every field every client might ever want (the web BFF's shape, served
  to mobile too) — correct, but wasteful, exactly as measured above.
- A pile of `if client_type == "mobile"` branches inside one endpoint — correct, but every client's logic is now
  tangled together in one file, owned by whichever team touches it last.

The **Backend-for-Frontend** pattern resolves this by giving each client type (or each client *team*) its own
gateway, each calling the same underlying services but shaping, filtering, and aggregating the response exactly for
that one client. The mobile team owns and evolves `mobile_bff.py` without needing sign-off from the web team, and
vice versa — each BFF's release cadence matches its client's release cadence (a mobile app update is slow and
app-store-gated; a web frontend deploys continuously), which a single shared gateway can't do without one team's
urgency blocking the other's stability.

The cost is real too: logic that's genuinely shared across clients (auth, the error-translation fix from Lesson 7)
now has to be duplicated or factored into a shared library across every BFF, and you've traded "N endpoints in one
gateway" for "N gateways," each needing its own deployment and monitoring.

## Common mistakes

- Creating a BFF per client *platform* when the real difference is per client *team* — two teams building for the
  same platform with genuinely different needs is a better BFF boundary than "iOS gets one, Android gets one" if
  both actually want the same shape.
- Letting business logic (not just response shaping) live in a BFF — a discount calculation implemented differently
  in `mobile_bff.py` and `web_bff.py` is the same "two copies of the truth" problem called out in Lesson 7, just
  multiplied by the number of BFFs.
- Skipping the shared error-handling fix from Lesson 7 in each BFF independently — without a shared library, each BFF
  can reintroduce the same "unavailable" bug on its own.

## Exercise

Add a third BFF, `partner_bff.py`, for a hypothetical B2B partner integration that only needs `id` and
`quantity_available` (pulled from the gRPC Inventory service, not Catalog) — no price, no title, no SEO data. Run it
alongside the other two, confirm its response is smaller than both, and write one sentence on why a partner
integration's data needs might diverge from both a mobile app's and a web app's.
