# HTTP Status Codes

## Concept
HTTP status codes tell the client what happened to the request.

## 2xx — Success
- `200 OK` → request succeeded
- `201 Created` → resource created
- `204 No Content` → success with no response body

**200 means the request succeeded, not "connection established".**

## 4xx — Client/Request Problem
- `400 Bad Request` → invalid request
- `401 Unauthorized` → authentication missing/invalid
- `403 Forbidden` → authenticated but not authorized
- `404 Not Found` → resource does not exist
- `429 Too Many Requests` → rate limited

```text
401 → "Who are you?"
403 → "I know who you are, but you're not allowed."
```

## 5xx — Server/Infrastructure Problem
- `500 Internal Server Error` → unexpected server/application error
- `502 Bad Gateway` → gateway/proxy received a bad or unexpected upstream response
- `503 Service Unavailable` → service unavailable
- `504 Gateway Timeout` → gateway/proxy timed out waiting for upstream

```text
502 → Bad upstream response
503 → Service unavailable
504 → Upstream timeout
```

## SRE Relevance
Status codes are often the first signal during an incident. Correlate them with latency, traffic, LB logs, application logs, and backend health.

## Quick Revision
```text
200 → Success
400 → Bad request
401 → Authentication
403 → Authorization
404 → Not found
429 → Rate limited
500 → Internal server error
502 → Bad upstream response
503 → Service unavailable
504 → Upstream timeout
```

## Real-World Example
If monitoring shows a spike in `504`, check application latency, database latency, downstream services, and LB timeout settings before assuming the LB itself is broken.
