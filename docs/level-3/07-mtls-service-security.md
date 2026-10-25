---
description: "A real self-signed CA and real mTLS failures: a connection reset for no client cert, and for one signed by the wrong CA, versus success with the right one."
---

# mTLS & Service-to-Service Security

## What was run

A real self-signed CA, server certificate, and client certificate were generated with real `openssl` commands — not
mocked, not pre-provided. A real `uvicorn` server, `secure_service.py` (port 8443), was started with TLS enabled
*and* `--ssl-cert-reqs 2` (Python's `ssl.CERT_REQUIRED`), meaning it refuses any TLS connection that doesn't present
a client certificate signed by the trusted CA — real mutual TLS, not just a server certificate. Three real `curl`
attempts were made: no client cert, the correct client cert, and a self-signed cert from an entirely different,
untrusted CA standing in for an attacker.

## Generating a real CA and real certificates

```bash
openssl req -x509 -new -nodes -key ca.key -sha256 -days 365 -out ca.crt -subj "/CN=Demo-Internal-CA"
openssl req -new -key server.key -out server.csr -subj "/CN=localhost"
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -sha256
openssl req -new -key client.key -out client.csr -subj "/CN=orders-service-client"
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365 -sha256
```

## Three real handshakes, three real outcomes

```bash
# no client certificate at all
curl -v --cacert ca.crt https://localhost:8443/secrets
# ...SSL certificate verify ok (server cert is trusted)...
# * Recv failure: Connection reset by peer
# curl: (56) Recv failure: Connection reset by peer

# the correct client certificate, signed by our CA
curl -s --cacert ca.crt --cert client.crt --key client.key https://localhost:8443/secrets
# {"data":"this endpoint required a valid client certificate to reach"}

# a self-signed cert from an UNTRUSTED CA, standing in for an attacker with their own key pair
curl -s --cacert ca.crt --cert untrusted.crt --key untrusted.key https://localhost:8443/secrets -w "http:%{http_code}"
# http:000   (connection reset, same as the no-cert case)
```

Only one of these three attempts succeeded. Critically, checking the *server's own log* for all three attempts:

```bash
tail secure.log
# INFO:     127.0.0.1:55228 - "GET /secrets HTTP/1.1" 200 OK
```

**Only the successful request appears in the application log at all.** The two rejected handshakes never show up as
HTTP requests, because they were rejected during the TLS handshake itself — before any HTTP request could even be
parsed. This is a real, important security property: a service protected by mTLS doesn't just reject unauthorized
callers at the application layer (where request logging, rate limiting, and business logic live), it refuses the
*connection* before any of that code ever runs.

## How It Actually Works

Regular TLS (what every `https://` site uses) is one-directional trust: the client verifies the server's identity
(via its certificate, chained to a CA the client trusts), but the server has no cryptographic idea who the client
is beyond "someone who could complete a TLS handshake." **Mutual TLS (mTLS)** adds the other direction: the server
also demands and verifies a certificate from the client, chained to a CA *the server* trusts. In a microservices
system, this means every service can cryptographically prove its identity to every other service it calls, and
every service can refuse to talk to a caller that can't prove its identity this way — exactly what
`--ssl-cert-reqs 2` enforced above.

This is the real mechanism behind a service mesh's "automatic mTLS everywhere" promise, directly connected to
[Lesson 5](05-service-mesh-sidecar.md)'s sidecar pattern: in a real mesh, each sidecar proxy — not the application
itself — holds that service's certificate, issued and rotated automatically by the mesh's control plane, and
performs the mTLS handshake with every other sidecar it talks to. The application code (like this demo's
`secure_service.py`, which has zero TLS-related code in it) never has to know about certificates at all when a mesh
handles this layer — this lesson built the mechanism directly in the application layer specifically to make the
underlying cryptographic handshake visible and testable without needing a full mesh control plane.

The untrusted-cert test above demonstrates the real security property this provides: possessing *any* valid-looking
certificate isn't enough — it must be signed by a CA the server specifically trusts. An attacker who generates
their own key pair and self-signs a certificate (exactly what `untrusted.crt` is) gets rejected identically to
presenting no certificate at all, because `ca.crt` never signed it.

## Common mistakes

- Treating "TLS is enabled" as equivalent to "the caller's identity is verified" — ordinary one-directional TLS
  only verifies the *server's* identity; without mTLS, any client (including an attacker) can connect if they can
  simply trust the server's certificate chain, which is public information.
- Checking a certificate's CN/SAN matches without also verifying the full chain to a specifically trusted CA — a
  certificate that's internally well-formed and even self-consistent (as `untrusted.crt` is) is worthless for
  authentication unless it chains to a CA the verifier actually trusts, which this demo's CA bundle (`ca.crt`)
  deliberately does not include.
- Assuming a rejected handshake is "logged like any other denied request" — as shown above, it may not appear in
  application-level logs at all, which matters for incident investigation; TLS-layer rejection metrics (available
  from a proxy like Envoy, or the TLS library itself) are a separate, necessary signal.

## Exercise

Generate a *second* valid client certificate signed by the real CA (`ca.crt`), but with a CN of
`"payment-service-client"` instead of `"orders-service-client"`. Confirm it's accepted by `secure_service.py` just
like the first one (since the check here is only "signed by the trusted CA," not "which specific service is this").
Then modify `secure_service.py` to read the actual client certificate's CN from the request (FastAPI/Starlette
exposes this via the ASGI scope's TLS extension, or via `request.scope["client_cert"]` depending on server
configuration) and reject any CN other than `"orders-service-client"` — turning "any certificate this CA signed" into
real, specific service-identity authorization, not just trust-chain verification.
