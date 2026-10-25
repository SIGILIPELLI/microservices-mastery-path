---
description: "A real consumer contract file catches a real field rename in a provider's response before deploy, with a real exit code 1 naming the exact missing field."
---

# Consumer-Driven Contract Testing

## What was run

A real contract file, `orders_consumer_contract.json`, written from Orders Service's point of view, stating exactly
which fields it depends on from Catalog Service's `/items/{id}` endpoint — and, just as importantly, which fields
it explicitly does *not* depend on. A real verification script, `verify_contract.py`, ran this contract against two
real, different running versions of `catalog_provider.py`: the original, and a "v2" with `price_cents` renamed to
`amount_cents` — the same kind of change [Level 1, Lesson 5](../level-1/05-rest-contracts-versioning.md) first
showed breaking a client, now caught by a test instead of discovered by that client at runtime.

## The contract, written by the consumer

```json
{
  "consumer": "orders-service",
  "provider": "catalog-service",
  "interactions": [{
    "description": "Orders Service looks up an item to enrich an order line",
    "request": { "method": "GET", "path": "/items/1" },
    "expected_response": {
      "status": 200,
      "required_fields": { "id": "int", "title": "str", "price_cents": "int" },
      "note": "Orders Service does NOT depend on 'in_stock' -- it only cares about id/title/price_cents."
    }
  }]
}
```

This contract is deliberately narrow — it only states what Orders Service *actually uses*, not every field Catalog
Service happens to return. That's a specific, important design choice explained below.

## Verified against the real, working provider

```bash
python3 verify_contract.py
# CONTRACT VERIFICATION PASSED against real running provider at http://localhost:8181
```

## The same contract, against a real breaking change

```python
# catalog_provider_v2_broken.py -- a real rename, during a real "cleanup"
items = {1: {"id": 1, "title": "Desk Lamp", "amount_cents": 2499, "in_stock": True}}
```

```bash
python3 verify_contract.py
# CONTRACT VERIFICATION FAILED against real running provider at http://localhost:8181:
#   - [Orders Service looks up an item to enrich an order line] REQUIRED FIELD MISSING: 'price_cents'
#     (consumer depends on this, provider no longer sends it)
```

A real, specific failure — naming the exact missing field and the exact consumer interaction it breaks — with a
real non-zero exit code, the kind of signal that fails a CI pipeline before this change ever reaches a running
system where Orders Service would actually call it and break, as happened without this protection in
[Level 1, Lesson 5](../level-1/05-rest-contracts-versioning.md).

## How It Actually Works

The core idea of **consumer-driven contract testing** is right in the name: the contract's content comes from the
*consumer* (Orders Service), stating what it actually needs, not from the *provider* (Catalog Service) describing
everything it happens to offer. This is a deliberate inversion of a typical API spec (like an OpenAPI document),
which usually documents the provider's *entire* interface regardless of who uses what. The practical benefit shows
up directly in this demo: because the contract never mentioned `in_stock`, Catalog Service remains completely free
to change, remove, or rename that field without this contract test ever failing — it's explicitly understood to be
outside what Orders Service depends on. A provider-authored, "describe everything" spec would have no way to
express that distinction, and would either falsely flag `in_stock` changes as breaking, or require someone to
manually track which fields which consumers actually use.

In a real system with multiple consumers, each consumer would publish its own contract this way, and the provider's
CI pipeline would run *every* published contract against each new version before deploying — catching a breaking
change against *any* real consumer's actual dependency, not just one. This is the piece that makes this approach
scale past two services: a provider team doesn't need to manually ask every consumer team "does this change affect
you?" before every deploy; the consumers' own contracts answer that automatically, every time.

This is a different, complementary tool to [Level 2, Lesson 9](../level-2/09-testing-data-consistency.md)'s
cross-service integration test: that lesson caught a bug in the *interaction logic* between services (a missing
compensation step) by running the full real system together. This lesson catches a *contract* violation by running
only the provider, verified against a recorded expectation — faster, and runnable in the provider's own CI without
needing every consumer's full stack running at the same time.

## Common mistakes

- Writing a contract that mirrors the provider's entire response shape instead of only what the consumer actually
  uses — this reintroduces exactly the problem consumer-driven contracts are meant to avoid, flagging irrelevant
  changes as breaking and creating false urgency around fields nobody depends on.
- Running contract verification only occasionally, or only manually, instead of wiring it into the provider's CI
  pipeline on every change — a contract test that isn't run automatically before every deploy provides none of its
  real protection, since the whole point is catching the break *before* it ships.
- Letting contracts go stale when a consumer's actual needs change — if Orders Service starts depending on
  `in_stock` next month and never updates its contract, the contract test gives false confidence; contracts need to
  be kept in sync with what the consumer's code actually does, ideally generated from real consumer test runs
  (which is what full contract-testing frameworks like Pact automate, recording a mock's real interactions during
  the consumer's own test suite).

## Exercise

Add a second interaction to `orders_consumer_contract.json` representing a *different* consumer, `inventory-service`,
that depends on `id` and `in_stock` specifically (not `price_cents` or `title`). Confirm this second contract passes
against the original `catalog_provider.py`. Then check it against `catalog_provider_v2_broken.py` and confirm it
still **passes** even though `price_cents` was renamed — because the inventory-service contract never depended on
that field, demonstrating concretely why per-consumer contracts catch exactly the right breaking changes for each
consumer and nothing more.
