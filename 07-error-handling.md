# Error Handling

**Category:** Design
**Tags:** errors, fault, error-response, 4xx, 5xx, faultId, traceId, errorCode, security, pii

---

## Summary of Rules

- All responses **MUST** return an appropriate HTTP status code (see [HTTP Status Codes](./06-http-status-codes.md)).
- Application code **MUST** respond to unhandled server errors with `500 Internal Server Error`. All other 5xx codes are for web servers and gateways.
- A `500` error **MUST NOT** return additional error information beyond `faultId`, `traceId`, and the description `"Internal Server Error"`. Do not leak implementation details.
- All `4xx` error responses **MUST** use the standard fault object format.
- A `4xx` error **MUST NOT** leak implementation details such as stack traces or exception messages.
- All error codes **MUST** be documented on the developer portal with advice on resolution.
- A `4xx` error **SHOULD** return information that allows the sender to correct the request and resend, if possible.
- A `4xx` error **MAY** extend the fault object with additional fields that help diagnose the error, **provided** those fields do not facilitate enumeration attacks or leak PII.

---

## Error Response Format

### 4xx Error Response

All client error responses **MUST** follow this structure:

```json
{
  "fault": {
    "faultId": "72d7036d-990a-4f84-9efa-ef5f40f6044b",
    "traceId": "0HLOCKDKQPKIU",
    "errors": [
      {
        "errorCode": "2150",
        "description": "The end date may not be before the start date",
        "startDate": "2024-03-12",
        "endDate": "2024-02-09"
      }
    ]
  }
}
```

### 5xx Error Response

Server error responses **MUST** follow this restricted structure:

```json
{
  "fault": {
    "faultId": "72dwr76d-990a-4f84-9efa-ef5f40fe644b",
    "traceId": "0HLONJ7KQPKAU",
    "errors": [
      {
        "description": "Internal Server Error"
      }
    ]
  }
}
```

---

## Fault Object Fields

### Common Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `faultId` | **Yes** | string (UUID) | Unique identifier for this specific error occurrence. Use it to correlate the response with server-side logs. |
| `traceId` | **Yes** | string | Trace identifier for the request. Use it to find the full request trace in the observability platform. |
| `errors` | **Yes** | array | One or more error detail objects. |

### Error Detail Fields (within `errors` array)

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `errorCode` | No | string | A code identifying the problem type. Must be documented on the developer portal. |
| `description` | No | string | Human-readable summary of the error. On 500 responses, the only permitted value is `"Internal Server Error"`. |
| `[extended members]` | No | any | Additional fields specific to this error that help the client diagnose the problem (see Security Considerations). |

### 5xx Fields Only

| Field | Location | Description |
|-------|----------|-------------|
| `Retry-After` | HTTP response header | Optional. Indicates the client MAY retry after the specified delay. Use when the error is transient. |

---

## Extended Error Members

The error object **MAY** be extended with fields that help the client understand what values they sent that caused the error:

```json
{
  "fault": {
    "faultId": "abc-123",
    "traceId": "xyz-456",
    "errors": [
      {
        "errorCode": "2150",
        "description": "The end date may not be before the start date",
        "startDate": "2024-03-12",
        "endDate": "2024-02-09"
      }
    ]
  }
}
```

Here, `startDate` and `endDate` are extended members echoing back the client-supplied values that caused the error. This helps the client identify the problem without the server leaking internal state.

---

## Security Considerations

Error messages are a common attack vector. Follow these rules when composing error responses:

### Do Not Leak Implementation Details

- **No stack traces** in 4xx or 5xx responses.
- **No exception class names** or internal framework error messages.
- **No database error messages** (e.g., SQL syntax errors, constraint names).
- **No internal server addresses, service names, or infrastructure details**.

### Do Not Leak PII

- **No names or contact details** in error messages. These can be exploited in social engineering attacks.
- **No email addresses, phone numbers, or national identifiers** in error messages.

### Avoid Facilitating Enumeration Attacks

Extended error members **MUST** be reviewed to determine if they could allow an attacker to probe the API:

- Reflecting back which specific resource ID was not found can confirm that other IDs exist.
- Do not distinguish between "resource does not exist" and "you do not have permission" in error messages — use `403` uniformly for permission denial (see [HTTP Status Codes](./06-http-status-codes.md)).
- Avoid "did you mean...?" style suggestions that reveal valid values.

### Provide Actionable Context

Outside of the security constraints above, error messages **SHOULD**:

- **Provide context**: What was the client attempting?
- **Provide actionable advice**: What should the client do to fix the problem?
- **Reference documentation**: Point to the error code's developer portal page for detailed guidance.

---

## Error Codes

Every `errorCode` value **MUST** be:

1. Documented on the developer portal.
2. Accompanied by human-readable documentation explaining the error and how to resolve it.

Error codes enable clients to programmatically handle specific error conditions and direct developers to relevant documentation.

---

## Why Not RFC 9457 (Problem Details)?

[RFC 9457](https://datatracker.ietf.org/doc/html/rfc9457) is a proposed standard for error responses. The format above differs from it for these reasons:

- RFC 9457 requires a URI for the error type, which mandates an HTML page to resolve to — adding unnecessary infrastructure.
- RFC 9457 requires a URI as a trace identifier.
- RFC 9457 duplicates the HTTP status code in the body.
- RFC 9457 examples explicitly encourage leaking sensitive details (e.g. account balance in an insufficient funds response).

The format above is inspired by RFC 9457's extensibility concept but avoids these drawbacks.

---

## Business Logic Errors and HTTP Status Codes

A recurring debate: should business logic errors use HTTP status codes or always return 2xx?

**Our position:** Use `4xx` for errors where client input is invalid for the current resource state. Use `5xx` for server-side faults where the input was well-formed.

**Rationale:** REST treats HTTP verbs as operations on resources. A `4xx` correctly signals that applying the requested verb to the resource failed because of client input. Wrapping business errors in `200 OK` hides the failure and complicates client error handling.

```
POST /orders          → 422 (order date in the past)          ← client error
POST /orders          → 409 (duplicate order ID)              ← client conflict
POST /orders          → 500 (database connection failed)      ← server fault
```

---

## Response Header for Transient 5xx Errors

When a 5xx error is transient (e.g. temporary overload), include:

```http
HTTP/1.1 503 Service Unavailable
Retry-After: 30

{
  "fault": {
    "faultId": "...",
    "traceId": "...",
    "errors": [{ "description": "Internal Server Error" }]
  }
}
```

The `Retry-After` value is in seconds and signals that the client **SHOULD** back off before retrying.
