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
- [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) — Date and Time
- [ISO 4217](https://en.wikipedia.org/wiki/ISO_4217) — Currency Codes
- [E.164](https://www.itu.int/rec/T-REC-E.164/) — Phone Number Format
- [Semantic Versioning](https://semver.org/) — Version numbering
- [OpenAPI 3.1](https://github.com/OAI/OpenAPI-Specification/blob/main/versions/3.1.0.md) — API specification format
