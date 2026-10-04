# 08 — Environment Variables

**Tags:** #docker #env #config #secrets  
**Vault:** `SRE-2027/08-Docker/08-Environment-Variables.md`

---

## Concept

Environment variables are the standard way to pass configuration into containers at runtime — ports, DB URLs, feature flags, log levels. They keep config separate from code and image, following the 12-factor app principle.

---

## Key Points

- `-e KEY=VALUE` → single variable at `docker run`
- `--env-file .env` → load multiple vars from a file
- `ENV` in Dockerfile → baked into image; visible in `docker inspect`; cannot be removed
- `ARG` in Dockerfile → build time only; gone at runtime
- **Never put secrets in `ENV` in Dockerfile** — they're in image history forever
- Variables set at `docker run` override `ENV` from Dockerfile
- `docker inspect <name>` → shows all env vars on a running container
- In Compose: use `environment:` for inline or `env_file:` for file-based
- `.env` file next to `docker-compose.yml` is auto-loaded for variable substitution in compose file itself (not container env)

---

## Diagram

```
Priority (highest → lowest):
─────────────────────────────────────────
docker run -e KEY=val          ← wins
  └── overrides
      docker-compose environment:
        └── overrides
            Dockerfile ENV KEY=val   ← lowest
```

---

## Example

```bash
# Single variable
docker run -d -e NODE_ENV=production -e PORT=3000 myapp:v1

# From file
docker run -d --env-file .env myapp:v1

# .env file format
NODE_ENV=production
PORT=3000
DB_URL=postgres://user:pass@db:5432/mydb

# See all env vars on running container
docker inspect myapp --format '{{json .Config.Env}}'

# Or from inside the container
docker exec myapp env
```

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:v1
    environment:
      - NODE_ENV=production
      - PORT=3000
    env_file:
      - .env.production
```

---

## SRE Relevance

Misconfigured env vars are a top cause of deployment failures — wrong DB URL, missing API keys, wrong port. Always `docker inspect` or `docker exec env` to verify what a container actually sees.

---

## Quick Revision

| Question | Answer |
|---|---|
| How to pass a single env var at runtime? | `docker run -e KEY=VALUE` |
| How to load env vars from a file? | `docker run --env-file .env` |
| `ENV` vs `ARG` in Dockerfile? | `ENV` = runtime + build; `ARG` = build only |
| Why not put secrets in Dockerfile `ENV`? | Baked into image history — visible forever |
| How to see what env vars a container has? | `docker inspect <name>` or `docker exec <name> env` |
| Which env var wins — Dockerfile or `docker run`? | `docker run -e` always wins |
| How does Compose auto-load `.env`? | For variable substitution in compose file — NOT auto-injected into containers |
