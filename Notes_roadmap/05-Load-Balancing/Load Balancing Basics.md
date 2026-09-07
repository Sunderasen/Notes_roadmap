## Concept
A Load Balancer distributes incoming traffic across multiple backend servers.

```text
             ┌── App 1
Client → LB ─┼── App 2
             └── App 3
```

## Why Use a Load Balancer?
- Distribute traffic
- Improve availability
- Scale horizontally
- Remove unhealthy servers
- Hide backend IPs
- TLS termination
- Traffic routing

## Reverse Proxy
A load balancer often acts as a reverse proxy:
```text
Client
  ↓
LB / Reverse Proxy
  ↓
Backend
```

## SRE Relevance
Important for high availability, scaling, deployments, health checks, traffic management, and incident troubleshooting.

## Quick Revision
```text
Client
 ↓
Load Balancer
 ↓
Healthy backend servers
```

## Real-World Example
Traffic increases 5x during a product launch. Multiple application servers sit behind the LB so traffic can be distributed and the service can scale horizontally.
