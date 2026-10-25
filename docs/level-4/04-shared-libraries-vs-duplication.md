---
description: "A real shared Python package, upgraded once, silently breaks two unrelated services' public JSON contracts at the same moment, neither asking for it."
---

# Shared Libraries vs Code Duplication

## What was run

A real, pip-installable local package, `shop-common` (built with a real `pyproject.toml` and installed with
`pip install -e .`), providing one function, `format_price()`, imported by two real, otherwise-unrelated services:
`orders_service.py` (port 8231) and `newsletter_service.py` (port 8232). The shared package was then upgraded in
place from a real v1.0.0 to a real v2.0.0 with a breaking change, and both services were restarted — the real
equivalent of a shared library's new version being rolled out everywhere it's used.

## Both services, sharing one real installed package

```python
# shop_common/__init__.py, v1.0.0
def format_price(cents: int) -> str:
    return f"${cents/100:.2f}"

# orders_service.py
from shop_common import format_price
return {"item": "keyboard", "price_display": format_price(5000)}

# newsletter_service.py
from shop_common import format_price
return {"item": "desk lamp", "price_display": format_price(2499)}
```

```bash
curl -s localhost:8231/orders/1/receipt
# {"item":"keyboard","price_display":"$50.00"}
curl -s localhost:8232/newsletter/deal-of-the-day
# {"item":"desk lamp","price_display":"$24.99"}
```

Both services' responses used the exact same shared formatting logic — real, working code reuse, no duplicated
`f"${cents/100:.2f}"` string floating around in two places.

## One upgrade, two services broken at the same instant

```python
# shop_common/__init__.py, v2.0.0 -- the orders team didn't ask for this, didn't review it
def format_price(cents: int) -> dict:
    return {"currency": "USD", "amount": round(cents / 100, 2)}
```

```bash
# after restarting BOTH services to pick up the new shared package version:
curl -s localhost:8231/orders/1/receipt
# {"item":"keyboard","price_display":{"currency":"USD","amount":50.0}}

curl -s localhost:8232/newsletter/deal-of-the-day
# {"item":"desk lamp","price_display":{"currency":"USD","amount":24.99}}
```

Neither service's own code changed by a single character. Both services' real, public JSON contracts changed
shape — `price_display` went from a plain string to a nested object — at exactly the same moment, because both
depend on the same installed copy of the same function. Any real client of either service that expected
`price_display` to be a string (to display it directly, say) breaks identically in both places, simultaneously,
from a change neither service's own team necessarily reviewed as "a change to my service's API."

## How It Actually Works

Sharing code between services through a real library (as opposed to each service keeping its own, possibly
duplicated, copy of the same logic) gives real, genuine benefits: write the logic once, fix a bug in one place
instead of N places, and get consistent behavior everywhere it's used — exactly what the v1.0.0 scenario
demonstrated working correctly. The real cost, demonstrated by the v2.0.0 upgrade, is that **a shared library
creates a deployment-coupling dependency between every service that uses it**, even services that have nothing
else to do with each other, as `orders_service.py` and `newsletter_service.py` don't. Upgrading the library is not
a decision any one service's team can make in isolation — it silently affects every consumer, whether or not that
consumer's team reviewed, tested against, or even knew about the change.

This is directly analogous to the shared-database problem from
[Level 2, Lesson 1](../level-2/01-database-per-service.md): there, two services sharing one database table were
coupled through an invisible schema dependency; here, two services sharing one library are coupled through an
invisible function-signature dependency. The fix follows the same shape too — **duplication, deliberately chosen**,
removes the coupling entirely: if `orders_service.py` and `newsletter_service.py` had each kept their own private
copy of a price-formatting function, upgrading one would never affect the other, at the real cost of maintaining
two (or more) copies of the same small piece of logic, and manually keeping any shared bug fixes in sync.

The actual engineering judgment call, applied per shared piece of code: is this logic genuinely stable and
unlikely to need a breaking change (good candidate for sharing — most pure utility functions, like parsing or
formatting with no business-rule awareness), or does it represent business logic that's likely to evolve
independently for different consumers' needs (better candidate for duplication, or at minimum, strict semantic
versioning with real consumers pinning specific major versions rather than always taking latest)?

## Common mistakes

- Sharing a library specifically to enforce "consistency" for logic that different consumers actually have
  legitimately different needs for — this creates pressure to either break one consumer's expectations (as this
  demo shows) or add consumer-specific branches inside the shared code, which defeats the purpose of sharing it.
- Treating a shared internal library as exempt from the versioning discipline [Lesson 3](03-versioning-backward-compatibility.md)
  described for service APIs — a breaking change to a widely-used internal package needs the same real-usage-data
  question ("who still depends on the old behavior?") before it ships everywhere at once.
- Letting "don't repeat yourself" become an absolute rule rather than a tradeoff — a few lines of duplicated,
  genuinely independent logic across two services is often cheaper, long-term, than a shared dependency that
  quietly couples their release schedules together.

## Exercise

Add a second function to `shop_common`, `format_date(timestamp)`, used only by `newsletter_service.py` (not by
`orders_service.py`). Make a breaking change to *only* `format_date`, leaving `format_price` untouched this time.
Restart both services and confirm `orders_service.py`'s behavior is completely unaffected — then write one sentence
on why this safer case (breaking only a function one consumer actually uses) still doesn't eliminate the deployment
coupling: both services still had to be considered, even if only one was actually affected, every time the shared
package changes at all.
