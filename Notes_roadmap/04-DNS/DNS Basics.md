## Concept
DNS (Domain Name System) translates hostnames into IP addresses and provides other service information.

```text
api.example.com
       ↓
203.0.113.10
```

## Simplified Flow
```text
Client
  ↓
Recursive Resolver
  ↓
DNS hierarchy
  ↓
Authoritative DNS
  ↓
Record
```

Caching often short-circuits this process.

## SRE Relevance
DNS failures can look like application failures. Common issues include wrong IPs, incorrect records, resolver problems, caching/TTL issues, and bad CNAMEs.

## Commands
```bash
dig example.com
dig +short example.com
nslookup example.com
```

## Quick Revision
```text
DNS = name → IP / service information
```

## Real-World Example
An application is healthy but unreachable. `dig +short api.example.com` returns an old load balancer IP. The application is fine; DNS is the problem.
