# 12 — Docker Security

**Tags:** #docker #security #hardening #sre  
**Vault:** `SRE-2027/08-Docker/12-Docker-Security.md`

---

## Concept

Containers are isolated but not sandboxed like VMs. A misconfigured container can escape to the host. Docker security is about reducing the attack surface at every layer — image, runtime, daemon, and network.

---

## Key Points

- **Never run as root** — `USER appuser` in Dockerfile; root in container ≈ root on host if escaped
- **Pin image versions** — `alpine:3.19` not `alpine:latest`; scan images for CVEs
- **Read-only filesystem** — `--read-only` flag; combine with `--tmpfs /tmp` for write areas
- **Drop capabilities** — containers get subset of root privileges; drop what you don't need
- **No `--privileged`** — gives container full host access; avoid in production always
- **Never mount Docker socket** — `/var/run/docker.sock` = root on host; avoid in prod
- **Secrets via env at runtime** — never bake secrets into image with `ENV` or `COPY`
- **Limit resources** — `--memory` + `--cpus` prevent DoS from one container
- **Use `.dockerignore`** — prevents secrets and unnecessary files entering image
- **Scan images** — `docker scout` or Trivy for CVE scanning before push

---

## Diagram

```
Attack surface → reduce at each layer:
──────────────────────────────────────────────────
Image layer:      pin versions, scan CVEs, non-root user
                  no secrets baked in, minimal base (Alpine)

Runtime layer:    --read-only, --cap-drop ALL, no --privileged
                  --memory + --cpus, no socket mount

Network layer:    custom networks, --internal for DB networks
                  expose only necessary ports

Daemon layer:     TLS on Docker socket if remote
                  rootless Docker mode
```

---

## Example

```dockerfile
# Secure Dockerfile
FROM alpine:3.19

RUN addgroup -S app && adduser -S app -G app && \
    apk add --no-cache nodejs

WORKDIR /app
COPY --chown=app:app . .

USER app                    # non-root
EXPOSE 3000
ENTRYPOINT ["node", "index.js"]
```

```bash
# Secure docker run
docker run -d \
  --name api \
  --read-only \                        # read-only filesystem
  --tmpfs /tmp \                       # writable RAM area
  --cap-drop ALL \                     # drop all capabilities
  --cap-add NET_BIND_SERVICE \         # add only what's needed
  --memory=256m \                      # resource limit
  --security-opt no-new-privileges \   # prevent privilege escalation
  -p 3000:3000 \
  myapp:v1

# Scan image for CVEs (Docker Scout)
docker scout cves myapp:v1

# Scan with Trivy (open source)
trivy image myapp:v1
```

---

## SRE Relevance

A container escape in production means full host compromise. Security hardening is not optional — non-root user and no `--privileged` are the absolute minimum. Everything else is layered on top.

---

## Quick Revision

| Question | Answer |
|---|---|
| Why not run containers as root? | Root in container = potential root on host if escaped |
| What does `--privileged` do? | Gives container full access to host — never use in production |
| Why never mount Docker socket in prod? | `/var/run/docker.sock` = full Docker control = root on host |
| What does `--read-only` do? | Makes container root filesystem read-only |
| What does `--cap-drop ALL` do? | Removes all Linux capabilities from container |
| How to allow writes with `--read-only`? | Add `--tmpfs /tmp` for RAM-based write areas |
| Where should secrets go? | Runtime env vars (`--env-file`) not baked into image |
| How to scan image for vulnerabilities? | `docker scout cves` or `trivy image <name>` |
