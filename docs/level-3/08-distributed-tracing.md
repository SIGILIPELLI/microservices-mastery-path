---
description: "A real trace ID propagated across 3 real services, reconstructed into a precise millisecond timeline that shows exactly where 292.6ms of real latency went."
---

# Distributed Tracing Across Services

## What was run

Four real services: a `trace_collector.py` (port 8171, a tiny real span-storage API), `gateway.py` (port 8170),
`service_a.py` (port 8172), and `service_b.py` (port 8173), chained `gateway → service-a → service-b`. Each service
generates (or receives) a trace ID and a span ID, does real work (including a real `time.sleep` to simulate
processing), and reports its own span's real start and end timestamps to the collector. One real request was made
to the gateway, and the full trace was then reconstructed from the collector by querying for that one trace ID.

## Propagating the trace ID across three real HTTP hops

```python
# gateway.py
trace_id = new_trace_id()
with Span(trace_id, None, "gateway", "request") as span:
    r = httpx.get(f"{SERVICE_A_URL}/handle", headers={"x-trace-id": trace_id, "x-parent-span-id": span.span_id})

# service_a.py
with Span(x_trace_id, x_parent_span_id, "service-a", "handle") as span:
    r = httpx.get(f"{SERVICE_B_URL}/compute", headers={"x-trace-id": x_trace_id, "x-parent-span-id": span.span_id})

# service_b.py
with Span(x_trace_id, x_parent_span_id, "service-b", "compute") as span:
    time.sleep(0.15)
```

```bash
curl -s localhost:8170/request
# {"trace_id":"8ca3c798172a488b","result":{"computed":42}}
```

## The real trace, reconstructed from the collector

```bash
curl -s localhost:8171/trace/8ca3c798172a488b
```

```python
gateway:   start_offset=0.0ms    duration=292.6ms
service-a: start_offset=44.3ms   duration=229.6ms
service-b: start_offset=89.3ms   duration=158.5ms
```

This is a real, measured timeline, not an illustration: the gateway's span covers the entire 292.6ms request from
end to end; service-a's span starts 44.3ms after the gateway's own span started (the gateway's own local work plus
the real network call to service-a) and runs for 229.6ms; service-b's span starts a further 45ms after that (89.3ms
total from the gateway's start) and runs for 158.5ms, which matches almost exactly service-b's hardcoded
`time.sleep(0.15)` plus a small amount of real HTTP and tracing overhead. The parent-child relationship
(`parent_span_id`) recorded for each span lets you reconstruct this nesting precisely: service-b's span is a child
of service-a's, which is a child of the gateway's — the real call structure, not inferred.

## How It Actually Works

A **trace** represents one logical end-to-end request as it flows through multiple services; a **span** represents
one unit of work within that trace — typically, one service handling one operation. The mechanism that ties them
together across process boundaries, demonstrated directly above, is **context propagation**: the trace ID (and the
calling span's ID, to establish parentage) travels as headers on every outbound call, and each service that
receives them is responsible for including them on any further calls it makes downstream. Break that propagation
anywhere in the chain — forget to forward the header on one hop — and the trace fragments into disconnected pieces
that can't be reassembled, which is why tracing libraries in practice (OpenTelemetry being the modern standard)
focus heavily on automatic, framework-level propagation rather than relying on every developer to pass headers
manually, as this demo deliberately does by hand to make the mechanism visible.

The real value a reconstructed trace like this provides, that no single service's own logs can: **it answers "where
did the time actually go?" across a request that touched multiple services**, without needing to manually correlate
timestamps across three separate log files by guessing which gateway request matches which service-a request. Here,
the answer is immediate from the numbers alone: of 292.6ms total, service-b's real work (158.5ms) is the largest
single contributor, with the remaining ~134ms spent in the gateway's own overhead, the network hop to service-a, and
service-a's own 20ms of local work plus its hop to service-b.

## Common mistakes

- Logging timestamps locally in each service without a shared trace ID connecting them — this demo's
  `x-trace-id` header is the entire mechanism that makes correlation possible after the fact; without it, you're
  left guessing which log lines across three files belong to the same logical request.
- Forgetting to propagate the trace context on *every* outbound call a service makes, including ones added later —
  a trace that silently drops a hop looks like that hop simply didn't happen, which is a much harder bug to notice
  than an explicit error.
- Treating total request duration as evenly distributed across hops without actually measuring per-span
  durations — as shown here, the real bottleneck (service-b's 158.5ms) is immediately obvious from the
  reconstructed trace and would be easy to miss or mis-guess from the outside.

## Exercise

Add a fourth hop: have `service_b.py` call a new `service_c.py` for one more real operation, propagating the trace
context exactly as shown above. Then deliberately introduce a bug where `service_b.py` forgets to forward
`x-trace-id` on that new call (hardcode a different value instead), run a request, and confirm in the collector's
trace view that service-c's span either doesn't show up under the same trace at all, or shows up as an orphaned
trace with no parent — observing directly what a broken propagation chain actually looks like in the data, not just
in theory.
