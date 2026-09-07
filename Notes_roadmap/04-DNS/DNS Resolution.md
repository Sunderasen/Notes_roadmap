## Concept
DNS resolution is the process of finding the record associated with a hostname.

```text
curl https://api.example.com
            ↓
DNS lookup
            ↓
203.0.113.10
            ↓
Connect to IP
```

## Simplified Resolution
```text
Client
  ↓
Recursive Resolver
  ↓
Root
  ↓
TLD (.com)
  ↓
Authoritative DNS
  ↓
IP address
```

Caching can skip parts of this process.

## DNS Cache and TTL
Resolvers cache DNS responses according to TTL.

```text
DNS record
   ↓
TTL
   ↓
Resolver cache
```

DNS changes may therefore take time to appear everywhere.

## Commands
```bash
dig example.com
dig +short example.com
dig @1.1.1.1 example.com
dig +trace example.com
```

## SRE Relevance
Useful for DNS outages, traffic migrations, load balancer changes, service discovery, unexpected IPs, and caching issues.

## Quick Revision
```text
Application
 ↓
DNS lookup
 ↓
Resolver/cache
 ↓
DNS record
 ↓
IP address
 ↓
Connection
```

## Real-World Example
During a migration from one LB to another, some users still reach the old LB because recursive resolvers have cached the old DNS answer until its TTL expires.
