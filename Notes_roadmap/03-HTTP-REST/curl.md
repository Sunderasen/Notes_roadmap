## Concept
`curl` is a command-line tool used to make HTTP requests and troubleshoot APIs/services.

## Important Options
```text
-i → include response headers
-v → verbose/debug
-H → add header
-d → send data/body
-X → specify method
-s → silent
-o → save output
```

## Examples
```bash
curl https://example.com
curl -i https://example.com
curl -v https://example.com
curl -H "Content-Type: application/json" https://example.com
curl -d '{"name":"John"}' https://example.com
curl -X POST https://example.com
curl -s https://example.com
curl -o output.html https://example.com
```

## POST JSON
```bash
curl -X POST   -H "Content-Type: application/json"   -d '{"name":"John"}'   https://api.example.com/users
```

## SRE Relevance
Use curl for API failures, HTTP errors, TLS issues, load balancers, reverse proxies, service connectivity, and health checks.

## Quick Revision
```text
-i → headers
-v → verbose/debug
-H → header
-d → data/body
-X → method
-s → silent
-o → output file
```

## Real-World Example
For a `502` incident:
```bash
curl -v https://api.example.com
```
Check whether the request reaches the LB and whether the upstream response is valid.
