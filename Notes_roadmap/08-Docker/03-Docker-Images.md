# 03 — Docker Images

**Tags:** #docker #images #layers #registry  
**Vault:** `SRE-2027/08-Docker/03-Docker-Images.md`

---

## Concept

A Docker image is a read-only, layered blueprint for a container. Each instruction in a Dockerfile adds a layer. Layers are cached and shared across images — pulling an image only downloads layers you don't already have.

---

## Key Points

- Images are **immutable** — never modified, only new layers added on top
- Layers are **content-addressed** — same layer shared across multiple images
- Container adds a thin **writable layer** on top of image layers at runtime
- Storage driver: **overlay2** (default on Linux) — stacks layers efficiently
- Images live in `/var/lib/docker/overlay2/` on the host
- Always **pin versions** — `alpine:3.19` not `alpine:latest`
- `:latest` is just a tag — it changes, it breaks builds silently
- `docker pull` downloads missing layers only (delta pulls)

---

## Diagram

```
Image: myapp:v1 (read-only layers)
────────────────────────────────────
  Layer 4: COPY src/ /app          ← your code
  Layer 3: RUN npm ci              ← dependencies
  Layer 2: RUN apk add curl        ← packages
  Layer 1: FROM alpine:3.19        ← base OS (~7MB)
────────────────────────────────────
  + Writable layer                 ← added at container runtime
    (dies with container)
```

---

## Example

```bash
# Pull specific version
docker pull alpine:3.19

# List local images
docker images

# See layer breakdown + sizes
docker history alpine:3.19

# Inspect full metadata
docker inspect alpine:3.19

# Remove image
docker rmi alpine:3.19

# Remove all unused images
docker image prune -a
```

---

## SRE Relevance

Image hygiene directly affects CI speed, disk usage, and security posture. Bloated images slow deployments; unpinned tags cause silent production breakage.

---

## Quick Revision

| Question | Answer |
|---|---|
| What is a Docker image? | Read-only layered blueprint for a container |
| Where are images stored on disk? | `/var/lib/docker/overlay2/` |
| What storage driver does Docker use by default? | overlay2 |
| What happens when you `docker run` an image? | A writable layer is added on top |
| Why pin image versions? | `:latest` changes silently and can break builds |
| How to see image layer sizes? | `docker history <image>` |
| How to remove all unused images? | `docker image prune -a` |
| What does `docker pull` skip? | Layers already present locally |
