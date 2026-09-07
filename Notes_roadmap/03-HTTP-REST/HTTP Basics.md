## Concept
HTTP (HyperText Transfer Protocol) is the protocol used for communication between clients and servers.

```text
Client
  ↓ HTTP Request
Server
  ↓ HTTP Response
Client
```

## HTTP Request
A request usually contains:
- Method
- URL
- Headers
- Body (optional)

Example:
```http
GET /users/10 HTTP/1.1
Host: api.example.com
Accept: application/json
```

## HTTP Response
A response usually contains:
- Status code
- Headers
- Body (optional)

Example:
```http
HTTP/1.1 200 OK
Content-Type: application/json

{"id":10,"name":"John"}
```

## HTTP vs HTTPS
```text
HTTP  → unencrypted
HTTPS → HTTP + TLS encryption
```

Production APIs should normally use HTTPS.

## SRE Relevance
HTTP is fundamental for API troubleshooting, load balancers, reverse proxies, health checks, monitoring, and service-to-service communication.

## Useful Command
```bash
curl -v https://api.example.com
```

## Quick Revision
```text
HTTP = Client ↔ Server communication
Request  → Method + URL + Headers + Body
Response → Status + Headers + Body
HTTPS = HTTP + TLS
```

## Real-World Example
An application is reported as "down". Before restarting anything:
```bash
curl -v https://api.example.com/health
```
This helps determine whether the problem is DNS, TLS, the load balancer, or the application.

Resource - https://github.com/iam-veeramalla/http-status-codes
