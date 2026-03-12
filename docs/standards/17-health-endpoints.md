# Health Endpoints

**Category:** Operations
**Tags:** health, readiness, liveness, health-check, kubernetes, probes, operational, uptime

---

## Summary of Rules

- Every service **MUST** expose a `/health` endpoint.
- The `/health` endpoint **MUST NOT** require authentication.
- The `/health` endpoint **MUST** return `200 OK` when the service is operating normally.
- The `/health` endpoint **MUST** return `503 Service Unavailable` when the service cannot serve traffic.
- The `/health` endpoint **MUST NOT** include sensitive data (PII, credentials, internal addresses).
- Services deployed to Kubernetes **SHOULD** expose separate `/health/live` (liveness) and `/health/ready` (readiness) endpoints.
- Health endpoint response bodies **SHOULD** follow the standard schema defined in this document.
- Health endpoints **SHOULD NOT** be included in the public OpenAPI spec; they are operational infrastructure, not the API contract.
- The response time of `/health` **SHOULD** be under 500 milliseconds; health checks **MUST NOT** perform expensive queries.

---

## Why Health Endpoints?

Health endpoints allow infrastructure components (load balancers, container orchestrators, monitoring systems) to determine whether a service instance is safe to receive traffic.

| Component | Uses Health Check For |
|-----------|----------------------|
| Load balancer | Route traffic only to healthy instances |
| Kubernetes | Restart unhealthy pods (liveness); hold traffic until ready (readiness) |
| API gateway | Remove degraded upstreams from the rotation |
| Monitoring / alerting | Alert on-call when service health degrades |

---

## Endpoint Definitions

### `/health` — General Health

A combined health check suitable for load balancers and monitoring tools.

```http
GET /health HTTP/1.1
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "status": "Healthy",
  "timestamp": "2024-07-23T10:30:00Z",
  "version": "1.4.2",
  "checks": [
    { "name": "database", "status": "Healthy" },
    { "name": "cache", "status": "Healthy" }
  ]
}
```

### `/health/live` — Liveness (Kubernetes)

Answers: **"Is this process alive and not deadlocked?"**

A failing liveness probe causes Kubernetes to restart the pod. Keep this check lightweight — it should only verify that the process is running and responsive:

- Is the process responding?
- Is there sufficient memory / file handles?

It **MUST NOT** check external dependencies (database, downstream services).

```http
GET /health/live HTTP/1.1
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "status": "Healthy",
  "timestamp": "2024-07-23T10:30:00Z"
}
```

### `/health/ready` — Readiness (Kubernetes)

Answers: **"Is this instance ready to serve traffic?"**

A failing readiness probe causes Kubernetes to remove the pod from the service's load balancer without restarting it. Use this to signal temporary unavailability:

- Is the database connection pool available?
- Have startup migrations or cache warm-ups completed?
- Are required downstream dependencies reachable?

```http
GET /health/ready HTTP/1.1
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "status": "Healthy",
  "timestamp": "2024-07-23T10:30:00Z",
  "checks": [
    { "name": "database", "status": "Healthy", "responseTimeMs": 4 },
    { "name": "config-service", "status": "Healthy", "responseTimeMs": 12 }
  ]
}
```

---

## Response Schema

### Status Values

| Status | HTTP Code | Meaning |
|--------|-----------|---------|
| `Healthy` | `200 OK` | All checks passed; service is operating normally. |
| `Degraded` | `200 OK` | One or more non-critical checks failed; service is operating with reduced capability. Load balancers **SHOULD** continue routing traffic. |
| `Unhealthy` | `503 Service Unavailable` | One or more critical checks failed; service cannot serve requests. Load balancers **MUST** stop routing traffic. |

### Top-Level Fields

```json
{
  "status": "Healthy",
  "timestamp": "2024-07-23T10:30:00Z",
  "version": "1.4.2",
  "checks": [ ... ]
}
```

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `status` | **Yes** | string | Overall health: `Healthy`, `Degraded`, or `Unhealthy`. |
| `timestamp` | **Yes** | string (ISO 8601 UTC) | When the health check was performed. |
| `version` | No | string | Deployed version of the service. |
| `checks` | No | array | Individual dependency check results. |

### Check Object Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `name` | **Yes** | string | Identifier for this check (e.g. `"database"`, `"cache"`, `"payment-service"`). |
| `status` | **Yes** | string | `Healthy`, `Degraded`, or `Unhealthy`. |
| `responseTimeMs` | No | integer | Time taken for this check, in milliseconds. |
| `description` | No | string | Human-readable detail about the check result (never include PII or credentials). |

### Example: Degraded Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "status": "Degraded",
  "timestamp": "2024-07-23T10:30:00Z",
  "version": "1.4.2",
  "checks": [
    { "name": "database", "status": "Healthy", "responseTimeMs": 5 },
    { "name": "email-service", "status": "Unhealthy", "description": "Connection timeout after 3 retries" }
  ]
}
```

Email service is non-critical (emails are queued); the service continues operating but is degraded.

### Example: Unhealthy Response

```http
HTTP/1.1 503 Service Unavailable
Content-Type: application/json
Cache-Control: no-store
Retry-After: 30

{
  "status": "Unhealthy",
  "timestamp": "2024-07-23T10:30:00Z",
  "version": "1.4.2",
  "checks": [
    { "name": "database", "status": "Unhealthy", "description": "Cannot connect to primary database" },
    { "name": "cache", "status": "Healthy", "responseTimeMs": 2 }
  ]
}
```

---

## What to Check

### Liveness Checks (fast, internal only)

| Check | What to Verify |
|-------|---------------|
| Process health | Is the service responding to HTTP? |
| Memory | Is the process within its memory limits? |
| Thread pool | Are worker threads available? |

### Readiness Checks (may include external dependencies)

| Check | What to Verify |
|-------|---------------|
| Database | Can the service execute a simple query? |
| Cache | Is the cache connection available? |
| Required config | Has configuration loaded successfully? |
| Startup tasks | Have migrations / warm-ups completed? |

### What NOT to Check in Health Endpoints

- **Downstream non-critical services**: If the service can function without them (via degraded mode or fallback), do not fail the health check.
- **Expensive queries**: Health checks must be fast (<500 ms). Use a trivial probe query (e.g. `SELECT 1`), not a business query.
- **External third parties**: If the third party is outside your control (e.g. a payment gateway), report it as `Degraded` rather than `Unhealthy` unless it completely blocks operation.

---

## URI Conventions

```
/health           → General health (combined liveness + readiness)
/health/live      → Liveness only
/health/ready     → Readiness only
```

These endpoints **SHOULD** be at the root of the service path, not under a versioned prefix (e.g. not `/v1/health`). Health endpoints are operational infrastructure, not part of the API version contract.

---

## Kubernetes Probe Configuration

Example Kubernetes deployment configuration:

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
```

---

## Caching

Health endpoint responses **MUST** include `Cache-Control: no-store` to prevent intermediaries from caching health status. Stale health responses could cause traffic to be routed to an unhealthy instance.

```http
Cache-Control: no-store
```
