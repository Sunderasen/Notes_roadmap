# 04 — Dockerfile

**Tags:** #docker #dockerfile #build #layers  
**Vault:** `SRE-2027/08-Docker/04-Dockerfile.md`

---

## Concept

A Dockerfile is a plain text script of instructions that builds a Docker image layer by layer. Each instruction creates a cached, reusable layer. Order of instructions directly affects build speed and image size.

---

## Key Points

- `FROM` — base image; always pin version; use Alpine for smallest size
- `RUN` — runs command at build time; each `RUN` = one layer; chain with `&&`
- `COPY` — copies files from build context into image; prefer over `ADD`
- `WORKDIR` — sets working directory; creates it if missing
- `ENV` — sets env vars baked into image; visible in `docker inspect`
- `ARG` — build-time only variable; gone at runtime
- `EXPOSE` — documents port; does NOT publish it
- `CMD` — default arguments; overridable at `docker run`
- `ENTRYPOINT` — fixed executable; use exec form `["cmd"]` always
- `USER` — sets runtime user; never run as root in production
- `HEALTHCHECK` — tells Docker how to check if app is healthy
- Layer cache invalidates from the changed line downward — put stable things first
- Use `.dockerignore` to keep build context small

---

## Diagram

```
Dockerfile                         Image Layers
──────────────────                 ──────────────────────
FROM alpine:3.19        →          Layer 1: base OS
RUN apk add curl        →          Layer 2: curl installed
COPY src/ /app          →          Layer 3: source copied    ← changes often
RUN chmod +x /app/run   →          Layer 4: permission set
CMD ["./run"]           →          (metadata, no layer)

Cache rule: change Layer 3 → Layers 4+ re-run. 1 and 2 stay cached.
```

---

## Example

```dockerfile
FROM alpine:3.19

RUN addgroup -S app && adduser -S app -G app && \
    apk add --no-cache nodejs npm

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY --chown=app:app . .

USER app
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s \
  CMD wget -qO- http://localhost:3000/health || exit 1

ENTRYPOINT ["node"]
CMD ["index.js"]
```

---

## SRE Relevance

Well-structured Dockerfiles mean faster CI builds (cache hits), smaller attack surface (Alpine, non-root), and reliable deployments (pinned versions, healthchecks).

---

## Quick Revision

| Question | Answer |
|---|---|
| What does each `RUN` create? | A new image layer |
| Why chain commands with `&&`? | File deletions in same layer actually reduce size |
| `COPY` vs `ADD`? | Prefer `COPY`; `ADD` auto-extracts tars |
| `CMD` vs `ENTRYPOINT`? | `CMD` is overridable; `ENTRYPOINT` is fixed |
| Why exec form `["node"]` over shell form `node`? | App becomes PID 1, receives signals correctly |
| `ARG` vs `ENV`? | `ARG` = build time only; `ENV` = build + runtime |
| What does `EXPOSE` do? | Documents port only — does not publish it |
| Why create a non-root user? | Root in container = potential root on host if escaped |
