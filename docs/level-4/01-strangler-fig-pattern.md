---
description: "A real Envoy facade routing /orders to a new real service while /customers and /reports keep hitting the real legacy monolith, unchanged, behind one URL."
---

# The Strangler Fig Pattern: Migrating a Monolith Incrementally

## What was run

A real `legacy_monolith.py` (port 8211) serving three endpoints — `/orders`, `/customers`, `/reports/sales` — and a
real `new_orders_service.py` (port 8213) that re-implements just `/orders`. A real Envoy instance (port 10002,
reusing the same proxy technology from [Level 3, Lesson 5](../level-3/05-service-mesh-sidecar.md), now doing
migration routing instead of sidecar duties) sits in front of both, routing by path prefix: `/orders/*` goes to the
new service, everything else still goes to the legacy monolith.

## One facade, two real backends, zero client-visible change

```yaml
routes:
  - match: { prefix: "/orders" }
    route: { cluster: new_orders_cluster }   # peeled off
  - match: { prefix: "/" }
    route: { cluster: legacy_cluster }        # everything else, unchanged
```

```bash
curl -s localhost:10002/orders/1
# {"id":1,"item":"legacy widget","migrated_notes":"rebuilt with real validation","handled_by":"NEW_ORDERS_SERVICE"}

curl -s localhost:10002/customers/1
# {"id":1,"name":"Legacy Customer","handled_by":"LEGACY_MONOLITH"}

curl -s localhost:10002/reports/sales
# {"total_sales_cents":999999,"handled_by":"LEGACY_MONOLITH"}
```

Every request went to the same URL, `localhost:10002`. The `handled_by` field — included here deliberately to make
the routing visible — proves that `/orders/1` was actually served by the brand-new service while `/customers/1` and
`/reports/sales` were actually served by the untouched legacy monolith, at the exact same time, through one
consistent entry point. No client of this facade needs to know this split exists.

## How It Actually Works

The strangler fig pattern (named after a real fig species that grows around a host tree, gradually replacing it
entirely while the host is still alive) migrates a monolith to microservices one real, shippable slice at a time:
put a routing facade in front of the monolith, extract one piece of functionality into a new service, route just
that piece's traffic to the new service through the facade, and repeat — with the legacy monolith still fully
running and serving everything not yet migrated, for as long as that takes.

This demo's Envoy facade is deliberately the *same mechanism* as the API Gateway from
[Level 1, Lesson 7](../level-1/07-api-gateway-pattern.md) and the sidecar from
[Level 3, Lesson 5](../level-3/05-service-mesh-sidecar.md) — path-based routing to different backends — applied
here with a migration-specific purpose: the split is temporary and one-directional, gradually shrinking the
legacy monolith's share of traffic over time, rather than a permanent architectural boundary.

The real alternative this pattern avoids is the **big-bang rewrite**: stop shipping anything new, rebuild the
entire system from scratch, and switch over all at once on a single cutover date. That approach concentrates all
the migration's risk into one moment, with the smallest number of lessons learned before the riskiest step, and a
very expensive point of no return if something is wrong. The strangler fig pattern's whole value is that each slice
is a small, independently reversible step — if `new_orders_service.py` has a real bug, the facade's routing rule can
be reverted to send `/orders` back to the legacy monolith in seconds, with everything else completely unaffected,
something a big-bang rewrite's cutover moment cannot offer.

## Common mistakes

- Extracting a slice whose data still lives entirely in the legacy monolith's database — the new service then needs
  either direct access to that database (recreating [Level 2, Lesson 1](../level-2/01-database-per-service.md)'s
  shared-database coupling) or a real data-ownership transfer plan, which is usually the harder part of a
  migration slice, not the routing.
- Migrating slices in an arbitrary order instead of starting with ones that are genuinely low-risk and
  high-value — a good first slice is well-bounded (clear inputs/outputs), doesn't depend heavily on other
  not-yet-migrated parts of the monolith, and has low blast radius if something goes wrong.
- Letting the facade's routing rules become permanent "temporary" infrastructure — the facade is meant to shrink
  the legacy side to zero eventually; a strangler-fig migration that stalls at "80% migrated" forever means
  permanently maintaining the facade's routing complexity *and* the legacy monolith *and* every new service, with
  none of the benefits of being fully migrated.

## Exercise

Extract a second slice: build a `new_customers_service.py` re-implementing `/customers/{cid}`, add a second
`prefix: "/customers"` route to `envoy_strangler.yaml` pointing at it, and reload Envoy. Confirm with `curl` that
`/orders` and `/customers` now both hit their new services while `/reports/sales` is still the only endpoint left
being served by `legacy_monolith.py` — and write down, honestly, what percentage of the original monolith's
functionality is now migrated versus still legacy, the exact kind of tracking a real migration needs to know how
much work remains.
