# API Governance Kit

## Purpose

This kit provides canonical rules, rationale, and examples for designing, building, and validating REST/HTTP APIs. It is structured for use as a reference corpus by an API Governance Assistant (RAG), as well as for human engineers designing or reviewing APIs.

Every rule is graded using RFC 2119 language:

| Keyword | Meaning |
|---------|---------|
| **MUST** / **MUST NOT** | Absolute requirement / absolute prohibition |
| **SHOULD** / **SHOULD NOT** | Strong recommendation; deviation requires documented justification |
| **MAY** | Optional; use when it adds value |

---

## Document Index

| # | File | Topics Covered |
|---|------|---------------|
| 01 | [API First Principles](./01-api-first-principles.md) | API taxonomy, contract-first design, composition strategies, anti-corruption layer |
| 02 | [API Lifecycle & Status](./02-api-lifecycle.md) | draft / stable / unstable / deprecated / obsolete states, OpenAPI x-status |
| 03 | [API Versioning](./03-api-versioning.md) | Breaking vs non-breaking changes, content-type negotiation, deprecation headers, sunset process |
| 04 | [URI Design](./04-uri-design.md) | Resource naming, kebab-case, collections, singletons, tenancy, application-specific prefixes |
| 05 | [HTTP Verbs](./05-http-verbs.md) | GET / POST / PUT / PATCH / DELETE rules, idempotence, JSON Patch |
| 06 | [HTTP Status Codes](./06-http-status-codes.md) | 2xx / 3xx / 4xx / 5xx selection flowcharts, 204 vs 404, 401 vs 403, 400 vs 422 |
| 07 | [Error Handling](./07-error-handling.md) | Error response schema, fault/traceId/errorCode format, security guidance |
| 08 | [Querying & Filtering](./08-querying-and-filtering.md) | Pagination (offset & cursor), sorting, filtering approaches, field selection, count |
| 09 | [Data Representations](./09-data-representations.md) | Dates, times, durations, money, geospatial, identifiers, phone numbers, email addresses |
| 10 | [Caching](./10-caching.md) | Cache-Control directives, ETags, CAP/PACELC theorems, eventual consistency |
| 11 | [Observability & Logging](./11-observability-and-logging.md) | Structured log schema, correlation IDs, telemetry chaining, GDPR constraints |
| 12 | [Documentation Standards](./12-documentation-standards.md) | Grammar, casing, summary naming conventions, description requirements |
| 13 | [Security](./13-security.md) | Authentication (Bearer/API key/mTLS), OAuth 2.0, CORS, TLS, rate limiting, OWASP API Top 10 |
| 14 | [Request & Response Headers](./14-request-response-headers.md) | Standard headers, custom x- headers, Content-Type, Accept, Vary, correlation header |
| 15 | [Idempotency](./15-idempotency.md) | Idempotency-Key header, safe retries, server deduplication, client retry pattern |
| 16 | [Webhooks & Async Patterns](./16-webhooks-and-async-patterns.md) | Webhook delivery, HMAC signing, async job pattern, event naming, polling |
| 17 | [Health Endpoints](./17-health-endpoints.md) | /health, /health/live, /health/ready, Kubernetes probes, health response schema |
| 18 | [Batch Operations](./18-batch-operations.md) | Batch create/update/delete, 207 Multi-Status, atomicity, size limits, per-item results |

---

## Quick Rule Reference

### Design Rules
- Every service **MUST** expose an API.
- The API contract **MUST** be agreed before writing any code (contract-first).
- Use an **anti-corruption layer** between domain entities and API resources.
- APIs **MUST** use HTTP + JSON for interoperability outside the service perimeter.
- Resources **MUST** use `kebab-case` URIs per RFC 3986.

### HTTP Method Rules
- Only GET, POST, PUT, PATCH, DELETE are permitted.
- **PATCH** is preferred for partial resource updates; **MUST** use JSON Patch format.
- **PUT** **MUST NOT** be used unless idempotence is guaranteed.
- **GET** **MUST NOT** change state.

### Status Code Rules
- Use only RFC 9110 + RFC 6585 status codes (see exclusion list in doc 06).
- Use `200` for empty list results, not `204`.
- Use `204` for valid responses with no body (e.g., void PUT/POST).
- Use `403` for both "no permission" and "resource doesn't exist but shouldn't be revealed" (prevents enumeration).
- Use `422` exclusively for JSON PATCH validation failures.
- Use `207 Multi-Status` for batch operations with mixed outcomes.

### Error Response Rules
- All 4xx responses **MUST** return a `fault` object with `faultId` and `traceId`.
- 5xx responses **MUST NOT** return implementation details; only `faultId`, `traceId`, and `"Internal Server Error"`.
- Error codes **MUST** be documented on the developer portal.

### Versioning Rules
- Clients **MUST** be tolerant readers.
- Prefer **content-type negotiation** (`application/json;v=1`) over URI versioning.
- Use `Warning` and `Sunset` headers to signal deprecation.
- After sunset: return `301` (if replacement exists) or `410` (if no replacement).

### Caching Rules
- **MUST** include `Cache-Control` header on all GET responses.
- **MUST NOT** use both `Cache-Control` and `Expires` simultaneously.

### Data Format Rules
- Dates: ISO 8601 `YYYY-MM-DD`
- DateTimes: ISO 8601 UTC `YYYY-MM-DDThh:mm:ssZ`
- Timezones (named): IANA format e.g. `Europe/London`
- Money: integer in minor currency unit (cents), ISO 4217 currency code
- Phone numbers: string, E.164 format
- Identifiers: always `string` type

### Security Rules
- All endpoints **MUST** require authentication unless explicitly public.
- All traffic **MUST** be served over HTTPS (TLS 1.2 minimum).
- Use Bearer token (OAuth 2.0) as the default authentication mechanism.
- API keys **MUST** be passed in the `x-api-key` header, never in the URI.
- CORS `Access-Control-Allow-Origin: *` **MUST NOT** be used on authenticated endpoints.
- `403 Forbidden` **MUST** be returned for permission denial regardless of resource existence.

### Header Rules
- Header names **MUST** use `Train-Case`; custom headers **MUST** use `x-` prefix.
- `x-conversation` **MUST** be used to propagate correlation IDs across service boundaries.
- `Location` **MUST** be returned on `201 Created` and `202 Accepted` responses.

### Idempotency Rules
- Non-idempotent POST endpoints with significant side effects **SHOULD** support `Idempotency-Key`.
- The `Idempotency-Key` value **MUST** be a client-generated UUID.
- On duplicate key receipt, servers **MUST** return the original response without re-executing.

### Webhook Rules
- Webhook payloads **MUST** include `eventId`, `eventType`, and `occurredAt`.
- Webhook deliveries **MUST** be signed with HMAC-SHA256; receivers **MUST** verify signatures.
- Webhook deliveries **MUST** be retried with exponential backoff on non-2xx responses.

### Health Endpoint Rules
- Every service **MUST** expose a `/health` endpoint.
- `/health` **MUST NOT** require authentication.
- `/health` **MUST** return `503` when the service cannot serve traffic.

### Batch Operation Rules
- Batch endpoints **MUST** use a dedicated `/batch` sub-resource URI.
- Mixed-outcome batches **MUST** return `207 Multi-Status` with per-item results.
- Maximum batch size **MUST** be documented and enforced.

---

## How to Use This Kit (for RAG Assistants)

When validating or generating an API specification, apply rules in this priority order:

1. **URI design** — Is the resource correctly named and structured? (doc 04)
2. **HTTP verb** — Is the correct method used? (doc 05)
3. **Status codes** — Are responses correctly coded? (doc 06)
4. **Error format** — Do error responses follow the schema? (doc 07)
5. **Data types** — Are data formats correct? (doc 09)
6. **Versioning** — Is versioning and deprecation handled? (doc 03)
7. **Querying** — Does the collection endpoint support pagination? (doc 08)
8. **Caching** — Is Cache-Control set on GET responses? (doc 10)
9. **Documentation** — Are all descriptions present and correctly formatted? (doc 12)
10. **Lifecycle** — Is the API status declared? (doc 02)
11. **Security** — Is authentication declared? Are scopes defined? (doc 13)
12. **Headers** — Are standard headers present and correctly named? (doc 14)
13. **Idempotency** — Do mutating POST endpoints support Idempotency-Key? (doc 15)
14. **Health** — Does the service expose /health? (doc 17)

---

## Related Standards Referenced

- [RFC 9110](https://datatracker.ietf.org/doc/html/rfc9110) — HTTP Semantics
- [RFC 6585](https://datatracker.ietf.org/doc/html/rfc6585) — Additional HTTP Status Codes
- [RFC 6902](https://datatracker.ietf.org/doc/html/rfc6902) — JSON Patch
- [RFC 5789](https://datatracker.ietf.org/doc/html/rfc5789) — PATCH Method
- [RFC 3986](https://datatracker.ietf.org/doc/html/rfc3986) — URI Syntax
- [RFC 7946](https://datatracker.ietf.org/doc/html/rfc7946) — GeoJSON
- [RFC 8594](https://datatracker.ietf.org/doc/html/rfc8594) — Sunset HTTP Header
- [RFC 7234](https://datatracker.ietf.org/doc/html/rfc7234) — Warning Header
- [RFC 9457](https://datatracker.ietf.org/doc/html/rfc9457) — Problem Details (reference only)
- [RFC 4918](https://datatracker.ietf.org/doc/html/rfc4918) — 207 Multi-Status (WebDAV)
- [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) — Date and Time
- [ISO 4217](https://en.wikipedia.org/wiki/ISO_4217) — Currency Codes
- [E.164](https://www.itu.int/rec/T-REC-E.164/) — Phone Number Format
- [Semantic Versioning](https://semver.org/) — Version numbering
- [OpenAPI 3.1](https://github.com/OAI/OpenAPI-Specification/blob/main/versions/3.1.0.md) — API specification format
- [OAuth 2.0 — RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749) — OAuth 2.0 Authorization Framework
- [PKCE — RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636) — Proof Key for Code Exchange
- [OWASP API Security Top 10](https://owasp.org/API-Security/) — API Security Risks
