# RelogioSoft API Governance Kit

> Canonical API governance standards for designing, building, and validating consistent, secure, and evolvable REST APIs.

[![Docs](https://img.shields.io/badge/docs-GitHub%20Pages-blue)](https://relogiosoft.github.io/relogiosoft-api-governance-kit)
[![Spectral](https://img.shields.io/badge/linting-Spectral-brightgreen)](https://stoplight.io/open-source/spectral)

---

## What is this?

The API Governance Kit is a structured corpus of **18 authoritative documents** covering every dimension of REST API design. All rules use [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) language — MUST, SHOULD, MAY — so teams know exactly what is mandatory versus advisory.

It is designed to serve as:
- **Human reference** — browsable docs portal for engineers and reviewers
- **RAG corpus** — knowledge base source for the [API Governance Assistant](https://github.com/RelogioSoft/relogiosoft-api-governance-assistant)
- **CI linting source** — Spectral ruleset for automated OpenAPI validation

---

## Standards overview

| # | Document | Key topics |
|---|----------|-----------|
| [01](docs/standards/01-api-first-principles.md) | API First Principles | Contract-first, API taxonomy, anti-corruption layers |
| [02](docs/standards/02-api-lifecycle.md) | API Lifecycle | draft → stable → deprecated → obsolete, `x-status` |
| [03](docs/standards/03-api-versioning.md) | API Versioning | Breaking vs non-breaking, content-type negotiation |
| [04](docs/standards/04-uri-design.md) | URI Design | `kebab-case`, plural nouns, no verbs, tenant scoping |
| [05](docs/standards/05-http-verbs.md) | HTTP Verbs | GET/POST/PUT/PATCH/DELETE semantics, JSON Patch (RFC 6902) |
| [06](docs/standards/06-http-status-codes.md) | HTTP Status Codes | 2xx/4xx/5xx decision flowcharts, 403 vs 404, 207 |
| [07](docs/standards/07-error-handling.md) | Error Handling | Fault object schema, `faultId`, `traceId`, security |
| [08](docs/standards/08-querying-and-filtering.md) | Querying & Filtering | Cursor pagination, sorting, field selection, `count` |
| [09](docs/standards/09-data-representations.md) | Data Representations | ISO 8601 dates, integer money, string IDs, geospatial |
| [10](docs/standards/10-caching.md) | Caching | `Cache-Control`, ETags, `If-None-Match`, `If-Match` |
| [11](docs/standards/11-observability-and-logging.md) | Observability & Logging | Structured logs, correlation IDs, GDPR, no-PII logging |
| [12](docs/standards/12-documentation-standards.md) | Documentation Standards | `operationId`, descriptions, grammar, casing |
| [13](docs/standards/13-security.md) | Security | OAuth 2.0, CORS, TLS, OWASP Top 10, rate limiting |
| [14](docs/standards/14-request-response-headers.md) | Request & Response Headers | Train-Case headers, `Location`, `x-conversation` |
| [15](docs/standards/15-idempotency.md) | Idempotency | `Idempotency-Key`, deduplication, 24-hour window |
| [16](docs/standards/16-webhooks-and-async-patterns.md) | Webhooks & Async Patterns | Delivery guarantees, HMAC signing, retries, async 202 |
| [17](docs/standards/17-health-endpoints.md) | Health Endpoints | `/health`, `/health/live`, `/health/ready`, no-auth |
| [18](docs/standards/18-batch-operations.md) | Batch Operations | `207 Multi-Status`, atomicity, max size, per-item errors |

Rule severity: **MUST/MUST NOT** = 🔴 Blocking · **SHOULD/SHOULD NOT** = 🟡 Recommendation · **MAY** = 🟢 Optional

---

## Browse the docs

```bash
pip install -r requirements-docs.txt
mkdocs serve
# Open http://127.0.0.1:8000
```

Or browse the published portal → **https://relogiosoft.github.io/relogiosoft-api-governance-kit**

---

## Lint your OpenAPI spec

```bash
npm install -g @stoplight/spectral-cli

# Against the published ruleset (no local clone needed)
spectral lint your-api.yaml \
  --ruleset https://raw.githubusercontent.com/RelogioSoft/relogiosoft-api-governance-kit/main/.spectral.yml

# Against a local clone
spectral lint your-api.yaml --ruleset .spectral.yml
```

The Spectral ruleset covers **30+ rules** across URI design, HTTP semantics, status codes, security, documentation, and data types. See [`docs/guides/spectral.md`](docs/guides/spectral.md) for the full rule reference and CI integration example.

A fully compliant example spec is at [`examples/openapi-example.yaml`](examples/openapi-example.yaml).

---

## AI Governance Assistant

The [RelogioSoft API Governance Assistant](https://github.com/RelogioSoft/relogiosoft-api-governance-assistant) answers governance questions grounded exclusively in this corpus using **AWS Bedrock Knowledge Bases**. Answers include citations back to specific sections of these documents.

---

## Repository structure

```
relogiosoft-api-governance-kit/
├── docs/
│   ├── index.md                    # Docs portal homepage
│   ├── standards/                  # 18 governance standard documents
│   │   ├── 01-api-first-principles.md
│   │   └── ... (18 total)
│   └── guides/
│       ├── spectral.md             # Spectral ruleset guide
│       └── contributing.md         # Contribution guide
├── examples/
│   └── openapi-example.yaml        # Governance-compliant example spec
├── .spectral.yml                   # Spectral linting ruleset
├── mkdocs.yml                      # MkDocs portal configuration
├── requirements-docs.txt           # Python deps for local preview
└── .github/
    └── workflows/
        ├── deploy-docs.yml         # Publish to GitHub Pages on push to main
        └── lint-openapi.yml        # Spectral lint on every PR
```

---

## Contributing

See [docs/guides/contributing.md](docs/guides/contributing.md).
