---
description: "A real gRPC service and generated client, with a measured 6x latency advantage over REST/JSON on the same machine, plus a real NOT_FOUND error."
---

# gRPC for Service-to-Service Calls

## What was run

A real `.proto` contract compiled with `grpc_tools.protoc` into real generated Python stubs, a real gRPC server on
port 50051, and a real client calling it — including one call for an item that doesn't exist, producing a real
`NOT_FOUND` status. Then a real 200-call timing comparison against the REST `catalog_service.py` from
[Lesson 5](05-rest-contracts-versioning.md), both running locally.

## The contract, compiled, not just documented

```protobuf
service Inventory {
  rpc GetStock (StockRequest) returns (StockReply);
}
message StockRequest { int32 item_id = 1; }
message StockReply { int32 item_id = 1; int32 quantity_available = 2; }
```

```bash
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. inventory.proto
# generates inventory_pb2.py and inventory_pb2_grpc.py
```

Unlike the REST contract in Lesson 5 — which lived only as convention and documentation until a client broke against
it — this contract is compiled into real Python classes. A client calling `inventory_pb2.StockRequest(item_id="x")`
with a string instead of an int fails immediately, locally, with a type error, before any network call happens.

```bash
python client.py 1
# item_id=1 quantity_available=42
python client.py 2
# item_id=2 quantity_available=0
python client.py 999
# RPC failed: code=StatusCode.NOT_FOUND details=item 999 not found
```

That `NOT_FOUND` is a real gRPC status code (`context.set_code(grpc.StatusCode.NOT_FOUND)` on the server), the gRPC
equivalent of an HTTP 404 — but delivered as a structured, typed error the client catches as `grpc.RpcError`, not a
JSON body the client has to parse and hope matches a documented error shape.

## The real latency difference

```bash
# 200 gRPC calls: 0.0300s total, 0.150ms/call
# 200 REST calls: 0.1799s total, 0.899ms/call
```

On this machine, for this tiny payload, gRPC was about **6x faster per call** than REST/JSON over HTTP/1.1. That gap
comes from two real, specific mechanisms, not "gRPC is just faster":

1. **Binary Protocol Buffers vs. text JSON.** Encoding `{"item_id": 1, "quantity_available": 42}` as JSON means
   parsing a text format with field names repeated on every message. Protobuf encodes the same data as a compact
   binary blob with field *numbers*, not names, and a schema both sides already agree on.
2. **HTTP/2 multiplexing vs. HTTP/1.1.** gRPC runs on HTTP/2, which can multiplex many requests over one connection;
   the REST call here used `httpx.Client()`, which reuses a connection but still pays HTTP/1.1's per-request
   overhead.

## How It Actually Works

gRPC is built on three pieces that work together: Protocol Buffers for the wire format and schema, HTTP/2 for
transport, and generated client/server code so neither side hand-writes serialization logic. The schema
(`inventory.proto`) is the single source of truth both services compile against — a genuine advantage over REST,
where the "schema" is usually just a shared understanding (or an OpenAPI doc that can silently drift from the real
behavior).

This makes gRPC a strong default for **internal, service-to-service** calls where both ends are services you
control and can recompile together: the type safety and performance both pay off, and nobody outside your system
needs to read the wire format directly. It's a weaker fit for **public-facing APIs**: browsers can't easily call
gRPC without a proxy layer (grpc-web), and external developers generally expect to read and test a JSON/REST API with
`curl` or Postman without installing your `.proto` files and a code generator first. Many real systems use both:
gRPC between internal services, REST (or GraphQL) at the edge facing browsers and third parties — which is exactly
why the [API Gateway pattern](07-api-gateway-pattern.md) in the next lesson matters: something has to translate
between the two worlds.

## Common mistakes

- Choosing gRPC for a public API because it's faster, then discovering browsers and casual API consumers can't call
  it directly without extra infrastructure (grpc-web, a REST-to-gRPC gateway).
- Changing a `.proto` field's number or type instead of only adding new fields with new numbers — protobuf's wire
  compatibility rules are stricter and less forgiving than JSON's "extra fields are ignored," and a mismatched field
  number silently reads the wrong data instead of erroring.
- Assuming the 6x speedup measured here holds at every payload size — for very large payloads, serialization format
  matters less than the payload size itself, and the gap narrows.

## Exercise

Add a second RPC to `inventory.proto`, `UpdateStock(UpdateStockRequest) returns (StockReply)`, that decrements
`quantity_available` for a given `item_id`. Regenerate the stubs, implement the server method, and write a client
call that decrements item 1's stock twice, printing the quantity after each call. Then try calling `UpdateStock` on
item 999 (which doesn't exist) and confirm you get the same kind of structured `grpc.RpcError` shown above, not a
Python crash.
