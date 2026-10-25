---
description: "3 real unit tests ran in 0.845ms; the real integration suite from Level 2 took 5247ms for 2 tests -- roughly a 9,300x per-test cost difference."
---

# Testing Strategies in a Microservices World

## What was run

A real, trivial unit test file (`unit_test_example.py`, three tests of a pure `apply_discount()` function, no
network, no services) timed with `time.perf_counter()`. Then the real integration test suite from
[Level 2, Lesson 9](../level-2/09-testing-data-consistency.md) (`test_saga_consistency.py`, which drives four real
running services over real HTTP, including the real 2.5-second payment delay added in
[Level 2, Lesson 8](../level-2/08-partial-failures-eventual-consistency.md)) timed the same way, as a subprocess.

## The real cost difference, measured

```bash
3 unit tests ran in 0.845ms
2 integration tests ran in 5247.1ms
```

Per test, that's roughly **0.28ms** for a unit test versus roughly **2,623ms** for an integration test in this real
suite — a difference of almost **9,300x**. This isn't a contrived example to make a point: the integration tests
are genuinely slower because they're doing genuinely more — starting real HTTP connections, waiting on a real
payment service that has a real, deliberate 2.5-second delay built into it, and verifying real cross-service state
in `order_service.py` and `payment_service.py`'s actual memory.

## How It Actually Works

This course has already built and run three different kinds of tests, each one deliberately chosen for what it
alone could catch:

- **Unit tests** (what `unit_test_example.py` represents) test one function or one small unit of logic, in
  isolation, with no network calls — fast enough (sub-millisecond here) to run thousands of times per minute,
  making them the right tool for the bulk of a test suite's pure logic (discount math, validation rules, parsing).
  They cannot catch anything about how two services behave *together*, because they never involve two services.
- **Integration tests** (`test_saga_consistency.py` from [Level 2, Lesson 9](../level-2/09-testing-data-consistency.md))
  run real services together and assert on real cross-service outcomes — the only kind of test that could have
  caught that lesson's missing-refund bug, because the bug lived entirely in the *interaction* between services,
  not inside any one of them. The real cost, measured above, is that they're slow, by a factor approaching four
  orders of magnitude for this suite, and that cost scales with however many services and real network calls a
  given test chain touches.
- **Contract tests** ([Level 3, Lesson 9](../level-3/09-consumer-driven-contracts.md)) sit in between: they run
  against one real service (fast, no multi-service chain) but verify a cross-service concern (does this provider
  still satisfy what a specific consumer depends on) that a unit test alone can't express.

The practical implication of the real 9,300x gap measured above: a test suite made up *entirely* of integration
tests becomes genuinely too slow to run on every code change, pushing teams toward running it less often (once a
day, only before a release) — which delays feedback and makes failures harder to diagnose, since more changes have
landed since the last run. The usual answer, often visualized as a "testing pyramid," is proportion: many fast unit
tests forming a large base, a smaller number of integration tests covering the specific cross-service interactions
that actually need real multi-service verification (not every possible one), and a handful of contract tests per
consumer-provider relationship — each tier sized to its real cost, not used uniformly for everything just because
it's possible.

## Common mistakes

- Writing an integration test for something a unit test could verify just as well — if a cross-service call could
  be mocked without losing the thing you're actually testing, the real multi-service cost measured above is being
  paid for no additional protection.
- Having *no* integration tests at all, relying only on fast unit tests — this leaves exactly the class of bug
  [Level 2, Lesson 9](../level-2/09-testing-data-consistency.md) demonstrated (a cross-service interaction bug)
  with no automated test capable of catching it, regardless of how many unit tests exist.
- Running the full integration suite on every single local save during development, when its real cost (seconds
  per test, as measured here) makes that workflow genuinely painful — reserving the full integration suite for CI
  or pre-merge checks, while developing against fast unit tests locally, is a common, practical compromise.

## Exercise

Write one more integration test against the real saga services (still running from this lesson, or restarted from
[Level 2, Lesson 4](../level-2/04-saga-pattern.md)) and time it the same way this lesson did. Then write the
*equivalent* logic as a unit test with the HTTP calls replaced by direct Python function calls or mocks, time that
too, and report your own measured ratio between the two — confirming for yourself, with fresh real numbers, that
this lesson's 9,300x figure is representative of a real, general pattern and not a one-off result from one
specific slow test.
