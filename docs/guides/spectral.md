# Using the Spectral Ruleset

The governance kit ships a [Spectral](https://stoplight.io/open-source/spectral) ruleset that validates OpenAPI 3.x specs against the RelogioSoft governance standards.

## Quick start

```bash
# Install Spectral CLI
npm install -g @stoplight/spectral-cli

# Lint your spec against the published ruleset
spectral lint your-api.yaml \
  --ruleset https://raw.githubusercontent.com/RelogioSoft/relogiosoft-api-governance-kit/main/.spectral.yml
```

## Local usage

Clone this repository and reference `.spectral.yml` directly:

```bash
spectral lint your-api.yaml --ruleset /path/to/relogiosoft-api-governance-kit/.spectral.yml
```

## Severity levels

| Spectral level | Governance meaning |
|---|---|
| `error` | MUST / MUST NOT — blocking violation, fails CI |
| `warn` | SHOULD / SHOULD NOT — strong recommendation, advisory |
| `info` | MAY — optional improvement |

## Rules covered

| Rule ID | Standard | Checks |
|---------|----------|--------|
| `rls-info-x-status` | doc 02 | `x-status` field declared on `info` |
| `rls-info-x-status-enum` | doc 02 | `x-status` is a valid lifecycle value |
| `rls-paths-kebab-case` | doc 04 | All path segments use `kebab-case` |
| `rls-paths-no-trailing-slash` | doc 04 | No trailing slashes in paths |
| `rls-paths-no-file-extension` | doc 04 | No `.json`/`.xml` extensions in paths |
| `rls-get-no-request-body` | doc 05 | GET has no request body |
| `rls-delete-no-request-body` | doc 05 | DELETE has no request body |
| `rls-patch-json-patch-content-type` | doc 05 | PATCH declares `application/json-patch+json` |
| `rls-post-201-location-header` | doc 06 | 201 responses include `Location` header |
| `rls-delete-204` | doc 06 | DELETE declares 204 |
| `rls-operations-must-have-4xx` | doc 06 | Operations declare at least one 4xx |
| `rls-operations-must-have-500` | doc 06 | Operations declare 500/5xx |
| `rls-error-response-application-json` | doc 07 | Error responses use `application/json` |
| `rls-no-number-format-float-for-identifiers` | doc 09 | ID fields are `type: string` |
| `rls-no-float-for-monetary-amounts` | doc 09 | Monetary fields are not `float`/`double` |
| `rls-date-time-format` | doc 09 | Date/time fields declare `format: date-time` |
| `rls-get-cache-control-header` | doc 10 | GET 200s declare `Cache-Control` header |
| `rls-operation-id-required` | doc 12 | All operations have `operationId` |
| `rls-operation-id-camel-case` | doc 12 | `operationId` uses camelCase |
| `rls-operation-summary-required` | doc 12 | All operations have a `summary` |
| `rls-operation-description-required` | doc 12 | All operations have a `description` |
| `rls-operation-tags-required` | doc 12 | All operations declare tags |
| `rls-schema-description-required` | doc 12 | Schemas have descriptions |
| `rls-schema-property-description-required` | doc 12 | Schema properties have descriptions |
| `rls-no-api-key-in-query` | doc 13 | API keys not passed in query string |
| `rls-no-http-basic-auth` | doc 13 | HTTP Basic auth not used |
| `rls-no-implicit-oauth-flow` | doc 13 | OAuth 2.0 implicit flow not declared |
| `rls-no-password-oauth-flow` | doc 13 | OAuth 2.0 password grant not declared |
| `rls-operations-must-have-security` | doc 13 | Operations declare security |
| `rls-servers-https-only` | doc 13/14 | Non-localhost server URLs use HTTPS |
| `rls-health-endpoint-exists` | doc 17 | `/health` or `/health/live` path declared |

## Example: adding to your own CI

```yaml
# .github/workflows/lint-api.yml
name: Lint API spec

on: [pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm install -g @stoplight/spectral-cli
      - run: |
          spectral lint openapi.yaml \
            --ruleset https://raw.githubusercontent.com/RelogioSoft/relogiosoft-api-governance-kit/main/.spectral.yml \
            --fail-severity error
```

## See also

- [Example compliant OpenAPI spec](../../examples/openapi-example.yaml) — a fully governance-compliant Orders API
- [Standard 04 · URI Design](../standards/04-uri-design.md)
- [Standard 12 · Documentation Standards](../standards/12-documentation-standards.md)
- [Standard 13 · Security](../standards/13-security.md)
