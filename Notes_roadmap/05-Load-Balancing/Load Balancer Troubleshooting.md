## Common Errors

### 502 Bad Gateway
```text
LB / Proxy
    ↓
Backend
    ↓
Bad / unexpected response
```
**502 = bad upstream response**

### 503 Service Unavailable
Possible causes:
- No healthy backends
- Application overloaded
- Maintenance
- Resource exhaustion

**503 = service unavailable**

### 504 Gateway Timeout
```text
LB / Proxy
    ↓
Backend
    ↓
No response within timeout
```
**504 = upstream timeout**

## Troubleshooting 502
```text
502
 ↓
curl -v
 ↓
Check LB logs
 ↓
Check backend health
 ↓
Test backend directly
 ↓
Check application logs
 ↓
Check DB/downstream services
 ↓
Check recent deployment/config change
```

## Troubleshooting 503
```text
503
 ↓
Check health checks
 ↓
How many healthy backends?
 ↓
Check application capacity
 ↓
CPU / memory / connections
 ↓
Deployment / maintenance?
```

## Troubleshooting 504
```text
504
 ↓
Check request latency
 ↓
Check LB timeout
 ↓
Check application latency
 ↓
Check DB latency
 ↓
Check downstream API latency
```

## Useful Commands
```bash
curl -v https://api.example.com
nc -vz api.example.com 443
ss -lntp
nginx -t
```

Common NGINX logs:
```text
/var/log/nginx/access.log
/var/log/nginx/error.log
```

## SRE Relevance
Do not jump to "restart the load balancer." Find the first broken layer:
```text
Client
 ↓
DNS
 ↓
LB
 ↓
Backend
 ↓
Application
 ↓
Database / dependencies
```

## Quick Revision
```text
502 → Bad upstream response
503 → No service / no healthy backend
504 → Upstream timeout
```

## Real-World Example
Users suddenly receive `503`. All backends are unhealthy because a recent deployment introduced a broken configuration. The LB is working correctly; the application deployment is the root cause.
