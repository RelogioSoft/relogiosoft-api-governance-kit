# Security

**Category:** Design
**Tags:** security, authentication, authorisation, oauth2, jwt, api-keys, cors, tls, rate-limiting, owasp, scopes

---

## Summary of Rules

### Authentication
- All API endpoints **MUST** require authentication unless explicitly designed for public access.
- Services **MUST** support Bearer token authentication via the `Authorization` header.
- API keys **MUST NOT** be passed in the URI or query string; they **MUST** be passed in a request header.
- Tokens **MUST NOT** be logged (see [Observability & Logging](./11-observability-and-logging.md)).

### Transport
- All API traffic **MUST** be served over HTTPS (TLS 1.2 minimum; TLS 1.3 preferred).
- Services **MUST NOT** accept plaintext HTTP connections for production endpoints.
- Services **SHOULD** respond to plaintext HTTP requests with `301 Moved Permanently` redirecting to the HTTPS equivalent, or drop the connection silently.

### Authorisation
- Authorisation decisions **MUST** be made after authentication is confirmed.
- A `403 Forbidden` response **MUST** be returned for any permission denial, regardless of whether the resource exists (prevents enumeration — see [HTTP Status Codes](./06-http-status-codes.md)).
- Authorisation scope **SHOULD** follow the principle of least privilege: request only the scopes needed.

### Input Validation
- Services **MUST** validate all inputs (path parameters, query parameters, headers, and request body) before processing.
- Identifier type-parse failures **MUST** return `403 Forbidden`, not `400 Bad Request` (see [Data Representations](./09-data-representations.md)).
- Services **MUST NOT** echo back raw unvalidated input in error messages where it could facilitate injection.

### CORS
- Services **MUST NOT** use `Access-Control-Allow-Origin: *` unless the endpoint is intentionally fully public and unauthenticated.
- CORS allowed-origin lists **SHOULD** be explicitly enumerated and maintained.
- Services **SHOULD NOT** reflect the `Origin` header value verbatim into `Access-Control-Allow-Origin` without validating it against an allowlist.

### Secrets
- API keys, client secrets, and tokens **MUST NOT** appear in API responses unless they are being issued for the first time (e.g. a token issuance endpoint).
- API keys **MUST NOT** be included in OpenAPI examples.
- Secrets **MUST NOT** be committed to source control.

---

## Authentication Strategies

### Bearer Token (OAuth 2.0 / OIDC) — Preferred

OAuth 2.0 access tokens are passed in the `Authorization` header using the Bearer scheme.

```http
GET /customers/123 HTTP/1.1
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Advantages:**
- Industry-standard, well-supported across frameworks and gateways.
- Tokens can carry claims (scopes, tenant, user identity) without a round trip to an identity store.
- Short-lived tokens limit the blast radius of token leakage.
- Supports delegation (a service acting on behalf of a user).

**When to use:** Any API that requires user or service identity. This is the default for Platform APIs, Gateway APIs, and Application Specific APIs.

### API Keys

An API key is a long-lived opaque credential issued to a specific client.

```http
GET /customers/123 HTTP/1.1
x-api-key: sk_live_AbCdEfGhIjKlMnOpQrStUvWx
```

**Rules:**
- API keys **MUST** be passed in the `x-api-key` header (not in the URI or query string).
- API keys **MUST** be rotatable without downtime.
- Compromised API keys **MUST** be revocable immediately.
- API keys **SHOULD** be scoped to specific operations or resources.

**Advantages:**
- Simple to implement and use.
- Suitable for server-to-server integrations where delegation is not required.

**Disadvantages:**
- Long-lived; leakage has a larger blast radius than short-lived Bearer tokens.
- Do not carry fine-grained identity claims.

**When to use:** B2B integrations, partner APIs, or service-to-service calls where OAuth flows are disproportionate to the use case.

### Mutual TLS (mTLS)

Both the client and server present certificates to authenticate each other.

**When to use:** High-security internal service meshes or regulated industries where client identity must be cryptographically verifiable at the transport layer.

---

## OAuth 2.0 Grant Types

| Grant Type | Use Case |
|-----------|---------|
| **Client Credentials** | Server-to-server (machine-to-machine) communication. No user involved. |
| **Authorization Code + PKCE** | User-facing applications (web, SPA, mobile). The user authenticates and consents. |
| **Device Code** | Input-constrained devices (smart TVs, IoT). |

**Rules:**
- The Implicit Grant and Password Grant **MUST NOT** be used; both are deprecated.
- PKCE (Proof Key for Code Exchange) **MUST** be used for all Authorization Code flows.
- `client_secret` **MUST NOT** be embedded in client-side (browser or mobile) application code.

---

## Token Handling

### Access Tokens

- Access tokens **SHOULD** be short-lived (typically 15 minutes to 1 hour).
- Services **MUST** validate token signature, expiry, audience (`aud`), and issuer (`iss`) on every request.
- Services **MUST NOT** accept tokens signed with symmetric keys shared across multiple services.

### OpenAPI Declaration

Declare security schemes in the OpenAPI Info Object and apply them to operations:

```yaml
components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: "OAuth 2.0 access token obtained from the identity provider."
    ApiKeyAuth:
      type: apiKey
      in: header
      name: x-api-key
      description: "API key issued via the developer portal."

security:
  - BearerAuth: []
```

To override at the operation level (e.g. a public health endpoint):

```yaml
/health:
  get:
    summary: Get service health
    security: []   # No auth required
```

---

## Authorisation and Scopes

### Scope-Based Access Control

OAuth 2.0 scopes constrain what a token holder is permitted to do. Scopes are declared on the token and validated by the service.

**Scope naming convention:**

```
{resource}:{action}
```

Examples:
- `customers:read` — read access to customer resources
- `customers:write` — create and update access to customer resources
- `orders:delete` — delete access to order resources

```yaml
# OpenAPI scope declaration
components:
  securitySchemes:
    BearerAuth:
      type: oauth2
      flows:
        clientCredentials:
          tokenUrl: https://auth.example.com/oauth/token
          scopes:
            customers:read: "Read customer data"
            customers:write: "Create and update customers"
            orders:read: "Read order data"
```

### Principle of Least Privilege

- Services **SHOULD** define the minimum scopes required for each operation.
- Clients **MUST** only request the scopes they need.
- Scopes that grant broad access (e.g. `admin`) **SHOULD NOT** be used where fine-grained scopes are viable.

---

## CORS (Cross-Origin Resource Sharing)

CORS is required when browsers make requests from one origin to a different origin. The server must explicitly allow this.

### When CORS Applies

CORS headers are only relevant to browser-based clients. Server-to-server communication does not use CORS.

### Response Headers

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PATCH, DELETE, OPTIONS
Access-Control-Allow-Headers: Authorization, Content-Type, x-conversation, x-api-key
Access-Control-Max-Age: 86400
```

- `Access-Control-Allow-Origin`: The origin(s) allowed to make requests. **MUST NOT** be `*` for authenticated endpoints.
- `Access-Control-Allow-Methods`: Permitted HTTP methods.
- `Access-Control-Allow-Headers`: Headers the browser is permitted to send.
- `Access-Control-Max-Age`: How long (seconds) the browser may cache a preflight response.

### Preflight Requests

Browsers send an `OPTIONS` preflight request before non-simple requests:

```http
OPTIONS /customers HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization, Content-Type
```

The service **MUST** respond to `OPTIONS` requests with the appropriate CORS headers and `200 OK` (or `204 No Content`). `OPTIONS` routes **MUST NOT** require authentication.

---

## Rate Limiting

Rate limiting protects services from abuse and denial-of-service conditions. The [HTTP Status Codes](./06-http-status-codes.md) document covers `429 Too Many Requests`; this section covers the response headers.

### Required Headers on 429 Responses

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
RateLimit-Limit: 100
RateLimit-Remaining: 0
RateLimit-Reset: 1719835200
```

| Header | Meaning |
|--------|---------|
| `Retry-After` | Seconds to wait before retrying (integer or HTTP date). **MUST** be present on 429 responses. |
| `RateLimit-Limit` | The maximum number of requests allowed in the current window. |
| `RateLimit-Remaining` | Remaining requests in the current window. |
| `RateLimit-Reset` | Unix timestamp when the window resets. |

### Proactive Headers

Services **SHOULD** include rate limit headers on all responses so clients can proactively back off:

```http
HTTP/1.1 200 OK
RateLimit-Limit: 100
RateLimit-Remaining: 87
RateLimit-Reset: 1719835200
```

### Rate Limiting Scope

Rate limits **SHOULD** be applied per client identity (client_id, API key, or IP address) rather than globally.

---

## OWASP API Security Top 10 — Governance Reference

The [OWASP API Security Top 10](https://owasp.org/API-Security/) identifies the most critical API security risks. The following table maps each risk to the governance rules that mitigate it.

| # | Risk | Mitigation |
|---|------|-----------|
| API1 | Broken Object Level Authorisation | Return `403` uniformly for denied access; validate that the authenticated identity owns the requested resource. |
| API2 | Broken Authentication | Require Bearer tokens; validate signature, expiry, audience, and issuer. Never accept unsigned or expired tokens. |
| API3 | Broken Object Property Level Authorisation | Do not expose properties the caller is not permitted to see. Use an anti-corruption layer (see [API First Principles](./01-api-first-principles.md)) to explicitly map responses. |
| API4 | Unrestricted Resource Consumption | Enforce pagination limits; apply rate limits per client identity. |
| API5 | Broken Function Level Authorisation | Check operation-level scopes; use fine-grained scopes, not broad admin tokens. |
| API6 | Unrestricted Access to Sensitive Business Flows | Rate-limit high-risk operations independently; require step-up authentication for sensitive actions. |
| API7 | Server-Side Request Forgery (SSRF) | Validate and allowlist URLs when the API makes server-side outbound calls based on client-supplied URIs. |
| API8 | Security Misconfiguration | Enforce HTTPS; restrict CORS origins; disable debug endpoints in production; keep TLS certificates current. |
| API9 | Improper Inventory Management | Apply lifecycle status (`x-status`) to all APIs; decommission deprecated endpoints (see [API Lifecycle](./02-api-lifecycle.md) and [Versioning](./03-api-versioning.md)). |
| API10 | Unsafe Consumption of APIs | Validate all responses from downstream services; do not assume upstream APIs are always well-formed. |

---

## Security Headers Reference

Include these headers on all API responses:

| Header | Recommended Value | Purpose |
|--------|------------------|---------|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Enforce HTTPS for 1 year. |
| `X-Content-Type-Options` | `nosniff` | Prevent MIME type sniffing. |
| `X-Frame-Options` | `DENY` | Prevent clickjacking (if API responses are rendered in browsers). |
| `Cache-Control` | See [Caching](./10-caching.md) | Prevent sensitive data caching by intermediaries. |
| `Content-Security-Policy` | `default-src 'none'` | Restrict browser resource loading (for API explorer / developer portal). |

**Note:** `X-Frame-Options` and `Content-Security-Policy` are browser-facing; they are less relevant for machine-to-machine APIs but **SHOULD** be included for consistency and defence in depth.
