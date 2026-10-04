# 07 — Docker Volumes

**Tags:** #docker #volumes #storage #persistence  
**Vault:** `SRE-2027/08-Docker/07-Docker-Volumes.md`

---

## Concept

Container filesystems are ephemeral — data dies with the container. Volumes provide storage that lives outside the container lifecycle. Three types: named volumes (Docker-managed), bind mounts (host path), tmpfs (RAM only).

---

## Key Points

- **Named volume** — Docker manages the path; survives `docker rm`; visible in `docker volume ls`
- **Bind mount** — you specify host path; tied to host filesystem; good for dev/config injection
- **tmpfs** — RAM only; never touches disk; gone on container stop; for sensitive temp data
- Named volumes live at `/var/lib/docker/volumes/<name>/_data`
- `-v` syntax: short, creates host path silently if missing
- `--mount` syntax: explicit, errors if host path missing → safer in scripts
- `docker rm` does NOT delete named volumes — must `docker volume rm` explicitly
- Use `:ro` to make any mount read-only

---

## Diagram

```
                    DOCKER HOST
┌────────────────────────────────────────────────┐
│                                                │
│  /var/lib/docker/volumes/mydata/  (named vol)  │
│  /host/config/app.conf            (bind mount) │
│  RAM                              (tmpfs)       │
│                                                │
└───────────┬────────────────────────────────────┘
            │ mounted into
            ▼
      CONTAINER /app/data
               /app/config  (read-only)
               /tmp/cache   (RAM, ephemeral)
```

---

## Example

```bash
# Named volume — survives container removal
docker run -d \
  --name db \
  -v mydata:/var/lib/postgresql/data \
  postgres:16-alpine

# Bind mount — inject config read-only
docker run -d \
  --name app \
  -v $(pwd)/config.yaml:/app/config.yaml:ro \
  myapp:v1

# tmpfs — sensitive scratch space
docker run -d \
  --name secure \
  --mount type=tmpfs,target=/tmp/tokens,tmpfs-size=50m \
  myapp:v1

# Volume management
docker volume ls
docker volume inspect mydata     # find host path
docker volume rm mydata
docker volume prune              # remove all unused
```

---

## SRE Relevance

Named volumes are how you keep database data alive across container replacements. Always `docker volume inspect` before pruning — data loss from accidental volume removal is unrecoverable.

---

## Quick Revision

| Question | Answer |
|---|---|
| Three mount types? | Named volume, bind mount, tmpfs |
| Where do named volumes live? | `/var/lib/docker/volumes/<name>/_data` |
| Does `docker rm` delete named volumes? | No — must `docker volume rm` explicitly |
| `-v` vs `--mount` if host path missing? | `-v` creates silently; `--mount` errors immediately |
| tmpfs data after container stops? | Gone — RAM only |
| How to make a mount read-only? | `:ro` with `-v` or `readonly` with `--mount` |
| How to find actual host path of a volume? | `docker volume inspect <name>` |
| Best for production databases? | Named volume |
