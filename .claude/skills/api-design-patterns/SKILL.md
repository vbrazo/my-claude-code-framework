---
name: api-design-patterns
description: Designing REST APIs — naming resources, choosing methods and status codes, paginating, versioning, and writing the OpenAPI spec
---

# Designing REST APIs

## Naming resources

- Name collections with plural nouns: `/users`, `/orders`, `/products`
- Express relationships by nesting: `/users/{id}/orders`
- Stop nesting at two levels — past that, reach for query params or a top-level resource
- Stick to kebab-case: `/user-profiles`, not `/userProfiles`
- Keep verbs out of paths: instead of `/users/{id}/activate`, model the action as a resource — `POST /users/{id}/activation`

## Choosing HTTP methods

| Method | Purpose | Idempotent | Request Body | Success Code |
|--------|---------|------------|-------------|-------------|
| GET | Read resource(s) | Yes | No | 200 |
| POST | Create resource | No | Yes | 201 |
| PUT | Full replace | Yes | Yes | 200 |
| PATCH | Partial update | No | Yes | 200 |
| DELETE | Remove resource | Yes | No | 204 |

On a successful POST, hand back a `Location` header pointing at the resource you just created.

## Status codes

```
200 OK              - Successful read/update
201 Created         - Successful creation
204 No Content      - Successful delete
400 Bad Request     - Validation error (include field-level errors)
401 Unauthorized    - Missing or invalid authentication
403 Forbidden       - Authenticated but not authorized
404 Not Found       - Resource does not exist
409 Conflict        - State conflict (duplicate, version mismatch)
422 Unprocessable   - Semantically invalid (valid JSON, bad values)
429 Too Many Reqs   - Rate limited (include Retry-After header)
500 Internal Error  - Unhandled server error (never expose stack traces)
```

## Shape of an error response

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      { "field": "email", "message": "Must be a valid email address" },
      { "field": "age", "message": "Must be at least 18" }
    ]
  }
}
```

Keep error codes consistent everywhere, and document each one in your API reference so clients can branch on them.

## Cursor pagination (the default choice)

```
GET /users?limit=20&cursor=eyJpZCI6MTAwfQ

Response:
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTIwfQ",
    "has_more": true
  }
}
```

Cursors are the right call for large or fast-moving datasets. Encode them as opaque base64 blobs and never leak raw IDs through them.

## Offset pagination (only when it's genuinely simple)

```
GET /users?page=3&per_page=20

Response:
{
  "data": [...],
  "pagination": {
    "page": 3,
    "per_page": 20,
    "total": 245,
    "total_pages": 13
  }
}
```

Reserve offset paging for cases where the total count is cheap to compute and the dataset stays small.

## Filtering and sorting

```
GET /orders?status=pending&created_after=2025-01-01&sort=-created_at,+total
GET /products?category=electronics&price_min=100&price_max=500
GET /users?search=john&fields=id,name,email
```

Let clients trim payloads with a `fields` param, and prefix a sort field with `-` to flip it to descending.

## Versioning

Path-based versioning is the simplest thing that works:
```
/api/v1/users
/api/v2/users
```

Ground rules:
- Once v1 is public, treat it as frozen — add fields, never remove or repurpose them
- A new *required* field means a new version, not a patch
- Announce retirement with a `Sunset` header and at least six months' notice
- Keep no more than two versions live at once

## Headers worth standardizing

```
Content-Type: application/json
Accept: application/json
Authorization: Bearer <token>
X-Request-Id: <uuid>          # For tracing
X-RateLimit-Limit: 100        # Requests per window
X-RateLimit-Remaining: 47     # Remaining in window
X-RateLimit-Reset: 1700000000 # Window reset Unix timestamp
Retry-After: 30               # Seconds until rate limit resets
```

Echo `X-Request-Id` back on every response so a request can be traced end to end.

## Writing the OpenAPI spec

- Write the spec before the implementation and let it drive the work
- Share schemas through `$ref`: `$ref: '#/components/schemas/User'`
- Give every endpoint at least one `examples` entry
- Model polymorphic responses with `oneOf`/`anyOf`
- Generate client SDKs from the spec rather than hand-writing them
- Validate incoming requests against the spec in middleware

```yaml
paths:
  /users/{id}:
    get:
      operationId: getUser
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: User found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          $ref: '#/components/responses/NotFound'
```

## Rate limiting

- Scope limits per user and per endpoint
- Prefer a sliding window over a fixed one to avoid burst-at-the-boundary
- Reject with `429` and a `Retry-After` header
- Leave health checks and auth endpoints unthrottled
- Log throttled requests so you can spot abuse patterns
