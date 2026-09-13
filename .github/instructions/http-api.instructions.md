---
applyTo: "**/*Controller.java,**/*Resource.java,**/*Endpoint.java,**/openapi*.yaml,**/openapi*.yml,**/openapi*.json,docs/api/**/*.yaml,docs/api/**/*.yml,docs/api/**/*.json"
---

# HTTP API instructions

Human-readable source: `10-Devops-SOP/04-http-api-design-standard.md`.

Treat OpenAPI 3.1 as the contract source of truth. Update the contract, implementation, tests, and examples together.

## Resource design

Use stable resource-oriented URIs:

```text
GET    /v1/orders
POST   /v1/orders
GET    /v1/orders/{orderId}
PUT    /v1/orders/{orderId}
DELETE /v1/orders/{orderId}
POST   /v1/orders/{orderId}/cancellations
```

Do not expose implementation names:

```text
/v1/orderController/findOrderByDatabaseId
/v1/getOrders
```

- Use plural nouns and shallow nesting.
- Treat IDs as opaque strings.
- Never put credentials or personal data in a URI.
- `GET` and `HEAD` must not create, charge, update, or delete.
- Use `PUT` for complete replacement at a known URI.
- Use a documented `PATCH` format for partial updates.
- Use `POST` for creation or commands that cannot be modeled as replacement.

## Status codes

Use:

| Situation | Status |
|---|---:|
| Successful read | 200 |
| Created | 201 with `Location` |
| Asynchronous acceptance | 202 with status URL |
| Successful with no body | 204 |
| Malformed request | 400 |
| Missing or invalid authentication | 401 |
| Authenticated but forbidden | 403 |
| Not found or deliberately hidden | 404 |
| State or idempotency conflict | 409 |
| Failed `If-Match` | 412 |
| Unsupported media type | 415 |
| Invalid fields or business rules | 422 |
| Rate limited | 429 with `Retry-After` when possible |
| Internal failure | 500 without internal details |
| Temporary overload | 503 with `Retry-After` when possible |

Never return `200` for an error with `"success": false`.

## Error format

Use RFC 9457 `application/problem+json`:

```json
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Request validation failed",
  "status": 422,
  "detail": "One or more fields are invalid.",
  "instance": "https://api.example.com/problems/instances/01K51",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "errors": [
    {
      "pointer": "/customer/email",
      "code": "invalid_format",
      "message": "Must be a valid email address."
    }
  ]
}
```

- Use an organization-controlled stable absolute URI for `type`.
- Clients must branch on `type` or a stable code, never parse `title` or `detail`.
- Never expose stack traces, SQL, class names, file paths, hosts, or downstream response bodies.

## Data representation

- Use RFC 3339 with an offset for time points; prefer UTC `Z`.
- Use `Instant` or `OffsetDateTime` in Java.
- Do not use `"2026-09-13 16:48:10"` for a time point.
- Represent money with an exact decimal value and currency.
- Document meaning, unit, range, format, and nullability in OpenAPI.
- In OpenAPI 3.1, use `type: [string, "null"]`, not `nullable: true`.

## Idempotency and concurrency

- Preserve HTTP idempotent semantics for safe methods, `PUT`, and `DELETE`.
- Payment, order creation, refund, and similar `POST` operations must support an organization-defined `Idempotency-Key`.
- Persist key, request fingerprint, and result atomically in shared storage.
- The same key with the same request returns the original business result.
- The same key with a different request returns `409`.
- Important mutable resources must support ETag and `If-Match`.
- Return `412` for a stale ETag.
- Retry only documented transient failures with exponential backoff and jitter.

```http
POST /v1/payments HTTP/1.1
Idempotency-Key: 01K51PMVJ7N6X9Z2J4M8SQ5M3K
```

```http
PUT /v1/orders/ord_789 HTTP/1.1
If-Match: "ord_789-v17"
```

## Pagination, cache, and tracing

- Set a server-side maximum page size.
- Prefer opaque cursor pagination for large changing datasets.
- Bind the cursor to filters and a stable sort with a unique tie-breaker.
- Allowlist filter fields and operators; never expose SQL or JPQL as a filter.
- Define a cache policy for every `GET` and `HEAD`.
- Use `Cache-Control: no-store` for credentials, tokens, and highly sensitive responses.
- Propagate a valid W3C `traceparent`; create a new trace when absent or invalid.
- Never use a trace ID as authentication, authorization, or replay protection.

## Compatibility and tests

Treat removal, rename, type/unit/timezone/nullability changes, new required fields, narrower values, changed status semantics, and changed default ordering as breaking changes.

Before completion:

- Validate OpenAPI syntax.
- Check implementation and contract consistency.
- Run the breaking-change check.
- Add positive, validation, authorization, idempotency, timeout, retry, and stale-ETag tests as applicable.

