# API Design Standards

Consistent API design reduces integration friction, enables predictable client behavior, and simplifies security review.

---

## Core Principles

1. **Resource-oriented** — APIs are nouns (resources), not verbs (actions)
2. **Predictable** — consistent patterns across all endpoints; no surprises
3. **Versioned from day one** — breaking changes will happen; plan for them
4. **Secure by default** — authentication and authorization checked on every endpoint
5. **Documented** — every endpoint has an OpenAPI spec before implementation

---

## REST Resource Naming

### Rules

- Resources are **plural nouns**: `/users`, `/orders`, `/products`
- Use **kebab-case** for multi-word resources: `/payment-methods`, `/audit-logs`
- Nest to express ownership (max 2 levels deep): `/users/{userId}/orders`
- Never use verbs in paths — use the HTTP method to express the action

```
# ✅ Good
GET    /users
GET    /users/{userId}
POST   /users
PUT    /users/{userId}
PATCH  /users/{userId}
DELETE /users/{userId}
GET    /users/{userId}/orders
GET    /users/{userId}/orders/{orderId}

# ❌ Bad
GET  /getUser
POST /createNewUser
POST /users/delete      ← use DELETE method instead
POST /users/update      ← use PUT/PATCH instead
GET  /users/{userId}/orders/{orderId}/items/{itemId}/discount  ← too deep
```

### Actions That Don't Fit CRUD

For non-CRUD operations, use a resource-action pattern with POST:

```
POST /users/{userId}/actions/deactivate
POST /orders/{orderId}/actions/cancel
POST /payments/{paymentId}/actions/refund
```

---

## HTTP Methods

| Method | Semantics | Idempotent | Safe | Body? |
|--------|-----------|-----------|------|-------|
| GET | Read resource(s) | Yes | Yes | No |
| POST | Create resource, or trigger action | No | No | Yes |
| PUT | Full replacement of a resource | Yes | No | Yes |
| PATCH | Partial update of a resource | No | No | Yes |
| DELETE | Remove a resource | Yes | No | Optional |
| HEAD | Same as GET but response body omitted | Yes | Yes | No |
| OPTIONS | Describe available methods | Yes | Yes | No |

Use `PATCH` for partial updates (the common case). Use `PUT` only when the client sends a complete replacement and partial updates would be ambiguous.

---

## URL Versioning

All APIs are versioned from the first release:

```
/api/v1/users
/api/v1/users/{userId}
/api/v2/users          ← new major version when breaking changes needed
```

### Versioning Policy

- A **major version bump** is required for any breaking change (removed fields, changed types, deleted endpoints, changed auth)
- **Minor additions** to a response body (new optional fields) are non-breaking and do not require a version bump
- A deprecated version must remain available for **6 months minimum** before removal
- Deprecation must be communicated via `Deprecation` and `Sunset` response headers:

```
Deprecation: Sat, 31 Dec 2025 23:59:59 GMT
Sunset: Sat, 30 Jun 2026 23:59:59 GMT
Link: <https://api.example.com/api/v2/users>; rel="successor-version"
```

---

## HTTP Status Codes

Use precise status codes. Do not return 200 for errors or generic 500 for validation failures.

| Code | When to Use |
|------|-------------|
| 200 OK | Successful GET, PUT, PATCH, DELETE |
| 201 Created | Successful POST that created a resource |
| 202 Accepted | Async operation accepted; not yet complete |
| 204 No Content | Successful DELETE with no response body |
| 400 Bad Request | Invalid input, validation failure, malformed JSON |
| 401 Unauthorized | Missing or invalid authentication credentials |
| 403 Forbidden | Authenticated but not authorized to perform action |
| 404 Not Found | Resource does not exist (or hidden for security) |
| 409 Conflict | Resource already exists, or state conflict |
| 410 Gone | Resource permanently deleted |
| 422 Unprocessable Entity | Input is syntactically valid but semantically invalid |
| 429 Too Many Requests | Rate limit exceeded |
| 500 Internal Server Error | Unexpected server error |
| 502 Bad Gateway | Upstream service failure |
| 503 Service Unavailable | Temporary outage; include `Retry-After` header |

**Never** return:
- 200 with `{ "success": false }` — use the appropriate 4xx/5xx code
- 500 for client input errors — those are 400/422

---

## Error Response Schema

All error responses use the same envelope:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request input failed validation",
    "details": [
      {
        "field": "email",
        "code": "INVALID_FORMAT",
        "message": "Must be a valid email address"
      },
      {
        "field": "password",
        "code": "TOO_SHORT",
        "message": "Must be at least 12 characters"
      }
    ],
    "traceId": "4bf92f3577b34da6a3ce929d0e0e4736"
  }
}
```

### Rules

- `error.code` is a **machine-readable string constant** in `SCREAMING_SNAKE_CASE`
- `error.message` is a **human-readable description** appropriate for logging (not necessarily for end users)
- `error.details` is optional; include it for validation errors to identify each failing field
- `error.traceId` must always be present — allows correlating with server logs
- **Never include stack traces** in error responses to clients (log them server-side)
- **Never include internal system details** (database name, table name, framework version) in error responses

---

## Pagination

Use **cursor-based pagination** for large or unbounded collections. Use **offset pagination** only for user-facing list UIs where a user needs to jump to a specific page.

### Cursor-Based Pagination (Default)

```
GET /api/v1/users?limit=20&after=dXNlcl8xMjM=

Response:
{
  "data": [ ... ],
  "pagination": {
    "hasNextPage": true,
    "hasPreviousPage": false,
    "endCursor": "dXNlcl8xMjM=",
    "startCursor": "dXNlcl8xMDA="
  }
}
```

### Offset Pagination (UI Lists Only)

```
GET /api/v1/users?page=3&pageSize=20

Response:
{
  "data": [ ... ],
  "pagination": {
    "page": 3,
    "pageSize": 20,
    "totalPages": 12,
    "totalCount": 234
  }
}
```

### Pagination Rules

- Default page size: **20**
- Maximum page size: **100** (return 400 if client requests more)
- Always return pagination metadata even when the result fits in one page

---

## Filtering and Sorting

Use query parameters for filtering and sorting:

```
# Filtering
GET /api/v1/orders?status=PENDING&createdAfter=2024-01-01T00:00:00Z

# Sorting
GET /api/v1/users?sort=lastName&order=asc
GET /api/v1/users?sort=-createdAt             # prefix with "-" for descending

# Combined
GET /api/v1/products?category=electronics&sort=-price&limit=20&after=cursor123
```

### Rules

- Sort field names match the response body field names exactly
- Date filters use **ISO 8601** format (`YYYY-MM-DDTHH:MM:SSZ`)
- Validate all filter parameters and return 400 for unknown or invalid filter names
- Never allow filtering/sorting on non-indexed database columns without pagination to prevent unbounded slow queries

---

## Request and Response Body Standards

### Request Body Rules

- Use `Content-Type: application/json` for all JSON bodies
- Validate **every field** before processing — return 400 with field-level errors for validation failures
- Accept ISO 8601 for all date/time fields
- Use `camelCase` for JSON field names

### Response Body Rules

- Always return a **consistent envelope** (not bare arrays):

```json
// ✅ Consistent envelope
{
  "data": { "id": "user-123", "email": "user@example.com" }
}

// Collection
{
  "data": [ ... ],
  "pagination": { ... }
}

// ❌ Bare array (breaks envelope consistency)
[ { "id": "user-123" }, ... ]
```

- **Never return computed secrets, hashed passwords, or internal IDs** in response bodies
- Use `null` for missing optional fields (not undefined / field omission) for predictable client behavior
- Use ISO 8601 for all date/time values — always include timezone (`Z` for UTC)

---

## Rate Limiting

Every API must implement rate limiting. Return standard headers:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1704067200
Content-Type: application/json

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Retry after 60 seconds.",
    "traceId": "..."
  }
}
```

| Endpoint Type | Suggested Limit |
|--------------|----------------|
| Authentication endpoints | 5 requests/min per IP |
| Write operations | 60 requests/min per user |
| Read operations | 300 requests/min per user |
| Public/unauthenticated | 20 requests/min per IP |

---

## Authentication Headers

All protected endpoints require a JWT Bearer token:

```
Authorization: Bearer <jwt>
```

For service-to-service calls, use a service identity token (managed identity or signed service token) — never share user tokens between services.

---

## OpenAPI Specification Requirements

Every API must have a complete OpenAPI 3.1 spec **before implementation begins**:

- [ ] Every endpoint defined (path, method, operation ID)
- [ ] Every request body schema defined with required fields marked
- [ ] Every response schema defined (200, 201, 400, 401, 403, 404, 422, 429, 500)
- [ ] Every path/query parameter described with type and validation constraints
- [ ] Security scheme defined and applied to every protected endpoint
- [ ] `description` field populated for every operation
- [ ] Example request/response included for every operation

The spec is the contract. Implementation must conform to the spec — not the other way around.
