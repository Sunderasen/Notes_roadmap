# 02 — Docker Run

**Tags:** #docker #run #containers  
**Vault:** `SRE-2027/08-Docker/02-Docker-Run.md`

---

## Concept

`docker run` creates and starts a container from an image in a single command. It is shorthand for `docker create` + `docker start`. Every flag controls one aspect of how the container runs.

---

## Key Points

- `-d` → detached (background); without it, container blocks your terminal
- `-p host:container` → publish port; without it, container is unreachable from outside
- `-e KEY=VAL` → inject environment variable
- `-v source:target` → mount volume or bind mount
- `--name` → give container a name; else Docker generates a random one
- `--rm` → auto-delete container when it exits (good for one-off tasks)
- `--restart=always` → auto-restart on failure or daemon restart
- `--memory` / `--cpus` → resource limits
- `--network` → attach to a specific network
- Without `--network`, container joins the default bridge (no DNS)

---

## Diagram

```
docker run -d --name web -p 8080:80 --restart=always nginx:alpine
     │         │            │                │               │
  detached   name      host:container    policy           image
  background           port mapping     always restart
```

---

## Example

```bash
# Basic run — detached, named, port mapped
docker run -d --name api -p 5050:5050 node:22-alpine

# One-off debug container — deleted on exit
docker run -it --rm alpine sh

# With env + volume + restart
docker run -d \
  --name app \
  -p 3000:3000 \
  -e NODE_ENV=production \
  -v mydata:/app/data \
  --restart=on-failure:3 \
  myapp:v1
```

---

## SRE Relevance

`docker run` flags are what you reach for during incident response — quickly spinning up a debug container, testing connectivity, or restarting a service with corrected env vars.

---

## Quick Revision

| Question | Answer |
|---|---|
| What does `docker run` actually do? | `docker create` + `docker start` |
| Flag for background (detached) mode? | `-d` |
| Flag to delete container after exit? | `--rm` |
| Flag to map host port 8080 to container port 80? | `-p 8080:80` |
| Flag to set env variable? | `-e KEY=VALUE` |
| Flag to always restart container? | `--restart=always` |
| How to get an interactive shell? | `-it` with `sh` or `bash` |
| Default network if none specified? | Default bridge (no DNS between containers) |
