## Concept
HTTP headers contain metadata and instructions about a request or response.

## Important Headers
- `Content-Type` → body format
- `Accept` → preferred response format
- `Authorization` → authentication credentials/token
- `Host` → target hostname
- `User-Agent` → identifies the client
- `Server` → responding server/software
- `Cache-Control` → caching behavior

Examples:
```http
Content-Type: application/json
Accept: application/json
Authorization: Bearer <token>
```

## Request vs Response
```text
Accept        → normally request
Authorization → request
Content-Type  → request or response
Server        → normally response
```

## curl
```bash
curl -H "Accept: application/json" https://api.example.com/users
```

## SRE Relevance
Headers matter when troubleshooting authentication, caching, routing, proxies, content negotiation, APIs, and request tracing.

## Quick Revision
```text
Headers = metadata/instructions

Content-Type  → body format
Accept        → preferred response format
Authorization  → authentication
Host           → target hostname
Server         → responding server
Cache-Control  → caching
```

## Real-World Example
An API returns `401`. Run:
```bash
curl -v https://api.example.com/users
```
and check whether the `Authorization` header is missing or invalid.
