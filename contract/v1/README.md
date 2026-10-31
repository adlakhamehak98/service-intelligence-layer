# SIL Contract v1

This is the standard every service and every SIL component follows. The gateway enforces it from outside a service, and SDKs implement it from inside. Any service in any language can follow it directly.

The key words MUST, SHOULD and MAY are used as described in RFC 2119.

## 1. Error response

Every error response MUST use the media type `application/problem+json` and follow [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) (Problem Details for HTTP APIs), extended with the fields below. The machine-readable schema is [`error.schema.json`](error.schema.json).

```json
{
  "type": "urn:sil:error:ORD-DEP-0007",
  "title": "Payment service unavailable",
  "status": 503,
  "detail": "Could not authorize payment for order 8812",
  "instance": "/orders",
  "errorCode": "ORD-DEP-0007",
  "category": "DEPENDENCY",
  "service": "order-service",
  "transactionId": "txn-5f2c9a1e4b7d4c21",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-10-31T10:15:30Z",
  "rootCause": { "service": "payment-service", "errorCode": "PAY-TMO-0002" },
  "errors": []
}
```

| Field | Required | Meaning |
| --- | --- | --- |
| `type` | yes | `urn:sil:error:` followed by the error code. |
| `title` | yes | Short, stable summary of the error code. The same code always has the same title. |
| `status` | yes | The HTTP status, repeated in the body. |
| `detail` | no | Human-readable explanation of this occurrence. MUST NOT contain stack traces, SQL, file paths or secrets. |
| `instance` | no | The request path. |
| `errorCode` | yes | See section 2. |
| `category` | yes | See section 3. MUST match the category in `errorCode`. |
| `service` | yes | Name of the service that produced this response. |
| `transactionId` | yes | Same value as the `X-Transaction-Id` response header. |
| `traceId` | no | W3C trace ID, when tracing is active. |
| `timestamp` | yes | When the error happened, ISO 8601 in UTC. |
| `rootCause` | no | Where a cascading failure started. See section 4. |
| `errors` | no | Field-level problems, for `VALIDATION` errors. Each entry has `field` and `message`. Entries MUST NOT echo the rejected value, since it may hold personal data. |

Stack traces MUST NOT appear anywhere in a response body. They belong in logs.

## 2. Error codes

Format: `SERVICE-CATEGORY-NNNN`, matching `^[A-Z]{2,6}-(VAL|ATN|ATZ|NFD|CON|RTL|DEP|TMO|INT)-[0-9]{4}$`.

- `SERVICE` is a short prefix owned by one service, such as `ORD` for order-service. It tells you which service failed without opening a log.
- `CATEGORY` is the three-letter code from section 3.
- `NNNN` is a number the service assigns. `0000` is reserved for generic errors produced by SIL itself (for example `ORD-INT-0000` for an unexpected exception).

Each service lists its codes in an error catalog, `sil-errors.yaml`:

```yaml
service: order-service
prefix: ORD
errors:
  - code: ORD-DEP-0007
    title: Payment service unavailable
    runbook: https://wiki.example.com/runbooks/ord-dep-0007
```

## 3. Categories

| Category | Code | HTTP status | Stack trace logged by default |
| --- | --- | --- | --- |
| VALIDATION | VAL | 400 | No |
| AUTHENTICATION | ATN | 401 | No |
| AUTHORIZATION | ATZ | 403 | No |
| NOT_FOUND | NFD | 404 | No |
| CONFLICT | CON | 409 | No |
| RATE_LIMITED | RTL | 429 | No |
| DEPENDENCY | DEP | 502 or 503 | Yes |
| TIMEOUT | TMO | 504 | Yes |
| INTERNAL | INT | 500 | Yes |

A service MAY override the status for a specific error (for example 405 or 415 for a `VALIDATION` error) when a more precise status exists. The category never changes.

## 4. Root cause

When a service fails because a service it called failed, it MUST report where the failure started:

- If the downstream response carries a `rootCause`, copy it unchanged.
- Otherwise, set `rootCause` to the downstream's own `service` and `errorCode`.
- If the downstream sent no contract-format body (a timeout, a refused connection, a non-SIL service), set `rootCause.service` to the downstream host and leave out `errorCode`.

This keeps the origin of a failure visible however long the call chain is.

## 5. Headers

| Header | Rule |
| --- | --- |
| `X-Transaction-Id` | Read from the request. If missing or invalid, generate one. Echo it on every response and forward it on every downstream call. Valid values match `^[A-Za-z0-9._-]{1,64}$`. Anything else is replaced, so it can never inject into logs. |
| `traceparent` | W3C Trace Context, so traces join across services and languages. |
| `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` | Sent on every rate-limited route. |
| `Retry-After` | Sent with 429 and 503 responses. |

## 6. Telemetry

Every request through a gateway produces one QoS event: service, route, method, status, outcome, duration and consumer. Outcome is one of `success`, `client_error`, `server_error`, `timeout` or `rejected`.

Metrics use the `sil_` prefix and low-cardinality labels only: service, route, method, status class and outcome. Transaction IDs and user IDs go in logs and traces, never in metric labels.

## 7. Conformance

[`conformance/cases.json`](conformance/cases.json) lists the checks every SDK must pass. Each SDK runs them against a small fixture app that exposes the endpoints described in that file. The Java runner lives in `sil-conformance`.
