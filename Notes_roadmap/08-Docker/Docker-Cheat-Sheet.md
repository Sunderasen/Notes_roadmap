# Docker Cheat Sheet

**Tags:** #docker #cheatsheet #reference  
**Vault:** `SRE-2027/08-Docker/Docker-Cheat-Sheet.md`

---

## Images

```bash
docker pull image:tag                        # download image
docker images                                # list local images
docker rmi image:tag                         # remove image
docker image prune -a                        # remove all unused
docker history image:tag                     # layer sizes
docker inspect image:tag                     # full metadata
docker tag image:tag registry/image:tag      # tag for push
docker push registry/image:tag              # push to registry
docker save image -o file.tar               # export to file
docker load -i file.tar                     # import from file
```

---

## Containers — Run

```bash
docker run -d --name web -p 8080:80 nginx:alpine      # basic
docker run -it --rm alpine sh                          # interactive, auto-delete
docker run -d -e KEY=val --env-file .env myapp        # env vars
docker run -d -v mydata:/app/data myapp               # named volume
docker run -d --memory=256m --cpus=0.5 myapp          # resource limits
docker run -d --restart=always myapp                  # auto-restart
docker run -d --network mynet myapp                   # custom network
docker run -d --read-only --tmpfs /tmp myapp          # secure run
```

---

## Containers — Inspect

```bash
docker ps                                    # running
docker ps -a                                 # all states
docker logs -f web                           # follow logs
docker logs --tail 100 --since 10m web      # scoped logs
docker stats                                 # live resource usage
docker stats --no-stream                     # snapshot
docker inspect web                           # full config
docker inspect web --format '{{.State.ExitCode}}'
docker inspect web --format '{{json .Config.Env}}'
docker inspect web --format '{{.NetworkSettings.IPAddress}}'
docker top web                               # processes inside
docker port web                              # port mappings
docker diff web                              # filesystem changes
```

---

## Containers — Manage

```bash
docker stop web                              # SIGTERM → SIGKILL after 10s
docker kill web                              # SIGKILL immediately
docker start web                             # start stopped container
docker restart web                           # stop + start
docker rm web                                # remove stopped
docker rm -f web                             # force remove
docker container prune                       # remove all stopped
docker exec -it web sh                       # shell inside
docker cp web:/app/log.txt ./               # copy file out
```

---

## Network

```bash
docker network ls
docker network create mynet
docker network inspect mynet
docker network connect mynet web
docker network disconnect mynet web
docker network rm mynet
docker network prune
```

---

## Volumes

```bash
docker volume ls
docker volume create mydata
docker volume inspect mydata
docker volume rm mydata
docker volume prune
```

---

## Build

```bash
docker build -t myapp:v1 .
docker build --no-cache -t myapp:v1 .
docker build -f Dockerfile.prod -t myapp:v1 .
docker build --target builder -t myapp:v1 .
docker build --build-arg VERSION=1.0 -t myapp:v1 .
```

---

## Compose

```bash
docker compose up -d                         # start all
docker compose down                          # stop (keep volumes)
docker compose down -v                       # stop + delete volumes
docker compose ps                            # service status
docker compose logs -f                       # all logs
docker compose logs -f app                   # one service
docker compose exec app sh                   # shell into service
docker compose build                         # rebuild images
docker compose restart app                   # restart one
docker compose pull                          # pull latest images
docker compose config                        # validate compose file
```

---

## System

```bash
docker info                                  # daemon info
docker version                               # client + daemon version
docker system df                             # disk usage
docker system prune                          # clean unused resources
docker system prune -a --volumes             # full clean (CAREFUL)
docker events --since 1h                     # daemon event log
docker events --filter event=die             # container deaths only
```

---

## Incident Cheatsheet

```bash
# Container down?
docker ps -a                                 # check state + exit code
docker logs --tail 200 web                   # what did it say?

# Container slow / crashing?
docker stats web                             # OOMing? CPU throttled?
docker inspect web --format '{{.State.OOMKilled}}'

# Can't connect to container?
docker port web                              # port mapping correct?
docker network inspect mynet                 # on same network?
docker exec -it web ping db                  # can it reach dependency?

# Disk full?
docker system df                             # what's eating space?
docker system prune -a                       # clean it

# Need to debug inside?
docker exec -it web sh                       # get a shell
docker run -it --rm --entrypoint sh myapp   # override entrypoint
```

---

## Exit Codes

| Code | Meaning |
|---|---|
| `0` | Clean exit |
| `1` | App error |
| `125` | Docker command failed |
| `127` | Command not found in container |
| `137` | OOM killed or `docker kill` |
| `143` | Graceful stop via `docker stop` |
