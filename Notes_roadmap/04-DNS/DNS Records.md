
| Record | Purpose                  |
| ------ | ------------------------ |
| A      | Hostname → IPv4          |
| AAAA   | Hostname → IPv6          |
| CNAME  | Hostname → hostname      |
| MX     | Mail server              |
| NS     | Authoritative nameserver |
| TXT    | Text/metadata            |

## Examples

### A
```text
api.example.com → 203.0.113.10
```
```bash
dig A api.example.com
```

### AAAA
```bash
dig AAAA api.example.com
```

### CNAME
```text
www.example.com → CNAME → example.com
example.com → A → 203.0.113.10
```
```bash
dig CNAME www.example.com
```

### MX
```bash
dig MX example.com
```

### NS
```bash
dig NS example.com
```

### TXT
Often used for domain verification and security metadata such as SPF.

## SRE Relevance
Know these records when debugging service endpoints, load balancers, domain migrations, email, and verification.

## Quick Revision
```text
A     → IPv4
AAAA  → IPv6
CNAME → hostname alias
MX    → mail
NS    → nameserver
TXT   → metadata
```
