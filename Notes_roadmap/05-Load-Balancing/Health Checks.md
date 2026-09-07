# Load Balancer Health Checks

## Concept
Health checks allow a load balancer to determine whether a backend can receive traffic.

```text
LB
├── App 1 → Healthy
├── App 2 → Healthy
└── App 3 → Unhealthy
```

Traffic is sent only to healthy backends.

## Types

### TCP Health Check
Checks whether a TCP connection can be established.

### HTTP Health Check
Calls an endpoint such as:
```text
GET /health
```
Expected response might be:
```text
200 OK
```

## Health Check Flow
```text
LB
 ↓
Health check
 ↓
Backend
 ↓
Healthy?
 ├── Yes → Send traffic
 └── No  → Remove from rotation
```

## Important Point
A server can be network-reachable while its application is unhealthy.

An HTTP readiness/health endpoint can detect this better than a simple TCP check.

## SRE Relevance
Bad health checks can remove healthy servers or keep unhealthy servers in rotation, causing outages.

## Quick Revision
```text
Healthy   → receive traffic
Unhealthy → remove from traffic
```

## Real-World Example
A backend's port is open, but the application cannot connect to its database. TCP checks pass, while a meaningful HTTP health/readiness check fails and removes the backend from traffic.
