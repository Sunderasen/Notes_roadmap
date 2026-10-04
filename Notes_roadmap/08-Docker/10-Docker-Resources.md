# 10 — Docker Resources

**Tags:** #docker #resources #cgroups #memory #cpu  
**Vault:** `SRE-2027/08-Docker/10-Docker-Resources.md`

---

## Concept

By default containers have no resource limits — they can consume all host CPU and memory. Docker uses **cgroups** to enforce limits. Without limits, a single container can starve the entire host.

---

## Key Points

- `--memory` → hard memory limit; container is OOM-killed if exceeded (exit code 137)
- `--memory-swap` → memory + swap combined; set equal to `--memory` to disable swap
- `--cpus` → limit CPU cores (e.g. `0.5` = half a core)
- `--cpu-shares` → relative CPU weight (default 1024); only enforced under contention
- No limits set → container can use 100% of host resources
- OOM kill = exit code `137` — always check `docker stats` first
- `docker stats` shows live usage vs limits
- Limits enforce at cgroup level — kernel does the actual enforcement
- In production: always set limits; sized based on profiling not guessing

---

## Diagram

```
Host: 4 CPU cores, 8GB RAM
──────────────────────────────────────────
┌─────────────────┐  ┌─────────────────┐
│  Container A    │  │  Container B    │
│  --cpus=1.0     │  │  --cpus=0.5     │
│  --memory=512m  │  │  --memory=256m  │
│                 │  │                 │
│  Gets: 1 core   │  │  Gets: 0.5 core │
│        512MB    │  │        256MB    │
└─────────────────┘  └─────────────────┘
Remaining: 2.5 cores, 7.25GB → other containers / host
```

---

## Example

```bash
# Set memory + CPU limits
docker run -d \
  --name api \
  --memory=256m \
  --memory-swap=256m \
  --cpus=0.5 \
  myapp:v1

# Check live usage
docker stats api

# Snapshot (no live refresh)
docker stats --no-stream api

# Output:
# NAME   CPU %   MEM USAGE / LIMIT    MEM %   NET I/O
# api    12.3%   180MiB / 256MiB      70.3%   ...

# Check if OOM killed
docker inspect api --format '{{.State.OOMKilled}}'   # true/false
docker inspect api --format '{{.State.ExitCode}}'    # 137 = OOM
```

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:v1
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          memory: 128M
```

---

## SRE Relevance

Unset resource limits are a production risk — one misbehaving container can OOM the host and take down everything. Always set limits, monitor with `docker stats`, and treat exit code `137` as a resource sizing problem.

---

## Quick Revision

| Question | Answer |
|---|---|
| What enforces Docker resource limits? | Linux cgroups |
| Flag to set memory limit? | `--memory=256m` |
| What happens if container exceeds memory limit? | OOM killed — exit code 137 |
| Flag to limit CPU? | `--cpus=0.5` (0.5 = half a core) |
| `--cpus` vs `--cpu-shares`? | `--cpus` = hard limit; `--cpu-shares` = relative weight under contention |
| How to check live resource usage? | `docker stats` |
| How to verify OOM kill? | `docker inspect <name> --format '{{.State.OOMKilled}}'` |
| Default limits if none set? | None — container can use all host resources |
