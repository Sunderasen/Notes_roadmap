# L4 vs L7 Load Balancing

## L4 Load Balancing
Works with network/transport information:
- IP
- TCP
- UDP
- Port
- Connections

```text
Client → TCP:443 → L4 LB → Backend
```

L4 does not understand HTTP paths or headers.

## L7 Load Balancing
Understands application-layer information:
- Hostname
- URL path
- HTTP headers
- Cookies
- HTTP method

Example:
```text
/api/*     → API servers
/images/*  → Image servers
/admin/*   → Admin servers
```

## TLS Termination
L7 load balancers commonly terminate TLS:
```text
Client
  ↓ HTTPS
L7 LB
  ↓ HTTP/HTTPS
Backend
```

## Key Difference
```text
L4 → Connection-level routing
L7 → Application/request-level routing
```

## SRE Relevance
Helps troubleshoot TCP connectivity, TLS, HTTP errors, path routing, and reverse proxies.

## Quick Revision
```text
L4 → IP / TCP / UDP / Port
L7 → HTTP / Host / Path / Headers / Cookies
```

## Real-World Example
Requests to `/api/*` and `/images/*` need different backend pools. Application-aware routing requires L7 behavior.
