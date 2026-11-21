---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Microservices Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "Why prefer a modular monolith first for a new product?",
    "options": [
      "Law",
      "The domain is still being learned and distributed-system costs aren't yet justified",
      "Microservices are slower",
      "It's trendy"
    ],
    "correctIndex": 1,
    "explanation": "Split when boundaries and teams are clear."
  },
  {
    "question": "'Customer' means different things to Sales and Billing. What does DDD suggest?",
    "options": [
      "One shared model",
      "Rename the field",
      "One table",
      "Separate bounded contexts with explicit translation between them"
    ],
    "correctIndex": 3,
    "explanation": "Context mapping."
  },
  {
    "question": "Three synchronous services at 99.9% each give what end-to-end availability?\n```python\nprint(round(0.999**3*100,3))\n```\n",
    "options": [
      "99.9",
      "97.0",
      "99.99",
      "99.7"
    ],
    "correctIndex": 3,
    "explanation": "Availabilities multiply. Verified by running it."
  },
  {
    "question": "Two services read and write the same tables. What anti-pattern is this?",
    "options": [
      "Saga",
      "A shared database creating hidden coupling (a distributed monolith)",
      "Outbox",
      "CQRS"
    ],
    "correctIndex": 1,
    "explanation": "Each service should own its data."
  },
  {
    "question": "Checkout needs data from three services. What are good options?",
    "options": [
      "API composition or a read model built from events (no cross-database joins)",
      "Share a schema",
      "Copy nightly",
      "Join their DBs"
    ],
    "correctIndex": 0,
    "explanation": "Data ownership."
  },
  {
    "question": "A payment fails after stock was reserved. What pattern restores consistency?",
    "options": [
      "Ignore",
      "A saga with a compensating action (release stock)",
      "2PC",
      "Retry forever"
    ],
    "correctIndex": 1,
    "explanation": "No distributed rollback."
  },
  {
    "question": "The service writes to its DB then publishes an event, and sometimes only one happens. What fixes it?",
    "options": [
      "Cache",
      "Two commits",
      "Retry",
      "The transactional outbox pattern"
    ],
    "correctIndex": 3,
    "explanation": "Dual-write problem."
  },
  {
    "question": "A client retries a payment POST after a timeout. How do you avoid double charging?",
    "options": [
      "Sleep",
      "An idempotency key stored with the result",
      "Disable retries",
      "Lock the DB"
    ],
    "correctIndex": 1,
    "explanation": "Idempotent endpoints."
  },
  {
    "question": "Events can be delivered twice. What must consumers be?",
    "options": [
      "Faster",
      "Stateless",
      "Ordered",
      "Idempotent (deduplicate by event ID)"
    ],
    "correctIndex": 3,
    "explanation": "At-least-once delivery."
  },
  {
    "question": "A downstream service is down and callers pile up threads. What pattern helps?",
    "options": [
      "Bigger pool",
      "Remove timeouts",
      "Circuit breaker with timeouts (and a fallback)",
      "More retries"
    ],
    "correctIndex": 2,
    "explanation": "Fail fast."
  },
  {
    "question": "Retries from 1,000 clients hit a recovering service at the same instant. What do you add?",
    "options": [
      "More clients",
      "Fixed delay",
      "Exponential backoff with jitter (and a retry budget)",
      "Instant retries"
    ],
    "correctIndex": 2,
    "explanation": "Avoid retry storms."
  },
  {
    "question": "One slow dependency exhausts the shared thread pool. What isolates it?",
    "options": [
      "Caching",
      "Bulkheads: separate pools/limits per dependency",
      "More retries",
      "Global pool"
    ],
    "correctIndex": 1,
    "explanation": "Contain the blast radius."
  },
  {
    "question": "What status should a rate-limited caller receive?",
    "options": [
      "429 Too Many Requests with Retry-After",
      "500",
      "200",
      "404"
    ],
    "correctIndex": 0,
    "explanation": "Clear backpressure signal."
  },
  {
    "question": "You want uniform mTLS and retries across languages without code changes. What do you use?",
    "options": [
      "DNS",
      "A library",
      "A gateway",
      "A service mesh with sidecar proxies"
    ],
    "correctIndex": 3,
    "explanation": "Control plane distributes policy."
  },
  {
    "question": "How do you roll out a risky version safely?",
    "options": [
      "Canary: shift a small percentage of traffic and promote based on error rate and latency",
      "Friday deploy",
      "Restart all",
      "Big bang"
    ],
    "correctIndex": 0,
    "explanation": "Automated analysis."
  },
  {
    "question": "A request is slow and you don't know which service is to blame. What helps?",
    "options": [
      "CPU graphs",
      "Distributed tracing with propagated trace context",
      "Restart",
      "More logs"
    ],
    "correctIndex": 1,
    "explanation": "Spans per hop."
  },
  {
    "question": "A provider changes an API and breaks an unknown consumer. What prevents it?",
    "options": [
      "Docs",
      "More e2e tests",
      "Consumer-driven contract tests verified in the provider's CI",
      "Email"
    ],
    "correctIndex": 2,
    "explanation": "Pact."
  },
  {
    "question": "How do you migrate a legacy system with minimal risk?",
    "options": [
      "Fork",
      "Big-bang rewrite",
      "Freeze",
      "Strangler fig: route functionality gradually to new services behind a facade"
    ],
    "correctIndex": 3,
    "explanation": "Reversible steps."
  },
  {
    "question": "Releases require coordinating five services. What is this symptom?",
    "options": [
      "Good design",
      "CQRS",
      "Mesh",
      "A distributed monolith: wrong boundaries or shared data/libraries"
    ],
    "correctIndex": 3,
    "explanation": "Re-draw boundaries."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A circuit breaker

Implement CircuitBreaker.call with closed, open and half-open states: open after N consecutive failures, fail fast while open, allow a probe after reset_after seconds (success closes, failure re-opens). A fake clock makes the simulation deterministic.

**Starter code**

```yaml
class Clock:
    def __init__(self): self.t = 0
    def now(self): return self.t
    def advance(self, s): self.t += s

class CircuitBreaker:
    """closed -> open after `threshold` consecutive failures; after `reset_after` seconds go half-open
    and allow ONE probe: success closes the breaker, failure re-opens it."""
    def __init__(self, clock, threshold=3, reset_after=10):
        self.clock, self.threshold, self.reset_after = clock, threshold, reset_after
        self.state, self.failures, self.opened_at = "closed", 0, None

    def call(self, fn):
        # open: reject immediately ("rejected (fail fast)") until reset_after seconds have passed, then half-open;
        # call fn(); on exception count the failure ("failed") and open when failures >= threshold
        # (or at once if half-open); on success close the breaker and reset the failure count; return the result
        return None

clock = Clock()
cb = CircuitBreaker(clock)
healthy = {"up": False}
def dependency():
    if not healthy["up"]:
        raise RuntimeError("boom")
    return "ok"

log = []
for _ in range(5):
    log.append(cb.call(dependency))
print(log, cb.state)
clock.advance(10)
print("probe while still down:", cb.call(dependency), cb.state)
clock.advance(10)
healthy["up"] = True
print("probe after recovery:", cb.call(dependency), cb.state)
print("normal call:", cb.call(dependency))
```

**Expected output**

```text
['failed', 'failed', 'failed', 'rejected (fail fast)', 'rejected (fail fast)'] open
probe while still down: failed open
probe after recovery: ok closed
normal call: ok
```

<details><summary>Show solution and explanation</summary>

<pre><code>class Clock:
    def __init__(self): self.t = 0
    def now(self): return self.t
    def advance(self, s): self.t += s

class CircuitBreaker:
    &quot;&quot;&quot;closed -&gt; open after `threshold` consecutive failures; after `reset_after` seconds go half-open
    and allow ONE probe: success closes the breaker, failure re-opens it.&quot;&quot;&quot;
    def __init__(self, clock, threshold=3, reset_after=10):
        self.clock, self.threshold, self.reset_after = clock, threshold, reset_after
        self.state, self.failures, self.opened_at = &quot;closed&quot;, 0, None

    def call(self, fn):
        if self.state == &quot;open&quot;:
            if self.clock.now() - self.opened_at &gt;= self.reset_after:
                self.state = &quot;half-open&quot;
            else:
                return &quot;rejected (fail fast)&quot;
        try:
            result = fn()
        except Exception:
            self.failures += 1
            if self.state == &quot;half-open&quot; or self.failures &gt;= self.threshold:
                self.state, self.opened_at = &quot;open&quot;, self.clock.now()
            return &quot;failed&quot;
        self.state, self.failures = &quot;closed&quot;, 0
        return result

clock = Clock()
cb = CircuitBreaker(clock)
healthy = {&quot;up&quot;: False}
def dependency():
    if not healthy[&quot;up&quot;]:
        raise RuntimeError(&quot;boom&quot;)
    return &quot;ok&quot;

log = []
for _ in range(5):
    log.append(cb.call(dependency))
print(log, cb.state)
clock.advance(10)
print(&quot;probe while still down:&quot;, cb.call(dependency), cb.state)
clock.advance(10)
healthy[&quot;up&quot;] = True
print(&quot;probe after recovery:&quot;, cb.call(dependency), cb.state)
print(&quot;normal call:&quot;, cb.call(dependency))
</code></pre>

<p>Failing fast while open protects callers and the struggling dependency; the half-open probe tests recovery without a flood. Verified by running the solution.</p>
</details>

### A saga with compensation, plus an outbox

Implement run_saga: execute steps in order and, when one fails, undo the completed steps in reverse order, returning ROLLED_BACK (or COMPLETED). The provided outbox relay republishes an event after a crash, and an idempotent consumer applies each event once.

**Starter code**

```yaml
class Service:
    def __init__(self, name, fail_on=None):
        self.name, self.fail_on, self.ledger = name, fail_on, {}

    def do(self, order, action):
        if self.fail_on == action:
            raise RuntimeError(f"{self.name} failed")
        self.ledger[order] = action

    def undo(self, order):
        self.ledger.pop(order, None)

def run_saga(order, steps, log):
    # steps = [(service, action, compensation)]. Run each: svc.do(order, action) and log "<svc>:<action>".
    # On RuntimeError log "FAILED <message>", then undo the COMPLETED steps in reverse order (svc.undo(order))
    # logging "<svc>:<compensation>" for each, and return "ROLLED_BACK"; return "COMPLETED" when all succeed.
    return ""

class Outbox:
    """Events are written with the state change; a relay publishes them at-least-once; consumers dedupe by id."""
    def __init__(self): self.rows, self.next_id = [], 1
    def add(self, payload):
        self.rows.append({"id": self.next_id, "payload": payload, "sent": False}); self.next_id += 1
    def relay(self, bus, crash_after_publish=False):
        for r in self.rows:
            if not r["sent"]:
                bus.append((r["id"], r["payload"]))
                if crash_after_publish:
                    return                      # crashed before marking as sent -> will be published again
                r["sent"] = True

def consume(bus, processed, effects):
    for eid, payload in bus:
        if eid in processed:
            continue                            # idempotent consumer
        processed.add(eid); effects.append(payload)

def new_steps(fail_on=None):
    inv, pay, ship = Service("inventory"), Service("payment", fail_on), Service("shipping")
    return [(inv, "reserve", "release"), (pay, "charge", "refund"), (ship, "schedule", "cancel")], (inv, pay, ship)

log = []; steps, svcs = new_steps()
print(run_saga("o1", steps, log), log)
log = []; steps, svcs = new_steps(fail_on="charge")
print(run_saga("o2", steps, log), log, [s.ledger for s in svcs])

ob, bus, processed, effects = Outbox(), [], set(), []
ob.add("OrderPlaced:o1"); ob.add("OrderPaid:o1")
ob.relay(bus, crash_after_publish=True)          # relay crashes after publishing the first event
ob.relay(bus)                                    # restart: the first event is published again
consume(bus, processed, effects)
print("bus:", [e for e, _ in bus])
print("effects:", effects)
```

**Expected output**

```text
COMPLETED ['inventory:reserve', 'payment:charge', 'shipping:schedule']
ROLLED_BACK ['inventory:reserve', 'FAILED payment failed', 'inventory:release'] [{}, {}, {}]
bus: [1, 1, 2]
effects: ['OrderPlaced:o1', 'OrderPaid:o1']
```

<details><summary>Show solution and explanation</summary>

<pre><code>class Service:
    def __init__(self, name, fail_on=None):
        self.name, self.fail_on, self.ledger = name, fail_on, {}

    def do(self, order, action):
        if self.fail_on == action:
            raise RuntimeError(f&quot;{self.name} failed&quot;)
        self.ledger[order] = action

    def undo(self, order):
        self.ledger.pop(order, None)

def run_saga(order, steps, log):
    &quot;&quot;&quot;steps = [(service, action, compensation)]. Run in order; on failure compensate completed steps in reverse.&quot;&quot;&quot;
    done = []
    for svc, action, comp in steps:
        try:
            svc.do(order, action)
            log.append(f&quot;{svc.name}:{action}&quot;)
            done.append((svc, comp))
        except RuntimeError as e:
            log.append(f&quot;FAILED {e}&quot;)
            for s, c in reversed(done):
                s.undo(order)
                log.append(f&quot;{s.name}:{c}&quot;)
            return &quot;ROLLED_BACK&quot;
    return &quot;COMPLETED&quot;

class Outbox:
    &quot;&quot;&quot;Events are written with the state change; a relay publishes them at-least-once; consumers dedupe by id.&quot;&quot;&quot;
    def __init__(self): self.rows, self.next_id = [], 1
    def add(self, payload):
        self.rows.append({&quot;id&quot;: self.next_id, &quot;payload&quot;: payload, &quot;sent&quot;: False}); self.next_id += 1
    def relay(self, bus, crash_after_publish=False):
        for r in self.rows:
            if not r[&quot;sent&quot;]:
                bus.append((r[&quot;id&quot;], r[&quot;payload&quot;]))
                if crash_after_publish:
                    return                      # crashed before marking as sent -&gt; will be published again
                r[&quot;sent&quot;] = True

def consume(bus, processed, effects):
    for eid, payload in bus:
        if eid in processed:
            continue                            # idempotent consumer
        processed.add(eid); effects.append(payload)

def new_steps(fail_on=None):
    inv, pay, ship = Service(&quot;inventory&quot;), Service(&quot;payment&quot;, fail_on), Service(&quot;shipping&quot;)
    return [(inv, &quot;reserve&quot;, &quot;release&quot;), (pay, &quot;charge&quot;, &quot;refund&quot;), (ship, &quot;schedule&quot;, &quot;cancel&quot;)], (inv, pay, ship)

log = []; steps, svcs = new_steps()
print(run_saga(&quot;o1&quot;, steps, log), log)
log = []; steps, svcs = new_steps(fail_on=&quot;charge&quot;)
print(run_saga(&quot;o2&quot;, steps, log), log, [s.ledger for s in svcs])

ob, bus, processed, effects = Outbox(), [], set(), []
ob.add(&quot;OrderPlaced:o1&quot;); ob.add(&quot;OrderPaid:o1&quot;)
ob.relay(bus, crash_after_publish=True)          # relay crashes after publishing the first event
ob.relay(bus)                                    # restart: the first event is published again
consume(bus, processed, effects)
print(&quot;bus:&quot;, [e for e, _ in bus])
print(&quot;effects:&quot;, effects)
</code></pre>

<p>Sagas replace distributed transactions with local steps plus compensations; the outbox gives at-least-once publishing and consumers dedupe by event id. This is a teaching model. Verified by running the solution.</p>
</details>
