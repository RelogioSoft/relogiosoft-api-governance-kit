# Request & Response Headers

**Category:** Design
**Tags:** headers, content-type, accept, authorization, correlation, custom-headers, train-case, request-headers, response-headers

---

## Summary of Rules

- Header names **MUST** use `Train-Case` (e.g. `Content-Type`, `Cache-Control`, `x-conversation`).
- Custom headers **MUST** be prefixed with `x-` to distinguish them from standard HTTP headers.
- The `Content-Type` header **MUST** be set on all requests and responses that have a body.
- The `Accept` header **SHOULD** be set on requests where the client cares about the response format.
- The `x-conversation` header **MUST** be used to propagate a correlation ID across service boundaries (see [Observability & Logging](./11-observability-and-logging.md)).
- Sensitive values (tokens, keys) **MUST NOT** be placed in URI query parameters; they **MUST** be placed in headers.
- The `Authorization` header **MUST** use the `Bearer` scheme for OAuth 2.0 tokens (see [Security](./13-security.md)).
- Responses **MUST** include `Cache-Control` on all `GET` responses (see [Caching](./10-caching.md)).
- Services **SHOULD** return `Content-Length` on all responses with a fixed-size body.
- Services **MUST** return `Location` on `201 Created` and `202 Accepted` responses.
- Services **MUST** include a `Vary` header when multiple representations can be returned for the same URI.
- Deprecated endpoints **MUST** include `Warning` and `Sunset` headers (see [Versioning](./03-api-versioning.md)).

---

## Casing Convention

All header names **MUST** use `Train-Case`: each word capitalised, separated by hyphens.

```
✅  Content-Type
✅  Authorization
✅  Cache-Control
✅  x-conversation       ← custom headers retain x- prefix in lowercase
✅  x-api-key

❌  content-type
❌  contentType
❌  CONTENT-TYPE
```

---

## Standard Request Headers

### Headers Required on Every Request

| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes (authenticated endpoints) | Bearer token or API key. See [Security](./13-security.md). |
| `Content-Type` | Yes (requests with a body) | Media type of the request body. See below. |
| `Accept` | Recommended | Media type(s) the client can process. Defaults to `application/json`. |
| `x-conversation` | Yes (for traced operations) | Correlation ID propagated across service boundaries. UUID format. |

### Headers Used Where Applicable

| Header | When to Use | Example |
|--------|-------------|---------|
| `Accept-Encoding` | When the client supports compressed responses | `Accept-Encoding: gzip, deflate` |
| `Accept-Language` | When the client requires localised content | `Accept-Language: en-GB, de;q=0.8` |
| `If-None-Match` | Conditional GET with ETag | `If-None-Match: "abc123"` |
| `If-Modified-Since` | Conditional GET with Last-Modified | `If-Modified-Since: Mon, 01 Jan 2024 12:00:00 GMT` |
| `If-Match` | Conditional PUT/PATCH to prevent lost updates | `If-Match: "abc123"` |
| `If-Unmodified-Since` | Conditional PUT/PATCH based on timestamp | `If-Unmodified-Since: Mon, 01 Jan 2024 12:00:00 GMT` |
| `Idempotency-Key` | Safe retry of non-idempotent operations | `Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000` |
| `x-api-key` | API key authentication | `x-api-key: sk_live_AbCd...` |

---

## Content-Type and Accept

### Content-Type on Requests

Set `Content-Type` whenever the request has a body:

```http
POST /customers HTTP/1.1
Content-Type: application/json

{ "name": "Acme Corp" }
```

For PATCH using JSON Patch format:

```http
PATCH /customers/123 HTTP/1.1
Content-Type: application/json-patch+json

[{ "op": "replace", "path": "/name", "value": "Acme Corp Ltd" }]
```

For file uploads:

```http
POST /documents HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
```

### Accept on Requests

The `Accept` header tells the server what media type the client can process. Omitting it implies `application/json`.

```http
GET /customers/123 HTTP/1.1
Accept: application/json
```

For content-type version negotiation:

```http
GET /customers/123 HTTP/1.1
Accept: application/json;v=2
```

See [API Versioning](./03-api-versioning.md) for the full versioning protocol.

### Content-Type on Responses

The server **MUST** set `Content-Type` on all responses with a body:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{ "id": "123", "name": "Acme Corp" }
```

---

## Standard Response Headers

### Headers Required on All Responses

| Header | Required | Description |
|--------|----------|-------------|
| `Content-Type` | Yes (responses with body) | Media type of the response body. |
| `Cache-Control` | Yes (all GET responses) | Caching directive. See [Caching](./10-caching.md). |

### Headers Required on Specific Responses

| Header | Required On | Example |
|--------|------------|---------|
| `Location` | `201 Created`, `202 Accepted`, `301`, `302`, `307`, `308` | `Location: /customers/123` |
| `Retry-After` | `429 Too Many Requests`, `503 Service Unavailable` | `Retry-After: 30` |
| `Warning` | Deprecated endpoints | `Warning: 299 - "This resource is deprecated and will be removed on 2025-12-31."` |
| `Sunset` | Deprecated endpoints | `Sunset: Tue, 31 Dec 2025 23:59:59 GMT` |
| `Link` | Deprecated endpoints (successor), paginated responses | `Link: </v2/customers>; rel="successor-version"` |
| `ETag` | GET responses supporting conditional requests | `ETag: "abc123"` |
| `Last-Modified` | GET responses supporting conditional requests | `Last-Modified: Mon, 01 Jan 2024 12:00:00 GMT` |
| `Vary` | Responses that differ by `Accept`, `Accept-Encoding`, etc. | `Vary: Accept` |
| `Content-Encoding` | Compressed response bodies | `Content-Encoding: gzip` |
| `Content-Length` | Fixed-size response bodies | `Content-Length: 1234` |

### Security Headers

Include these headers on all responses (typically set at the gateway or load balancer level):

| Header | Value | Purpose |
|--------|-------|---------|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Force HTTPS. |
| `X-Content-Type-Options` | `nosniff` | Prevent MIME-type sniffing. |

See [Security](./13-security.md) for the full security headers reference.

---

## The x-conversation Correlation Header

The `x-conversation` header carries a shared trace identifier across the full request chain. It connects HTTP calls, message broker events, and async operations into a single traceable unit.

```http
GET /orders/456 HTTP/1.1
x-conversation: a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

**Rules:**
- If a service receives a request **with** `x-conversation`, it **MUST** forward the same value to all downstream HTTP calls.
- If a service receives a request **without** `x-conversation`, it **SHOULD** generate a new UUID and use it for all downstream calls.
- The `x-conversation` value **MUST** be logged as `TraceId` (see [Observability & Logging](./11-observability-and-logging.md)).
- The value **MUST** be a valid UUID.
- The `x-conversation` header **SHOULD** be echoed back in the response so clients can correlate errors.

See [Observability & Logging](./11-observability-and-logging.md) for the complete telemetry chain diagram.

---

## Custom Headers

### Naming

Custom headers **MUST** be prefixed with `x-` and use lowercase `kebab-case`:

```
✅  x-conversation
✅  x-api-key
✅  x-request-id
✅  x-tenant-id

❌  Conversation
❌  X-Conversation     ← omit capitalisation after x-
❌  myHeader
```

### Common Custom Headers

| Header | Direction | Purpose |
|--------|-----------|---------|
| `x-conversation` | Request & Response | Distributed trace correlation ID. |
| `x-api-key` | Request | API key credential. |
| `x-request-id` | Request & Response | Unique identifier for a single HTTP request (narrower scope than `x-conversation`). |
| `x-tenant-id` | Request | Tenant identifier when tenancy cannot be expressed in the URI. |

### When Not to Use Custom Headers

Do **not** use custom headers to pass information that belongs in:
- The URI (resource identity).
- The query string (filtering, pagination, field selection).
- The request body (business data).

Custom headers are appropriate for **cross-cutting, infrastructural concerns** that apply across many operations: tracing, API keys, tenant identity, and similar.

---

## Conditional Requests

Conditional requests allow clients to avoid re-downloading unchanged resources and protect against lost updates.

### Avoiding Unnecessary Downloads (Read)

Client stores the `ETag` from a previous response and sends it on the next request:

```http
GET /products/789 HTTP/1.1
If-None-Match: "abc123"
```

If the resource has not changed, the server returns `304 Not Modified` with no body.

### Preventing Lost Updates (Write)

Client sends the `ETag` of the version it last read. The server rejects the update if the resource has changed since:

```http
PATCH /products/789 HTTP/1.1
Content-Type: application/json-patch+json
If-Match: "abc123"

[{ "op": "replace", "path": "/price", "value": 4999 }]
```

If the resource has been modified since `"abc123"`, the server returns `412 Precondition Failed`.

See [Caching](./10-caching.md) for the full ETag and conditional request reference.

---

## Vary Header

The `Vary` header tells caches that the response may differ based on specific request headers. Without it, a cache may return the wrong version to a different client.

```http
HTTP/1.1 200 OK
Content-Type: application/json;v=2
Cache-Control: public, max-age=300
Vary: Accept
```

**Rules:**
- **MUST** be set when content-type versioning is in use (`Accept` header determines version).
- **SHOULD** be set when `Accept-Encoding` is supported (gzip compression).
- **SHOULD** be set when `Accept-Language` produces different response content.

---

## Header Reference by Verb

| Verb | Typical Request Headers | Typical Response Headers |
|------|------------------------|-------------------------|
| `GET` | `Authorization`, `Accept`, `x-conversation`, `If-None-Match`, `If-Modified-Since` | `Content-Type`, `Cache-Control`, `ETag`, `Vary`, `Last-Modified` |
| `POST` (create) | `Authorization`, `Content-Type`, `x-conversation`, `Idempotency-Key` | `Location`, `Content-Type` |
| `POST` (async) | `Authorization`, `Content-Type`, `x-conversation` | `Location` (`202`) |
| `PUT` | `Authorization`, `Content-Type`, `x-conversation`, `If-Match` | `Content-Type` or `(empty 204)` |
| `PATCH` | `Authorization`, `Content-Type: application/json-patch+json`, `x-conversation`, `If-Match` | `Content-Type` or `(empty 204)` |
| `DELETE` | `Authorization`, `x-conversation`, `If-Match` | `(empty 204)` |
| `OPTIONS` | `Origin`, `Access-Control-Request-Method`, `Access-Control-Request-Headers` | `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods` |
