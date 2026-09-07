## Concept
HTTP methods tell the server what operation the client wants to perform.

| Method | Purpose | Example |
|---|---|---|
| GET | Read data | `GET /users/10` |
| POST | Create/send data | `POST /users` |
| PUT | Replace a resource | `PUT /users/10` |
| PATCH | Partially update | `PATCH /users/10` |
| DELETE | Delete resource | `DELETE /users/10` |

## Examples
```bash
curl https://api.example.com/users
curl -X POST https://api.example.com/users
curl -X DELETE https://api.example.com/users/10
```

POST JSON:
```bash
curl -X POST   -H "Content-Type: application/json"   -d '{"name":"John"}'   https://api.example.com/users
```

## SRE Relevance
Useful when reproducing API issues, debugging application logs, investigating unexpected writes, understanding traffic, and creating health checks.

## Quick Revision
```text
GET     → Read
POST    → Create
PUT     → Replace
PATCH   → Partial update
DELETE  → Delete
```

## Real-World Example
A developer reports that `GET /orders/123` is failing. Reproduce it with:
```bash
curl -v https://api.example.com/orders/123
```
Then compare the request with LB and application logs.
