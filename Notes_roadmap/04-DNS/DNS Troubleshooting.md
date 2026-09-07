## Basic Flow
```text
1. Check DNS
2. Check TCP connectivity
3. Check HTTP
4. Check Load Balancer
5. Check Backend
6. Check Application
7. Check Dependencies
```

## Step 1 — DNS
```bash
dig +short api.example.com
```
Ask:
- Did DNS return an IP?
- Is it the expected IP?

## Step 2 — TCP
```bash
nc -vz api.example.com 443
```

## Step 3 — HTTP
```bash
curl -v https://api.example.com
```

## Failure Path
```text
DNS
 ↓
Load Balancer
 ↓
Backend
 ↓
Application
 ↓
Database / dependencies
```

## Common Problems
- Wrong IP
- Wrong CNAME
- Resolver/cache issue
- Authoritative DNS issue
- Incorrect TTL expectations

## SRE Relevance
Do not immediately restart application servers. First prove which layer is broken.

## Quick Revision
```bash
dig +short domain.com
nc -vz domain.com 443
curl -v https://domain.com
```

## Real-World Example
Users report an outage. DNS resolves to an unexpected IP, while application servers are healthy. The root cause is incorrect DNS configuration.
