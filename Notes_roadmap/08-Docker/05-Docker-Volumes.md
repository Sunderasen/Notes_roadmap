# Docker Volumes

**Tags:** #docker #volumes #storage #persistence #sre  
**Vault Path:** `SRE-2027/08-Docker/05-Docker-Volumes.md`

---

## 1. Concept

By default, everything a container writes to its filesystem **dies with the container**. The moment you `docker rm`, it's gone. Docker volumes solve this by providing storage that lives **outside the container lifecycle**. Understanding volumes — and which type to use when — is essential for running stateful workloads (databases, caches, file uploads) reliably in production.

---

## 2. Definition — Why Volumes Exist

```
WITHOUT volumes:

  docker run mysql → Container writes data to its writable layer
  docker rm mysql  → All data gone. Database wiped.

WITH a named volume:

  docker run -v dbdata:/var/lib/mysql mysql
  docker rm mysql  → Container gone
  docker run -v dbdata:/var/lib/mysql mysql  → Data still there
```

Volumes also solve:
- **Sharing data** between multiple containers
- **Injecting config** from host into containers
- **Performance** — bypass the copy-on-write overhead of the container layer
- **Backups** — named volumes have a known location on the host

---

## 3. The Three Mount Types

```
┌──────────────────────────────────────────────────────────────────┐
│                        DOCKER HOST                               │
│                                                                  │
│  ┌───────────────┐  ┌────────────────────┐  ┌────────────────┐  │
│  │  Bind Mount   │  │   Named Volume     │  │  tmpfs Mount   │  │
│  │               │  │                   │  │                │  │
│  │ /your/path    │  │ /var/lib/docker/  │  │  RAM only      │  │
│  │ on host disk  │  │ volumes/mydata/   │  │  no disk       │  │
│  └──────┬────────┘  └────────┬──────────┘  └───────┬────────┘  │
│         │                   │                      │           │
└─────────┼───────────────────┼──────────────────────┼───────────┘
          │                   │                      │
     ┌────▼───────────────────▼──────────────────────▼────┐
     │                   CONTAINER                         │
     │    /app/config    /var/lib/mysql    /tmp/cache      │
     └─────────────────────────────────────────────────────┘
```

---

## 4. Mount Type 1 — Named Volume

### What it is
Docker creates and manages the storage. You refer to it by name. Docker handles the host path (`/var/lib/docker/volumes/<name>/_data`).

### Syntax

```bash
# -v syntax
docker run -v mydata:/var/lib/mysql mysql

# --mount syntax (explicit, preferred in production)
docker run --mount type=volume,source=mydata,target=/var/lib/mysql mysql

# Read-only volume
docker run --mount type=volume,source=config,target=/app/config,readonly myapp
```

### Volume Management

```bash
# Create explicitly (optional — Docker creates on first run)
docker volume create mydata

# List all volumes
docker volume ls
docker volume ls --filter dangling=true      # volumes not attached to any container

# Inspect — find actual host path
docker volume inspect mydata

# Remove
docker volume rm mydata                      # fails if volume in use
docker volume prune                          # remove all unused volumes
docker volume prune --filter until=24h       # unused for more than 24h
```

### Inspect output

```json
[
  {
    "CreatedAt": "2025-09-01T10:00:00Z",
    "Driver": "local",
    "Mountpoint": "/var/lib/docker/volumes/mydata/_data",
    "Name": "mydata",
    "Scope": "local"
  }
]
```

Data actually lives at the `Mountpoint` — useful for direct host access or backup scripts.

### Examples

```bash
# MySQL with persistent data
docker run -d \
  --name db \
  -e MYSQL_ROOT_PASSWORD=secret \
  -v mysqldata:/var/lib/mysql \
  mysql:8

# Two containers sharing the same volume
docker run -d --name writer -v shared:/data writer-app
docker run -d --name reader -v shared:/data:ro reader-app
# reader gets read-only access; writer can write

# Pre-populate a volume from a container
docker run --rm \
  -v mydata:/target \
  ubuntu \
  bash -c "echo 'initial data' > /target/init.txt"
```

### When to use
- Production databases (MySQL, Postgres, MongoDB)
- App file uploads that must survive redeploys
- Shared state between containers
- Anything you want managed and visible via `docker volume ls`

---

## 5. Mount Type 2 — Bind Mount

### What it is
Maps a **specific host path** directly into the container. You control where the data lives on the host. The host filesystem is the source of truth.

### Syntax

```bash
# -v syntax
docker run -v /host/path:/container/path myapp
docker run -v $(pwd)/config:/app/config:ro myapp     # read-only

# --mount syntax
docker run --mount type=bind,source=/host/path,target=/container/path myapp
docker run --mount type=bind,source=$(pwd)/config,target=/app/config,readonly myapp
```

### Key Behaviour

```bash
# -v creates host path if it doesn't exist (silently, as directory)
docker run -v /nonexistent:/app myapp          # creates /nonexistent as dir

# --mount ERRORS if host path doesn't exist
docker run --mount type=bind,source=/nonexistent,target=/app myapp
# Error: invalid mount config: stat /nonexistent: no such file or directory
# ← This is why --mount is safer in scripts
```

### Examples

```bash
# Local dev: live code reload
docker run -d \
  --name dev \
  -v $(pwd)/src:/app/src \
  -p 3000:3000 \
  myapp:dev

# Inject config file (read-only)
docker run -d \
  --name web \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  nginx

# Mount host logs dir (read-only for monitoring)
docker run -d \
  --name logparser \
  -v /var/log/app:/logs:ro \
  log-parser

# Mount Docker socket (CI builds — security risk)
docker run -v /var/run/docker.sock:/var/run/docker.sock docker docker ps
```

### When to use
- **Local development** — live code changes without rebuilding
- **Config injection** — mount `nginx.conf`, `.env`, certs
- **Log access** — read host logs from inside a container
- **Not recommended for production data** — host path must exist and be managed manually

---

## 6. Mount Type 3 — tmpfs

### What it is
Mounts a **temporary filesystem in RAM**. Data never touches disk. Disappears when the container stops. Fast and ephemeral.

### Syntax

```bash
# --mount syntax
docker run --mount type=tmpfs,target=/tmp myapp

# With size limit
docker run --mount type=tmpfs,target=/tmp,tmpfs-size=100m myapp

# With permissions (octal)
docker run --mount type=tmpfs,target=/tmp,tmpfs-size=100m,tmpfs-mode=1770 myapp

# Shorthand (Linux only)
docker run --tmpfs /tmp myapp
docker run --tmpfs /tmp:size=100m,mode=1770 myapp
```

### Examples

```bash
# Sensitive data that must never hit disk
docker run -d \
  --name payments \
  --mount type=tmpfs,target=/tmp/tokens,tmpfs-size=50m \
  payments-service:v3

# Read-only root filesystem + tmpfs for writes (security hardening)
docker run -d \
  --name secure-app \
  --read-only \
  --mount type=tmpfs,target=/tmp \
  --mount type=tmpfs,target=/var/run \
  myapp:v2
```

### When to use
- Sensitive temporary data (auth tokens, session data, PII processing)
- High-speed scratch space — RAM I/O is much faster than disk
- Containers with `--read-only` root filesystem that still need to write somewhere
- Compliance: data must not persist to disk

### Limitations
- Data lost on container stop — not for anything you need to keep
- Uses host RAM — factor into memory resource planning
- Not supported identically on Docker Desktop (Mac/Windows) vs Linux

---

## 7. The Two Syntaxes — `-v` vs `--mount`

| | `-v` / `--volume` | `--mount` |
|---|---|---|
| Syntax | Short: `source:target:opts` | Verbose: `key=value` pairs |
| Readability | Compact | Explicit and clear |
| Missing host path (bind) | Creates dir silently | Errors immediately |
| tmpfs options | Limited | Full options (`tmpfs-size`, `tmpfs-mode`) |
| Volume driver options | Not supported | `volume-opt=key=value` |
| Production scripts | Not ideal | Preferred |
| Docker Compose | Both work | `--mount` style maps to `volumes:` config |

**Rule:** Use `-v` for quick local testing. Use `--mount` in scripts, CI, and production configs.

---

## 8. Anonymous Volumes

A volume with no name — Docker generates a random ID.

```bash
# Created with -v and no source name
docker run -v /var/lib/mysql mysql      # anonymous volume

# Or specified in Dockerfile via VOLUME instruction
VOLUME /var/lib/mysql                   # anonymous volume created at run time

# List them
docker volume ls
# DRIVER    VOLUME NAME
# local     a1b2c3d4e5f6...             ← anonymous (random ID)
# local     mysqldata                   ← named

# Remove with container
docker rm -v web                        # removes anonymous volumes too
# Named volumes are NOT removed with -v flag
```

**Problem with anonymous volumes:** Hard to manage — you can't reference them by name for backups or sharing. Generally prefer named volumes.

---

## 9. Volume Drivers

Named volumes support pluggable drivers — storage can live on network-attached storage instead of local disk.

```bash
# NFS volume (shared across multiple hosts)
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/exports/appdata \
  nfsdata

docker run --mount type=volume,source=nfsdata,target=/data myapp
```

| Driver | Backend | Use Case |
|---|---|---|
| `local` (default) | Host disk | Single-node, dev, simple prod |
| `local` + NFS opts | NFS server | Shared storage across hosts |
| `rexray/ebs` | AWS EBS | Cloud block storage |
| `rexray/efs` | AWS EFS | Cloud shared storage |
| Custom plugins | S3, GCS, Azure | Object storage as volume |

**SRE note:** In Kubernetes, this becomes **PersistentVolumes** and **StorageClasses** — same concept with a richer API.

---

## 10. Volumes in Docker Compose

```yaml
version: "3.9"

services:
  db:
    image: postgres:15
    volumes:
      - dbdata:/var/lib/postgresql/data          # named volume
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro  # bind mount

  app:
    image: myapp:v2
    volumes:
      - ./src:/app/src                           # bind mount (dev)
      - uploads:/app/uploads                     # named volume
      - type: tmpfs                              # tmpfs
        target: /tmp/processing
        tmpfs:
          size: 100000000                        # 100MB in bytes

  cache:
    image: redis:7
    tmpfs:
      - /data                                    # tmpfs shorthand in compose

# All named volumes must be declared here
volumes:
  dbdata:
  uploads:
    driver: local
  nfsdata:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=192.168.1.100,rw"
      device: ":/exports/data"
```

---

## 11. Backup and Restore Pattern

```bash
# === BACKUP A VOLUME ===
# Spin up a helper container, mount the volume + a host dir
docker run --rm \
  -v dbdata:/source:ro \
  -v $(pwd)/backups:/backup \
  ubuntu \
  tar czf /backup/dbdata-$(date +%Y%m%d).tar.gz -C /source .

# === RESTORE A VOLUME ===
docker volume create dbdata-restored
docker run --rm \
  -v dbdata-restored:/target \
  -v $(pwd)/backups:/backup:ro \
  ubuntu \
  tar xzf /backup/dbdata-20250901.tar.gz -C /target

# === COPY VOLUME TO ANOTHER HOST ===
# Step 1: export on source host
docker run --rm -v dbdata:/data ubuntu tar czf - /data | ssh user@remotehost "docker run --rm -i -v dbdata:/data ubuntu tar xzf - -C /"
```

---

## 12. SRE Decision Tree

```
Need to store data?
│
├── Must survive container restarts?
│   ├── YES
│   │   ├── Must be shared across hosts?  → Volume driver (NFS/EFS)
│   │   ├── Shared between containers?    → Named Volume (same name)
│   │   └── Single container, single host → Named Volume
│   │
│   └── NO
│       ├── Must never touch disk?        → tmpfs
│       └── Development / config inject?  → Bind Mount
│
└── Production database?                  → Named Volume (always)
    Injecting config read-only?           → Bind Mount :ro
    Sensitive temp data?                  → tmpfs
    Dev with live code reload?            → Bind Mount
```

---

## 13. Commands — Quick Reference

```bash
# === NAMED VOLUME ===
docker volume create mydata
docker volume ls
docker volume ls --filter dangling=true
docker volume inspect mydata
docker volume rm mydata
docker volume prune
docker volume prune -f

# === RUN WITH VOLUMES ===
# Named volume
docker run -v mydata:/container/path myapp
docker run --mount type=volume,source=mydata,target=/container/path myapp

# Bind mount
docker run -v /host/path:/container/path myapp
docker run -v /host/path:/container/path:ro myapp
docker run --mount type=bind,source=/host/path,target=/container/path myapp
docker run --mount type=bind,source=/host/path,target=/container/path,readonly myapp

# tmpfs
docker run --tmpfs /tmp myapp
docker run --mount type=tmpfs,target=/tmp,tmpfs-size=100m myapp

# Read-only filesystem + tmpfs writeable areas
docker run --read-only --tmpfs /tmp --tmpfs /var/run myapp

# === INSPECT MOUNTS ===
docker inspect web --format '{{json .Mounts}}'
docker inspect web | grep -A 20 '"Mounts"'

# === BACKUP PATTERN ===
docker run --rm \
  -v mydata:/source:ro \
  -v $(pwd)/backup:/backup \
  ubuntu tar czf /backup/mydata.tar.gz -C /source .
```

---

## 14. Quick Revision

| Question | Answer |
|---|---|
| What are the 3 Docker mount types? | Named Volume, Bind Mount, tmpfs |
| Where do named volumes live on the host? | `/var/lib/docker/volumes/<name>/_data` |
| What happens to tmpfs data when container stops? | Gone — it lives in RAM only |
| Named volume vs bind mount — key difference | Volume: Docker manages path; Bind: you control the host path |
| Which mount type is recommended for production databases? | Named Volume |
| What does `-v` do if the host path doesn't exist (bind)? | Creates it silently as a directory |
| What does `--mount type=bind` do if host path doesn't exist? | Errors immediately |
| How do you make a mount read-only? | `:ro` with `-v`, or `readonly` with `--mount` |
| How do you share a volume between two containers? | Both containers reference the same named volume name |
| How do you find the actual host path of a named volume? | `docker volume inspect <name>` → `Mountpoint` |
| What is an anonymous volume? | Volume with no name — random ID, hard to manage |
| How do you backup a Docker volume? | Mount it into a helper container + tar to bind-mounted backup dir |
| What Compose key declares top-level volumes? | `volumes:` at the root of the compose file |
| How do you completely wipe a container + its anonymous volumes? | `docker rm -v <container>` |

---

*Phase 2 complete. Next:* `Dockerfile` → building your own images layer by layer.
