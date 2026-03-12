# Documentation Standards

**Category:** Governance
**Tags:** openapi, documentation, naming, descriptions, examples, casing, style-guide

---

## Summary of Rules

- API documentation **MUST** be grammatically correct and pass spell check.
- API documentation **MUST** be written as if presented to external integrators — clear, professional, and unambiguous.
- API names and property names **MUST** use `camelCase`.
- API documentation **SHOULD** be consistent in style and structure across all specifications.
- All API calls **MUST** include fully-formed, realistic request and response examples.
- Operation summaries **SHOULD** be as few words as possible, following the prescribed naming conventions.
- All descriptions **MUST** be present for parameters, headers, request body fields, response body fields, and response types.

---

## Operation Summary Naming Conventions

Operation summaries must be concise, consistent, and follow these patterns:

| HTTP Method | Summary Pattern | Examples |
|-------------|----------------|---------|
| `GET` | `Get [something]` | `Get customer`, `Get order items` |
| `POST` (create) | `Create [something]` | `Create customer`, `Create order` |
| `POST` (update) | `Update [something]` | `Update customer preferences` |
| `POST` (create or upsert) | `Create or update [something]` | `Create or update product` |
| `PUT` | `Create [something]` or `Update [something]` | `Update product catalogue entry` |
| `DELETE` | `Delete [something]` | `Delete customer`, `Delete order item` |
| `POST` (query, returns `200`) | `Query [something]` | `Query orders`, `Query product catalogue` |
| `PATCH` | `Update [something]` | `Update customer address` |

**Rules:**
- Use the simplest possible description of what the operation does.
- Do not start with a verb other than those above (avoid "Retrieve", "Fetch", "Remove", "Search").
- Do not include the resource name redundantly if it is already clear from context.
- Use sentence case (only the first word capitalised), not Title Case.

---

## Casing Conventions

| Context | Convention | Example |
|---------|-----------|---------|
| API property names (request/response bodies) | `camelCase` | `customerId`, `emailAddress`, `createdAt` |
| URI path segments | `kebab-case` | `/customer-accounts`, `/order-line-items` |
| URI path parameters | `camelCase` | `{customerId}`, `{orderId}` |
| Query string parameter names | `camelCase` | `?sortby`, `?limit`, `?after` |
| HTTP header names | `Train-Case` | `Content-Type`, `Cache-Control`, `x-conversation` |
| OpenAPI component names | `PascalCase` | `CustomerResponse`, `OrderLineItem` |

---

## Required Descriptions

Linting rules enforce that **all** of the following have descriptions in the OpenAPI spec:

| Element | Where to add description |
|---------|------------------------|
| Parameters (path, query, header) | `description` field on the parameter object |
| Request body fields | `description` field on each schema property |
| Response body fields | `description` field on each schema property |
| Response types (status codes) | `description` field on the response object |
| Path-level summary | `summary` field on the operation object |
| Component schemas | `description` field on the schema object |
| Enum values | `description` field where supported |

**A description MUST NOT be empty or placeholder text.** Examples of invalid descriptions:
- `"TODO"`
- `"string"`
- `"The id"`
- `"value"`

---

## Request and Response Examples

Every operation **MUST** include:

- A fully-formed **request example** (where a request body exists).
- A fully-formed **response example** for each documented status code.

Examples **MUST**:
- Use realistic, representative data (not `"string"`, `"example"`, `1` for every field).
- Reflect the actual data structure that will be returned.
- Include all required fields.
- Demonstrate the expected format for special types (dates, phone numbers, etc.).

### Good Example

```yaml
requestBody:
  content:
    application/json:
      schema:
        $ref: '#/components/schemas/CreateCustomerRequest'
      example:
        name: "Acme Corporation"
        emailAddress: "accounts@acme.com"
        phoneNumber: "+442071234567"
        address:
          line1: "10 Tech Street"
          city: "London"
          postCode: "EC1A 1BB"
          country: "GB"
```

### Poor Example (avoid)

```yaml
requestBody:
  content:
    application/json:
      schema:
        $ref: '#/components/schemas/CreateCustomerRequest'
      example:
        name: "string"
        emailAddress: "string"
        phoneNumber: "string"
```

---

## OpenAPI Structure Best Practices

### Info Object

```yaml
info:
  title: Customer API
  description: |
    Manages customer accounts including creation, retrieval, and update operations.

    ## Authentication
    All endpoints require a Bearer token in the Authorization header.

    ## Rate Limiting
    Requests are limited to 100/minute per client.
  version: "1.0"
  x-status: stable
  contact:
    name: Platform Team
    email: platform@example.com
```

### Parameter Descriptions

```yaml
parameters:
  - name: customerId
    in: path
    required: true
    description: "The unique identifier of the customer."
    schema:
      type: string
    example: "cust_abc123"

  - name: limit
    in: query
    required: false
    description: "Maximum number of records to return per page. Defaults to 20."
    schema:
      type: integer
      minimum: 1
      maximum: 100
      default: 20
```

### Schema Property Descriptions

```yaml
components:
  schemas:
    Customer:
      type: object
      description: "A customer account."
      required:
        - id
        - name
        - emailAddress
      properties:
        id:
          type: string
          description: "The unique identifier of the customer."
          example: "cust_abc123"
        name:
          type: string
          description: "The full legal name of the customer or organisation."
          example: "Acme Corporation"
        emailAddress:
          type: string
          description: "The primary email address for the customer account."
          example: "accounts@acme.com"
        createdAt:
          type: string
          format: date-time
          description: "The date and time the customer account was created, in UTC."
          example: "2024-07-23T10:30:00Z"
```

---

## Error Response Documentation

Every error status code **MUST** be documented with:

1. A description of when the error occurs.
2. The error codes that may be returned.
3. Guidance on how to resolve the error.

```yaml
responses:
  '400':
    description: |
      The request body is malformed or contains invalid values.

      **Error codes:**
      - `1001`: Required field missing
      - `1002`: Field value exceeds maximum length
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/FaultResponse'
        example:
          fault:
            faultId: "72d7036d-990a-4f84-9efa-ef5f40f6044b"
            traceId: "0HLOCKDKQPKIU"
            errors:
              - errorCode: "1001"
                description: "The 'name' field is required."
```

---

## Deprecation Documentation

When deprecating an operation or field, the description **MUST** include:

- A deprecation notice.
- The date of planned removal (sunset date).
- The replacement (if one exists) with a link or reference.

```yaml
/customers/{customerId}/legacy-address:
  get:
    summary: Get customer address (deprecated)
    deprecated: true
    description: |
      **⚠️ Deprecated:** This endpoint is deprecated and will be removed on 2025-12-31.

      Use [Get customer addresses](#/paths/customers/{customerId}/addresses/get) instead.
```

---

## Writing Quality Guidelines

### Grammar and Spelling

- Use standard English (British or American — choose one and be consistent within a spec).
- Run spell-check before publishing.
- Use active voice: "Returns the customer" not "The customer is returned".
- Be precise: "The date the record was created (UTC)" not "Created date".

### Tone

- Write for an external developer who has no knowledge of your internal systems.
- Avoid internal jargon, team names, or project code names in descriptions.
- Do not assume knowledge of your data model or domain.

### Completeness

- Every endpoint must be understandable from its documentation alone.
- Do not rely on out-of-band documentation (Confluence pages, README files) for critical API details.
- If behaviour differs between environments (dev/staging/production), document it.

---

## Linting and Validation

API specifications **SHOULD** be validated with a linting tool before merging. Recommended rules to enforce:

| Rule | Severity |
|------|----------|
| All operations have a `summary` | Error |
| All operations have a `description` | Warning |
| All path parameters have descriptions | Error |
| All query parameters have descriptions | Error |
| All request body properties have descriptions | Error |
| All response body properties have descriptions | Error |
| All 4xx/5xx responses have examples | Warning |
| All 2xx responses have examples | Error |
| `x-status` field present on Info Object | Warning |
| No empty or placeholder descriptions | Error |
| Property names are camelCase | Error |
| Operation IDs are present and unique | Error |
