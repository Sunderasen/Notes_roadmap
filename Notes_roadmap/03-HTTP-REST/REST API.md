## Concept
A REST API is an HTTP-based interface for interacting with resources.

Common resources:
```text
/users
/orders
/products
```

## Typical Operations
```text
GET    /users       → List users
GET    /users/10    → Get user 10
POST   /users       → Create user
PUT    /users/10    → Replace user 10
PATCH  /users/10    → Partially update user 10
DELETE /users/10    → Delete user 10
```

## Key Components
```text
URL
Method
Headers
Request Body
Response
Status Code
```

## Idempotency
For SRE work, understand that repeating an idempotent operation should have the same intended effect.

Common examples:
```text
GET    → idempotent
PUT    → idempotent
DELETE → generally idempotent
POST   → generally not idempotent
```

This matters when designing retries.

## SRE Relevance
REST APIs are common in microservices, Kubernetes, cloud APIs, monitoring systems, internal services, and automation.

## Quick Revision
```text
REST API = Resources + HTTP methods + status codes + headers
```

## Real-World Example
A service-to-service call returns `500`. Reproduce the request with curl, inspect the status/body/headers, then trace the request through the LB and application logs.
