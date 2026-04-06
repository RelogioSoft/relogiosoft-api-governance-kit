# RelogioSoft API Governance Kit

**Canonical governance standards for building consistent, secure, and evolvable REST APIs.**

The API Governance Kit is a structured corpus of 18 authoritative documents covering every dimension of API design — from URI naming and HTTP semantics to security, observability, and lifecycle management. All rules are graded using [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) language so teams know exactly what is mandatory versus advisory.

---

## Why governance matters

Ungoverned APIs accumulate inconsistencies, security gaps, and breaking changes that compound over time. This kit provides a shared language for API authors, reviewers, and governance tooling — so teams can move fast without diverging.

---

## Standards coverage

| # | Document | Key topics |
|---|----------|-----------|
| 01 | [API First Principles](standards/01-api-first-principles.md) | Contract-first, API taxonomy, anti-corruption layers |
| 02 | [API Lifecycle](standards/02-api-lifecycle.md) | draft → stable → deprecated → obsolete, `x-status` |
| 03 | [API Versioning](standards/03-api-versioning.md) | Breaking vs non-breaking, content-type negotiation, deprecation headers |
| 04 | [URI Design](standards/04-uri-design.md) | `kebab-case`, plural nouns, no verbs, tenant scoping |
| 05 | [HTTP Verbs](standards/05-http-verbs.md) | GET/POST/PUT/PATCH/DELETE semantics, JSON Patch (RFC 6902) |
| 06 | [HTTP Status Codes](standards/06-http-status-codes.md) | 2xx/4xx/5xx decision flowcharts, 403 vs 404, 207 |
| 07 | [Error Handling](standards/07-error-handling.md) | Fault object schema, `faultId`, `traceId`, security |
| 08 | [Querying & Filtering](standards/08-querying-and-filtering.md) | Cursor pagination, sorting, field selection, `count` |
| 09 | [Data Representations](standards/09-data-representations.md) | ISO 8601 dates, integer money, string IDs, geospatial |
| 10 | [Caching](standards/10-caching.md) | `Cache-Control`, ETags, `If-None-Match`, `If-Match` |
| 11 | [Observability & Logging](standards/11-observability-and-logging.md) | Structured logs, correlation IDs, GDPR, no-PII logging |
| 12 | [Documentation Standards](standards/12-documentation-standards.md) | `operationId`, descriptions, grammar, casing |
| 13 | [Security](standards/13-security.md) | OAuth 2.0, CORS, TLS, OWASP Top 10, rate limiting |
| 14 | [Request & Response Headers](standards/14-request-response-headers.md) | Train-Case headers, `Location`, `x-conversation` |
| 15 | [Idempotency](standards/15-idempotency.md) | `Idempotency-Key`, deduplication, 24-hour window |
| 16 | [Webhooks & Async Patterns](standards/16-webhooks-and-async-patterns.md) | Delivery guarantees, HMAC signing, retries, async 202 |
| 17 | [Health Endpoints](standards/17-health-endpoints.md) | `/health`, `/health/live`, `/health/ready`, no-auth |
| 18 | [Batch Operations](standards/18-batch-operations.md) | `207 Multi-Status`, atomicity, max size, per-item errors |

---

## Rule severity

All rules are graded with RFC 2119 language:

| Marker | Meaning | Governance level |
|--------|---------|-----------------|
| **MUST** / **MUST NOT** | Absolute requirement / prohibition | 🔴 Blocking |
| **SHOULD** / **SHOULD NOT** | Strong recommendation | 🟡 Recommendation |
| **MAY** | Optional / advisory | 🟢 Optional |

---

## Using this kit

### Browse the standards
Use the navigation above to explore any standard. Each document includes rationale, examples, and decision flowcharts where applicable.

### Lint your OpenAPI specs
Use the [Spectral ruleset](guides/spectral.md) to validate OpenAPI specs against these standards in CI:

```bash
npm install -g @stoplight/spectral-cli
spectral lint your-api.yaml --ruleset https://raw.githubusercontent.com/RelogioSoft/relogiosoft-api-governance-kit/main/.spectral.yml
```

### AI-powered governance assistant
The [RelogioSoft API Governance Assistant](https://github.com/RelogioSoft/relogiosoft-api-governance-assistant) answers governance questions grounded exclusively in this corpus, with citations, using AWS Bedrock Knowledge Bases.

---

## Local preview

```bash
pip install -r requirements-docs.txt
mkdocs serve
# Open http://127.0.0.1:8000
```
