# Batch Operations

**Category:** Design
**Tags:** batch, bulk, bulk-create, bulk-update, bulk-delete, partial-success, mixed-results, performance

---

## Summary of Rules

- Batch endpoints **MUST** use a dedicated collection-level URI with a meaningful sub-resource name (e.g. `/products/batch`).
- Batch requests **MUST** validate all items before processing any, or process each item independently and report failures per-item.
- Batch endpoints **MUST** return a structured response that identifies which items succeeded and which failed.
- Batch endpoints **SHOULD NOT** return `200 OK` if any items failed; they **SHOULD** use `207 Multi-Status` for mixed results.
- Batch endpoints **MUST** return `400 Bad Request` if the batch request itself is malformed (e.g. the payload is not valid JSON, or items are missing required fields at the structural level).
- Batch operations **SHOULD** be size-limited; the maximum batch size **MUST** be documented.
- Batch operations **SHOULD** be idempotent where possible. When idempotency is required, clients **SHOULD** use the `Idempotency-Key` header (see [Idempotency](./15-idempotency.md)).
- Batch operations **MUST** be atomic OR per-item independent — partial atomic commits (some items committed, some not) **MUST NOT** occur silently.

---

## When to Use Batch Operations

Batch operations reduce network round trips when a client needs to create, update, or delete multiple resources of the same type.

| Scenario | Recommendation |
|----------|---------------|
| Create 5–500 resources in one call | Batch create |
| Update multiple resources with different values | Batch update (PATCH) |
| Delete a set of resources by ID | Batch delete |
| Apply the same update to many resources | Batch update (more efficient than individual PATCHes) |
| Import data from an external system | Batch create, potentially async |

**Avoid batch for:**
- Heterogeneous operations (creating an order AND updating a customer). Use separate requests.
- Operations that require immediate per-item feedback (use individual requests for interactive UIs).
- Very large datasets (> 1000 items). Consider the async job pattern instead (see [Webhooks & Async Patterns](./16-webhooks-and-async-patterns.md)).

---

## URI Design for Batch Endpoints

Use a dedicated batch sub-resource under the collection:

```
POST /products/batch           → Batch create
PATCH /products/batch          → Batch update
DELETE /products/batch         → Batch delete
```

Do **not** overload the base collection endpoint with a special query parameter or header to trigger batch mode.

```
❌  POST /products?mode=batch
❌  POST /products  (with a magic header)
✅  POST /products/batch
```

---

## Batch Create

### Request

```http
POST /products/batch HTTP/1.1
Authorization: Bearer eyJ...
Content-Type: application/json
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

{
  "items": [
    {
      "sku": "WIDGET-RED-S",
      "name": "Red Widget Small",
      "priceInCents": 1299,
      "currency": "GBP"
    },
    {
      "sku": "WIDGET-RED-M",
      "name": "Red Widget Medium",
      "priceInCents": 1499,
      "currency": "GBP"
    },
    {
      "name": "Missing SKU — invalid item"
    }
  ]
}
```

### Response — All Succeeded

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "summary": {
    "total": 2,
    "succeeded": 2,
    "failed": 0
  },
  "results": [
    {
      "index": 0,
      "status": 201,
      "id": "prod_abc",
      "location": "/products/prod_abc"
    },
    {
      "index": 1,
      "status": 201,
      "id": "prod_def",
      "location": "/products/prod_def"
    }
  ]
}
```

### Response — Mixed Results (207 Multi-Status)

When some items succeed and others fail, return `207 Multi-Status`:

```http
HTTP/1.1 207 Multi-Status
Content-Type: application/json

{
  "summary": {
    "total": 3,
    "succeeded": 2,
    "failed": 1
  },
  "results": [
    {
      "index": 0,
      "status": 201,
      "id": "prod_abc",
      "location": "/products/prod_abc"
    },
    {
      "index": 1,
      "status": 201,
      "id": "prod_def",
      "location": "/products/prod_def"
    },
    {
      "index": 2,
      "status": 400,
      "errors": [
        {
          "errorCode": "REQUIRED_FIELD_MISSING",
          "description": "The 'sku' field is required.",
          "field": "sku"
        }
      ]
    }
  ]
}
```

---

## Batch Update (PATCH)

Use JSON Patch format per item, consistent with the single-resource PATCH standard (see [HTTP Verbs](./05-http-verbs.md)).

### Request

```http
PATCH /products/batch HTTP/1.1
Authorization: Bearer eyJ...
Content-Type: application/json

{
  "items": [
    {
      "id": "prod_abc",
      "patch": [
        { "op": "replace", "path": "/priceInCents", "value": 1099 }
      ]
    },
    {
      "id": "prod_def",
      "patch": [
        { "op": "replace", "path": "/priceInCents", "value": 1299 },
        { "op": "replace", "path": "/name", "value": "Red Widget M (New)" }
      ]
    }
  ]
}
```

### Response

```http
HTTP/1.1 207 Multi-Status
Content-Type: application/json

{
  "summary": { "total": 2, "succeeded": 2, "failed": 0 },
  "results": [
    { "index": 0, "id": "prod_abc", "status": 200 },
    { "index": 1, "id": "prod_def", "status": 200 }
  ]
}
```

---

## Batch Delete

### Request

```http
DELETE /products/batch HTTP/1.1
Authorization: Bearer eyJ...
Content-Type: application/json

{
  "ids": ["prod_abc", "prod_def", "prod_xyz_nonexistent"]
}
```

### Response

```http
HTTP/1.1 207 Multi-Status
Content-Type: application/json

{
  "summary": { "total": 3, "succeeded": 2, "failed": 1 },
  "results": [
    { "index": 0, "id": "prod_abc", "status": 204 },
    { "index": 1, "id": "prod_def", "status": 204 },
    { "index": 2, "id": "prod_xyz_nonexistent", "status": 404,
      "errors": [{ "description": "Resource not found." }] }
  ]
}
```

---

## HTTP Status Codes for Batch

| Situation | Status Code |
|-----------|------------|
| All items succeeded (creates) | `201 Created` |
| All items succeeded (updates/deletes) | `200 OK` |
| Some items succeeded, some failed | `207 Multi-Status` |
| All items failed | `207 Multi-Status` (with all failed results) |
| The batch request itself is malformed | `400 Bad Request` |
| Authentication failure | `401 Unauthorized` |
| Authorisation failure | `403 Forbidden` |
| Batch size exceeds limit | `400 Bad Request` |

**Note:** `207 Multi-Status` is an existing HTTP status code defined in WebDAV (RFC 4918). It is broadly supported and appropriate for mixed-result batch responses.

---

## Atomicity

Batch operations **MUST** declare their atomicity model and document it explicitly.

| Model | Description | When to Use |
|-------|-------------|-------------|
| **All-or-nothing (atomic)** | All items are processed in a transaction. If any item fails, all are rolled back. | Financial operations, inventory adjustments, any domain where partial state is invalid. |
| **Best-effort (per-item independent)** | Each item is processed independently. Some may succeed while others fail. | Import operations, catalogue updates, notifications. |

```yaml
# OpenAPI documentation should state the atomicity model
description: |
  Creates multiple products in a single request.

  **Atomicity:** This operation is best-effort. Each item is processed independently.
  If some items fail validation, the remaining items are still created.
  Check the `results` array for per-item outcomes.
```

---

## Size Limits

Every batch endpoint **MUST** document its maximum batch size and enforce it:

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "fault": {
    "faultId": "abc-123",
    "traceId": "xyz-456",
    "errors": [
      {
        "errorCode": "BATCH_SIZE_EXCEEDED",
        "description": "A maximum of 100 items may be submitted per batch request.",
        "itemCount": 250,
        "maxAllowed": 100
      }
    ]
  }
}
```

**Recommended default limits:**

| Operation | Suggested Limit |
|-----------|----------------|
| Batch create | 100 items |
| Batch update | 100 items |
| Batch delete | 500 items (IDs are cheaper than full resources) |

For larger datasets, use the async job pattern (see [Webhooks & Async Patterns](./16-webhooks-and-async-patterns.md)).

---

## Response Schema Reference

### Batch Response Envelope

```yaml
components:
  schemas:
    BatchResponse:
      type: object
      required: [summary, results]
      properties:
        summary:
          $ref: '#/components/schemas/BatchSummary'
        results:
          type: array
          items:
            $ref: '#/components/schemas/BatchItemResult'

    BatchSummary:
      type: object
      required: [total, succeeded, failed]
      properties:
        total:
          type: integer
          description: "Total number of items submitted in the batch."
        succeeded:
          type: integer
          description: "Number of items that were processed successfully."
        failed:
          type: integer
          description: "Number of items that failed to process."

    BatchItemResult:
      type: object
      required: [index, status]
      properties:
        index:
          type: integer
          description: "Zero-based index of this item in the submitted batch."
        status:
          type: integer
          description: "HTTP status code representing the outcome for this item."
        id:
          type: string
          description: "The ID of the created or affected resource (when successful)."
        location:
          type: string
          format: uri
          description: "URI of the created resource (for 201 results)."
        errors:
          type: array
          description: "Error details for this item (when status is 4xx)."
          items:
            $ref: '#/components/schemas/BatchItemError'

    BatchItemError:
      type: object
      properties:
        errorCode:
          type: string
          description: "Application error code."
        description:
          type: string
          description: "Human-readable description of the error."
        field:
          type: string
          description: "The field that caused the error, if applicable."
```
