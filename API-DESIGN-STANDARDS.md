# API Design Standards

## Executive Summary
Comprehensive REST and GraphQL API design guidelines ensuring consistency, security, performance, and developer experience across all services.

## Part 1: RESTful API Design

### 1.1 URL Structure

#### Resource Naming Conventions
- Use nouns for resource names (not verbs)
- Use plural forms for collections
- Use hyphens for multi-word resource names
- Use lowercase letters

```
✓ /api/v1/users
✓ /api/v1/users/{id}/orders
✓ /api/v1/user-preferences

✗ /api/v1/getUsers
✗ /api/v1/User
✗ /api/v1/users_orders
```

#### Versioning
- API version in URL path (v1, v2, etc.)
- Version incrementally on breaking changes
- Support multiple versions for migration period

```
/api/v1/users
/api/v2/users    # Breaking changes only
```

### 1.2 HTTP Methods

| Method | Purpose | Idempotent | Safe |
|--------|---------|------------|------|
| GET | Retrieve resource | Yes | Yes |
| POST | Create resource | No | No |
| PUT | Replace entire resource | Yes | No |
| PATCH | Partial update | No | No |
| DELETE | Remove resource | Yes | No |

#### Method Usage Rules

```
GET /api/v1/users                # List all users
GET /api/v1/users/{id}           # Get single user
POST /api/v1/users               # Create user
PUT /api/v1/users/{id}           # Replace user
PATCH /api/v1/users/{id}         # Update user fields
DELETE /api/v1/users/{id}        # Delete user
```

### 1.3 Status Codes

#### 2xx Success
- 200 OK: Successful GET, PUT, PATCH
- 201 Created: Successful POST
- 202 Accepted: Async processing started
- 204 No Content: Successful DELETE or empty response

#### 4xx Client Error
- 400 Bad Request: Invalid parameters
- 401 Unauthorized: Missing/invalid auth
- 403 Forbidden: Authenticated but not authorized
- 404 Not Found: Resource doesn't exist
- 409 Conflict: Resource conflict (e.g., duplicate)
- 422 Unprocessable Entity: Validation errors
- 429 Too Many Requests: Rate limit exceeded

#### 5xx Server Error
- 500 Internal Server Error: Server error
- 502 Bad Gateway: Upstream service error
- 503 Service Unavailable: Maintenance/overload
- 504 Gateway Timeout: Upstream timeout

### 1.4 Request/Response Format

#### Standard Request

```json
{
  "email": "user@example.com",
  "name": "John Doe",
  "preferences": {
    "notifications": true,
    "theme": "dark"
  }
}
```

#### Standard Response

```json
{
  "status": "success",
  "data": {
    "id": 123,
    "email": "user@example.com",
    "name": "John Doe",
    "created_at": "2024-01-03T05:15:00Z"
  },
  "meta": {
    "request_id": "req-12345abc",
    "timestamp": "2024-01-03T05:15:00Z"
  }
}
```

#### Error Response

```json
{
  "status": "error",
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": [
      {
        "field": "email",
        "message": "Email is required"
      }
    ]
  },
  "meta": {
    "request_id": "req-12345abc"
  }
}
```

### 1.5 Pagination

```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 150,
    "pages": 8,
    "has_next": true,
    "has_prev": false
  }
}
```

Query Parameters:
```
GET /api/v1/users?page=1&per_page=20&sort=-created_at
```

### 1.6 Filtering & Sorting

#### Filtering
```
GET /api/v1/orders?status=completed&user_id=123
GET /api/v1/orders?created_at.gte=2024-01-01&created_at.lte=2024-01-31
GET /api/v1/orders?amount.gt=100&amount.lt=1000
```

#### Sorting
```
GET /api/v1/orders?sort=created_at       # Ascending
GET /api/v1/orders?sort=-created_at      # Descending
GET /api/v1/orders?sort=-created_at,amount  # Multiple fields
```

### 1.7 Field Selection

```
GET /api/v1/users?fields=id,email,name
GET /api/v1/users?exclude=password,api_key
```

## Part 2: Authentication & Authorization

### 2.1 Bearer Token Authentication

```
Authorization: Bearer <access_token>
```

### 2.2 Rate Limiting Headers

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1609459200
```

### 2.3 CORS Headers

```
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

## Part 3: GraphQL Design

### 3.1 Query Structure

```graphql
query GetUser($id: ID!) {
  user(id: $id) {
    id
    email
    name
    orders(first: 10) {
      edges {
        node {
          id
          total
          status
        }
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
}
```

### 3.2 Mutation Structure

```graphql
mutation CreateOrder($input: CreateOrderInput!) {
  createOrder(input: $input) {
    order {
      id
      status
      total
    }
    errors {
      field
      message
    }
  }
}
```

## Part 4: API Documentation

### 4.1 OpenAPI/Swagger Specification

```yaml
openapi: 3.0.0
info:
  title: User API
  version: 1.0.0
servers:
  - url: https://api.example.com/v1
paths:
  /users:
    get:
      summary: List users
      parameters:
        - name: page
          in: query
          schema:
            type: integer
      responses:
        '200':
          description: List of users
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserList'
    post:
      summary: Create user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserInput'
      responses:
        '201':
          description: User created
```

## Part 5: Webhook Design

### 5.1 Event Format

```json
{
  "id": "evt_123abc",
  "type": "order.completed",
  "timestamp": "2024-01-03T05:15:00Z",
  "data": {
    "order_id": 456,
    "user_id": 123,
    "total": 99.99
  }
}
```

### 5.2 Delivery Guarantees

- At-least-once delivery with retries
- Exponential backoff: 5s, 30s, 2m, 5m, 30m
- Max retry attempts: 5
- Signature verification (HMAC-SHA256)

## Part 6: Security Standards

### 6.1 HTTPS Only
- All endpoints require HTTPS
- HSTS header with 1-year max-age
- No sensitive data in URLs

### 6.2 Input Validation

```python
from pydantic import BaseModel, EmailStr, validator

class CreateUserRequest(BaseModel):
    email: EmailStr
    name: str
    age: int
    
    @validator('name')
    def name_length(cls, v):
        if len(v) < 2 or len(v) > 100:
            raise ValueError('Name must be 2-100 characters')
        return v
    
    @validator('age')
    def age_range(cls, v):
        if v < 18 or v > 120:
            raise ValueError('Age must be 18-120')
        return v
```

### 6.3 SQL Injection Prevention
- Use parameterized queries
- ORM frameworks (SQLAlchemy, Sequelize)
- No string concatenation in queries

### 6.4 CORS Security
- Whitelist specific origins
- Never use wildcard (*) for credentials
- Validate preflight requests

## Part 7: Error Handling

### 7.1 Error Code Taxonomy

```
BUSINESS_ERROR:RESOURCE_NOT_FOUND
BUSINESS_ERROR:INSUFFICIENT_FUNDS
BUSINESS_ERROR:DUPLICATE_EMAIL
VALIDATION_ERROR:INVALID_EMAIL
VALIDATION_ERROR:MISSING_FIELD
AUTH_ERROR:INVALID_TOKEN
RATE_LIMIT_ERROR:TOO_MANY_REQUESTS
```

### 7.2 Comprehensive Error Response

```json
{
  "status": "error",
  "error": {
    "code": "VALIDATION_ERROR:INVALID_EMAIL",
    "message": "The provided email address is invalid",
    "documentation_url": "https://docs.example.com/errors#invalid_email",
    "request_id": "req-12345abc",
    "details": {
      "field": "email",
      "value": "not-an-email"
    }
  }
}
```

## Part 8: Performance Standards

### 8.1 Response Time Targets
- 95th percentile: <200ms
- 99th percentile: <500ms
- All endpoints including database queries

### 8.2 Caching Headers

```
Cache-Control: public, max-age=3600
ETag: "12345abc"
Last-Modified: Wed, 03 Jan 2024 05:15:00 GMT
Vary: Authorization
```

### 8.3 Compression
- Enable gzip/brotli for responses >1KB
- Accept-Encoding header support

## Success Metrics

- API uptime: 99.99%
- Response time p95: <200ms
- Documentation completeness: 100%
- Breaking changes: Zero without deprecation
- Security audits: Monthly
