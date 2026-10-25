---
description: "Real Deprecation/Sunset HTTP headers on a live endpoint, plus a real usage log proving a 'deprecated' version still has 4 callers -- unsafe to remove."
---

# Versioning & Backward Compatibility Across Services

## What was run

A real `catalog_versioned.py` (port 8221) extending the `/v1` vs `/v2` split from
[Level 1, Lesson 5](../level-1/05-rest-contracts-versioning.md) with real deprecation signaling: standard
`Deprecation` and `Sunset` HTTP response headers (from RFC 8594) on the old version, plus a real server-side log of
every actual call `/v1` receives — the concrete mechanism a real team would use to answer "is it actually safe to
delete this yet?"

## Announcing deprecation with real, standard headers

```python
@app.get("/v1/items/{item_id}")
def get_item_v1(item_id: int, response: Response):
    v1_usage_log.append(time.time())
    response.headers["Deprecation"] = "true"
    response.headers["Sunset"] = "Wed, 01 Jan 2027 00:00:00 GMT"
    response.headers["Link"] = '</v2/items/{}>; rel="successor-version"'.format(item_id)
```

```bash
curl -sD - localhost:8221/v1/items/1
# HTTP/1.1 200 OK
# deprecation: true
# sunset: Wed, 01 Jan 2027 00:00:00 GMT
# link: </v2/items/1>; rel="successor-version"
#
# {"id":1,"title":"Desk Lamp","price":24.99}
```

The old endpoint still works exactly as before — this is backward compatibility, not a breaking removal — but every
caller that actually inspects response headers (or runs tooling that does) now gets a real, standardized signal:
this version is going away, here's roughly when, and here's where to migrate to.

## Proving whether it's actually safe to remove

```python
@app.get("/v1-usage-report")
def v1_usage_report():
    last_24h = [t for t in v1_usage_log if time.time() - t < 86400]
    return {"total_v1_calls_ever_recorded": len(v1_usage_log), "v1_calls_last_24h": len(last_24h),
            "safe_to_remove_v1": len(last_24h) == 0}
```

```bash
# a "legacy client we forgot about" makes a few real calls to v1...
curl -s localhost:8221/v1-usage-report
# {"total_v1_calls_ever_recorded":4,"v1_calls_last_24h":4,"safe_to_remove_v1":false}
```

This is the real, concrete answer to a question that's otherwise just a guess: `/v1` has had **4 real calls** in
the last 24 hours, so `safe_to_remove_v1` is honestly `false`. Without this kind of real usage tracking, a team
deciding whether to delete `/v1` is working from assumption ("probably nobody uses this anymore") rather than
evidence — and the cost of being wrong is a real outage for whichever caller was still depending on it.

## How It Actually Works

Backward compatibility and versioning together answer two separate, related questions: **how do you let an API
evolve without breaking existing callers** ([Level 1, Lesson 5](../level-1/05-rest-contracts-versioning.md)'s `/v1`
vs `/v2` split), and **how do you eventually retire an old version safely, once it exists**, which is what this
lesson adds. The second question is harder than it looks, because the one piece of information you actually need —
"does anyone still call this?" — isn't visible from the API's code at all; it's only visible from real, ongoing
traffic data, exactly what `v1_usage_log` captures here.

The `Deprecation` and `Sunset` headers are a real, standardized (RFC 8594) way to communicate retirement plans
*to automated clients*, not just to humans reading a changelog — a well-behaved client library can check for
`Deprecation: true` and log a warning, or alert on a dashboard, entirely automatically, without a human needing to
notice a blog post. This is the HTTP-level analog of a compiler deprecation warning: the information is attached to
the thing itself, at the point of use, rather than living only in separate documentation that's easy to miss.

Putting these two pieces together gives a complete, real lifecycle: announce deprecation with real timestamps and a
pointer to the replacement (this lesson's headers), keep the old version fully functional during the deprecation
window (standard backward compatibility), and use real traffic data — not a guess, not an announcement date alone —
to decide when actual removal is safe. The *announced* sunset date (January 2027 above) is a target, not a
guarantee; the actual decision to remove `/v1` should wait for the usage data to confirm it, exactly as
`safe_to_remove_v1` does here.

## Common mistakes

- Announcing a sunset date and then removing the old version on that date regardless of what real usage data
  shows — an announced deadline is a target to work toward, not a substitute for checking whether removal is
  actually safe.
- Adding deprecation headers but never actually logging or monitoring real usage of the deprecated endpoint —
  without the usage log this lesson built, the headers alone tell you nothing about whether anyone's actually
  reading them or still depending on the old version.
- Deprecating a version with no working, documented replacement already available — the `Link: successor-version`
  header above only has value if `/v2` is a real, complete, production-ready substitute by the time `/v1` starts
  warning callers away from itself.

## Exercise

Extend `v1_usage_log` to record *which caller* (via a real `User-Agent` or a custom client-identifying header) made
each call, not just when. Simulate three different "client" identities calling `/v1` with different header values,
then extend `/v1-usage-report` to list each distinct caller still active in the last 24 hours — turning "is it safe
to remove" into "here is exactly who still needs to migrate before it's safe," the real, actionable version of this
question a team actually needs answered before deleting anything.
