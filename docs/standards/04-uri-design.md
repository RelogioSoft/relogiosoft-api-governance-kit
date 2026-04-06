# URI Design

**Category:** Design
**Tags:** uri, resources, collections, singletons, kebab-case, tenancy, application-specific, naming

---

## Summary of Rules

- A resource **MUST** be accessed from a collection.
- A resource that is specific to a tenant **MUST** be identified with the URI format `{tenant}/{id}`.
- A resource designed for a specific application experience (Gateway, BFF, BFI, or vendor integration) **MUST** have its URI prefixed with `/applications/{application}`.
- A resource name **MUST** be in `kebab-case` (see [RFC 3986](https://datatracker.ietf.org/doc/html/rfc3986)).
- A singleton resource **SHOULD** be accessed independently of other resources where possible.
- A singleton resource **MAY** contain other singleton resources.
- A singleton resource **MAY** contain sub-collection resources.

---

## Resource and Collection Structure

### Collections and Singletons

Resources follow a collection → singleton pattern:

| Pattern | Description | Example URI |
|---------|-------------|-------------|
| Collection | A group of resources of the same type | `/customers` |
| Singleton | A single resource within a collection | `/customers/{customerId}` |
| Sub-collection | A collection nested within a singleton | `/customers/{customerId}/addresses` |
| Singleton within singleton | A resource that only makes sense in context of its parent | `/customers/{customerId}/name` |

### Rules

**Collections use plural nouns. Singletons use the collection path + identifier.**

```
GET /customers                    → List all customers
GET /customers/{customerId}       → Get a specific customer
POST /customers                   → Create a new customer
PUT /customers/{customerId}       → Replace a customer
PATCH /customers/{customerId}     → Partially update a customer
DELETE /customers/{customerId}    → Delete a customer
```

**Sub-collections** represent resources that belong to a specific parent:

```
GET /customers/{customerId}/addresses             → List customer addresses
GET /customers/{customerId}/addresses/{addressId} → Get a specific address
```

**Prefer independent access** when a resource can be referenced from multiple contexts:

```
# Preferred: order accessible independently
GET /orders/{orderId}

# Avoid: order locked under customer
GET /customers/{customerId}/orders/{orderId}
```

This allows the resource to be referenced by different parent resources without ambiguity.

---

## URI Naming Conventions

- Use `kebab-case` for all resource path segments (lowercase, words separated by hyphens).
- Do **not** use snake_case, camelCase, or PascalCase in URI path segments.
- Use plural nouns for collections.
- Avoid verbs in URIs — operations are expressed by HTTP methods, not URI names.

```
✅  /product-types
✅  /order-line-items
✅  /customer-accounts/{accountId}

❌  /productTypes
❌  /ProductTypes
❌  /get-orders
❌  /createCustomer
```

---

## Tenant-Specific Resources

When the same logical resource can exist for multiple tenants and identifiers are only unique within a tenant, the URI **MUST** include the tenant:

```
GET /orders/{tenant}/{orderId}
```

Example:
```
GET /orders/uk/12345
GET /orders/de/12345    ← different resource, same orderId but different tenant
```

---

## Application-Specific Resources

Resources designed for a specific application experience (Gateway, BFF/BFI, vendor integration) **MUST** be prefixed with `/applications/{application}`:

```
/applications/catalog-orchestrator/catalog
/applications/twilio-ivr/callback
/applications/partner-portal/shipments
```

This makes it immediately obvious that the resource is application-specific and NOT a stable platform resource. It signals to consumers that the resource is coupled to a specific client and **SHOULD NOT** be used as a general-purpose API.

### Characteristics of Application-Specific Resources

**Gateway resource:**
- Built for a specific client or group of clients.
- Reduces the number of TCP connections a client needs by combining multiple platform resources.

**BFF/BFI resource:**
- Same as Gateway resource.
- May contain orchestration of business logic specific to the client.
- May represent resources differently from how the underlying systems represent them.

**Vendor-specific resource:**
- Schema defined by the vendor.
- Built for a specific vendor or partner.

Application-specific resources are **not stable platforms** for other applications to build on — they change with the client's requirements.

---

## Referencing Resources Across Contexts

When a resource can logically belong to multiple parent resources, do **not** nest it under multiple parents. Instead, give it an independent top-level URI that all parents can reference.

**Problem (avoid):**

```
GET /customers/{customerId}/orders/{orderId}
GET /warehouses/{warehouseId}/orders/{orderId}
```

There is no single canonical URI for an order, making linking ambiguous.

**Solution (preferred):**

```
GET /orders/{orderId}         ← canonical URI
```

Both `/customers/{customerId}` and `/warehouses/{warehouseId}` can include a reference to the order's URI in their response bodies.

---

## URI Design Examples

```
# Platform API — independent resources
GET  /products                           → list products
GET  /products/{productId}               → get product
GET  /products/{productId}/images        → list product images
GET  /products/{productId}/images/{id}   → get specific image

# Tenant-scoped resource
GET  /entities/{tenant}/{entityId}       → get entity for tenant

# Application-specific resource (Gateway/BFF)
GET  /applications/mobile-app/dashboard  → mobile-specific dashboard composition
POST /applications/twilio-ivr/callback   → vendor callback endpoint

# Sub-resource that only makes sense in context
GET  /customers/{customerId}/preferences → preferences are never accessed independently
```

---

## URI Design Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
|--------------|---------|-----------------|
| `/getCustomer` | Verb in URI | `GET /customers/{id}` |
| `/customers/create` | Verb in URI | `POST /customers` |
| `/CustomerList` | PascalCase + non-collection name | `GET /customers` |
| `/customer_orders` | snake_case | `GET /customer-orders` |
| `/customers/{id}/orders/{orderId}` when orders exist independently | Unnecessary nesting | `GET /orders/{orderId}` |
| Omitting tenant from tenant-scoped resource | Ambiguous identity | `GET /orders/{tenant}/{orderId}` |
