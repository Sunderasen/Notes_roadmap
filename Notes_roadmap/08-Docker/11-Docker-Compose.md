# 11 — Docker Compose

**Tags:** #docker #compose #multicontainer #yaml  
**Vault:** `SRE-2027/08-Docker/11-Docker-Compose.md`

---

## Concept

Docker Compose defines and runs multi-container applications from a single YAML file. One command (`docker compose up`) starts your entire stack — app, database, cache, reverse proxy — with correct networking and volumes.

---

## Key Points

- Compose auto-creates a **custom bridge network** — all services reach each other by service name
- `depends_on` → controls start order (not readiness — app may still start before DB is ready)
- `volumes:` at top level → declares named volumes shared between services
- `env_file:` → loads vars from file into container
- `docker compose up -d` → starts all services detached
- `docker compose down` → stops + removes containers + networks (NOT volumes)
- `docker compose down -v` → also removes volumes (DESTRUCTIVE — data gone)
- `docker compose logs -f` → follow all service logs at once
- `docker compose exec <service> sh` → shell into a running service
- `docker compose build` → rebuild images before starting
- Each service = one container (by default)

---

## Diagram

```
docker-compose.yml
─────────────────────────────────────────────
services:
  app ──────────────────────────┐
  db  ──────────────────────────┤  All on same
  cache ────────────────────────┤  auto-created
                                │  bridge network
networks:                       │
  default (auto) ───────────────┘

  app → reaches db    by service name "db"
  app → reaches cache by service name "cache"
  No manual network setup needed
```

---

## Example

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DB_URL=postgres://user:pass@db:5432/mydb
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  pgdata:
```

```bash
docker compose up -d              # start all
docker compose ps                 # status
docker compose logs -f app        # follow app logs
docker compose exec app sh        # shell into app
docker compose restart app        # restart one service
docker compose down               # stop (keep volumes)
docker compose down -v            # stop + delete volumes
```

---

## SRE Relevance

Compose is the standard for local dev and staging environments. Understanding it is a prerequisite for Kubernetes — same concepts (services, volumes, networking) just a different API.

---

## Quick Revision

| Question | Answer |
|---|---|
| What network does Compose create automatically? | Custom bridge network named `<project>_default` |
| How do services reach each other? | By service name — Docker DNS resolves it |
| Does `depends_on` wait for readiness? | No — only waits for container to start, not app readiness |
| `docker compose down` vs `down -v`? | `down` keeps volumes; `down -v` deletes them |
| How to follow logs of all services? | `docker compose logs -f` |
| How to shell into a service? | `docker compose exec <service> sh` |
| Where are named volumes declared? | Top-level `volumes:` key in compose file |
| How to rebuild and restart? | `docker compose build && docker compose up -d` |
