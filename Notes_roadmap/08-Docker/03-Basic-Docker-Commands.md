# Basic Docker Commands

**Tags:** #docker #commands #cli #sre #cheatsheet  
**Vault Path:** `SRE-2027/08-Docker/03-Basic-Docker-Commands.md`

---

## 1. Concept

Docker commands are the day-to-day interface between you and the Docker daemon. These are the commands you will use during deployments, debugging, incident response, and cleanup. This note covers commands grouped by what they do — not alphabetically — so you can reach for the right one in the right situation.

---

## 2. Command Groups

---

### 📦 IMAGE COMMANDS

Images are read-only blueprints. You pull them, build them, tag them, push them, and inspect them.

```bash
# Pull an image from registry
docker pull nginx                       # latest tag
docker pull nginx:1.25                  # specific version
docker pull registry.io/myapp:v2        # private registry

# List local images
docker images
docker images nginx                     # filter by name
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Remove images
docker rmi nginx:1.25                   # remove one
docker rmi nginx:1.25 myapp:v1         # remove multiple
docker rmi $(docker images -q)          # remove all (careful)
docker image prune                      # remove dangling images only
docker image prune -a                   # remove all unused images

# Tag an image
docker tag myapp:v2 registry.io/myapp:v2
docker tag myapp:v2 myapp:latest

# Push to registry
docker login registry.io
docker push registry.io/myapp:v2

# Inspect an image
docker inspect nginx:1.25              # full JSON metadata
docker history nginx:1.25             # layer breakdown with sizes

# Save / load images (offline transfer)
docker save myapp:v2 -o myapp-v2.tar  # export to tar
docker load -i myapp-v2.tar           # import from tar

# Search DockerHub
docker search nginx
docker search nginx --filter is-official=true
```

**SRE notes:**
- Always pin image versions (`:1.25` not `:latest`) in production — `:latest` changes under you
- `docker history` is your first tool when an image is unexpectedly large
- `docker save` / `docker load` = image transfer in air-gapped environments

---

### 🚀 CONTAINER RUN COMMANDS

```bash
# Basic run
docker run nginx                        # foreground, blocking
docker run -d nginx                     # detached (background)
docker run -d --name web nginx          # with a name

# Port mapping
docker run -d -p 8080:80 nginx          # host:container
docker run -d -p 8080:80 -p 443:443 nginx  # multiple ports
docker run -d -p 127.0.0.1:8080:80 nginx   # bind to specific interface

# Environment variables
docker run -d -e APP_ENV=prod myapp
docker run -d --env-file .env myapp

# Detached + interactive
docker run -it ubuntu bash              # interactive shell
docker run -it --rm ubuntu bash         # delete on exit

# Override entrypoint / command
docker run --entrypoint bash myapp
docker run myapp echo "hello"

# Resource limits
docker run -d --memory="256m" --cpus="0.5" myapp

# Restart policy
docker run -d --restart=always myapp
docker run -d --restart=on-failure:3 myapp

# Run as specific user
docker run -d --user 1000:1000 myapp

# Read-only filesystem (security)
docker run -d --read-only --tmpfs /tmp myapp
```

---

### 🔍 CONTAINER INSPECTION COMMANDS

Your primary tools during incidents.

```bash
# List containers
docker ps                               # running only
docker ps -a                            # all states
docker ps -q                            # IDs only
docker ps -a --filter name=web          # filter by name
docker ps -a --filter status=exited     # only stopped ones
docker ps -a --filter exited=137        # OOM killed containers
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Logs
docker logs web                         # all logs
docker logs -f web                      # follow (like tail -f)
docker logs --tail 100 web              # last 100 lines
docker logs --since 10m web             # last 10 minutes
docker logs --since "2025-09-01" web    # since timestamp
docker logs --until "2025-09-01T12:00" web
docker logs -f --tail 50 web            # follow + last 50 lines

# Inspect
docker inspect web                      # full JSON config
docker inspect web --format '{{.State.Status}}'      # state only
docker inspect web --format '{{.State.ExitCode}}'    # exit code
docker inspect web --format '{{.NetworkSettings.IPAddress}}'  # IP
docker inspect web --format '{{json .Mounts}}'       # mounts

# Processes inside container
docker top web                          # running processes
docker top web aux                      # with ps options

# Resource stats
docker stats                            # all containers, live
docker stats web                        # single container, live
docker stats --no-stream                # snapshot (no live update)
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# Port mappings
docker port web                         # all port mappings
docker port web 80                      # specific container port

# Filesystem changes
docker diff web                         # A=added, C=changed, D=deleted
```

**SRE incident order:**
```
docker ps           → is it running?
docker ps -a        → did it exit? what code?
docker logs -f web  → what did it say?
docker stats web    → is it resource constrained?
docker inspect web  → what is its full config?
```

---

### 💻 EXEC & DEBUG COMMANDS

Getting inside a container. This is your SSH equivalent.

```bash
# Open a shell in running container
docker exec -it web bash                # bash (most images)
docker exec -it web sh                  # sh (minimal/Alpine images)
docker exec -it web /bin/sh             # explicit path

# Run a single command
docker exec web cat /etc/hosts
docker exec web env                     # show env variables
docker exec web ls -la /app
docker exec web ps aux                  # processes inside

# Run as specific user
docker exec -it --user root web bash    # escalate to root
docker exec -it --user 1000 web bash    # run as specific UID

# Copy files
docker cp web:/app/logs/error.log ./          # container → host
docker cp ./config.json web:/app/config.json  # host → container
docker cp web:/etc/nginx/nginx.conf ./        # grab config file

# Attach to container's main process
docker attach web                       # careful: Ctrl+C kills PID 1
```

**Key tips:**
- `exec -it` = interactive terminal. Always use both `-i` and `-t` together for shell sessions
- If `bash` fails → try `sh`. Alpine Linux doesn't include bash
- `docker attach` attaches to PID 1 — Ctrl+C will kill the container. Use `Ctrl+P, Ctrl+Q` to detach safely
- `docker cp` is your friend during incidents when logs aren't centrally collected

---

### 🔄 CONTAINER LIFECYCLE COMMANDS

```bash
# Stop / kill
docker stop web                         # SIGTERM → wait 10s → SIGKILL
docker stop -t 30 web                   # 30s grace period
docker kill web                         # immediate SIGKILL
docker kill --signal SIGHUP web         # send custom signal

# Start / restart
docker start web                        # start stopped container
docker restart web                      # stop + start
docker restart -t 5 web                 # 5s grace on stop

# Pause / resume
docker pause web
docker unpause web

# Remove
docker rm web                           # remove stopped container
docker rm -f web                        # force remove (even if running)
docker rm $(docker ps -a -q)            # remove all stopped containers
docker container prune                  # same, cleaner
docker container prune --filter until=24h  # older than 24h only
```

---

### 🌐 NETWORK COMMANDS

```bash
# List networks
docker network ls

# Create networks
docker network create mynet                         # bridge (default)
docker network create --driver bridge mynet
docker network create --subnet 172.20.0.0/16 mynet  # custom subnet

# Inspect
docker network inspect mynet
docker network inspect mynet --format '{{json .Containers}}'

# Connect / disconnect
docker network connect mynet web            # attach running container
docker network disconnect mynet web         # detach

# Remove
docker network rm mynet
docker network prune                        # remove all unused networks
```

---

### 💾 VOLUME COMMANDS

```bash
# List
docker volume ls
docker volume ls --filter dangling=true     # only dangling volumes

# Create
docker volume create mydata

# Inspect
docker volume inspect mydata               # find host path, driver

# Remove
docker volume rm mydata
docker volume prune                        # remove all unused volumes

# Mount in run
docker run -v mydata:/app/data myapp                     # named
docker run -v /host/path:/app/data myapp                 # bind
docker run --mount type=volume,source=mydata,target=/app/data myapp
```

---

### 🧹 CLEANUP COMMANDS

Critical for keeping hosts healthy. Disk fills up fast on CI/build servers.

```bash
# See what's eating disk
docker system df                          # summary
docker system df -v                       # verbose (per image/container)

# Prune specific resources
docker image prune                        # dangling images only
docker image prune -a                     # all unused images
docker container prune                    # stopped containers
docker volume prune                       # unused volumes
docker network prune                      # unused networks
docker builder prune                      # build cache

# Nuclear option — clean everything unused
docker system prune                       # containers + networks + dangling images
docker system prune -a                    # + all unused images
docker system prune -a --volumes          # + volumes (DANGEROUS on DB hosts)

# Force without confirmation prompt
docker system prune -f
docker system prune -a -f
```

**Safety rules:**
- Always run `docker system df` before `prune` — know what you're removing
- Never run `prune --volumes` on a host without first checking `docker volume ls`
- On CI hosts: `docker system prune -a -f` is a standard post-job cleanup step

---

### 📊 SYSTEM COMMANDS

```bash
# Info and version
docker info                               # daemon info, storage driver, OS, resources
docker version                            # client + daemon versions

# Events (real-time audit log)
docker events                             # stream live events
docker events --since 1h                  # last 1 hour
docker events --filter type=container     # container events only
docker events --filter event=die          # only container deaths
docker events --filter container=web      # events for specific container

# Registry auth
docker login                              # DockerHub
docker login registry.io                  # private registry
docker login -u user -p pass registry.io  # non-interactive
docker logout registry.io

# Build (covered in Dockerfile note)
docker build -t myapp:v2 .
docker build -t myapp:v2 -f Dockerfile.prod .
```

---

## 3. SRE Relevance — Command Patterns

### Incident — Container Down

```bash
docker ps -a --filter name=myapp        # find it, check status
docker logs --tail 200 myapp            # what happened?
docker inspect myapp --format '{{.State.ExitCode}}'  # exit code
docker events --filter container=myapp --since 1h    # what events?
```

### Incident — High CPU / Memory

```bash
docker stats --no-stream                # snapshot of all containers
docker stats myapp                      # watch specific container
docker exec -it myapp top               # processes inside
docker exec -it myapp sh -c "cat /proc/meminfo"
```

### Incident — Network / Connectivity

```bash
docker network inspect mynet            # who's on the network?
docker inspect myapp --format '{{.NetworkSettings.Networks}}'
docker exec -it myapp ping db           # can it reach other containers?
docker exec -it myapp nslookup db       # DNS resolution
docker exec -it myapp curl http://api:8080/health
```

### Disk Full on Host

```bash
docker system df                        # what's eating space?
docker system prune -a -f               # clean unused
docker logs --since 1h daemon | grep "no space"  # confirm cause
```

### Config Debugging

```bash
docker inspect myapp                    # full config
docker inspect myapp --format '{{json .Config.Env}}'    # env vars
docker inspect myapp --format '{{json .HostConfig}}'    # resource limits
docker inspect myapp --format '{{json .Mounts}}'        # volumes
```

---

## 4. Quick Reference — All Commands One Page

```bash
# IMAGES
docker pull IMAGE[:TAG]
docker images
docker rmi IMAGE
docker inspect IMAGE
docker history IMAGE
docker tag IMAGE NEW_TAG
docker push IMAGE
docker save IMAGE -o file.tar
docker load -i file.tar
docker image prune [-a]

# RUN
docker run [-d] [-it] [--rm] [--name] [-p] [-e] [-v] [--restart] IMAGE

# CONTAINERS
docker ps [-a] [-q] [--filter] [--format]
docker logs [-f] [--tail N] [--since] CONTAINER
docker inspect CONTAINER [--format]
docker stats [CONTAINER] [--no-stream]
docker top CONTAINER
docker port CONTAINER
docker diff CONTAINER

# EXEC / COPY
docker exec -it CONTAINER bash|sh
docker exec CONTAINER COMMAND
docker cp CONTAINER:PATH HOST_PATH
docker cp HOST_PATH CONTAINER:PATH
docker attach CONTAINER

# LIFECYCLE
docker start CONTAINER
docker stop [-t N] CONTAINER
docker kill [--signal] CONTAINER
docker restart CONTAINER
docker pause CONTAINER
docker unpause CONTAINER
docker rm [-f] CONTAINER
docker container prune

# NETWORK
docker network ls
docker network create NAME
docker network inspect NAME
docker network connect NETWORK CONTAINER
docker network disconnect NETWORK CONTAINER
docker network rm NAME
docker network prune

# VOLUMES
docker volume ls
docker volume create NAME
docker volume inspect NAME
docker volume rm NAME
docker volume prune

# SYSTEM
docker info
docker version
docker system df [-v]
docker system prune [-a] [--volumes] [-f]
docker events [--since] [--filter]
docker login [REGISTRY]
docker logout [REGISTRY]
```

---

## 5. Quick Revision

| Question | Answer |
|---|---|
| How do you see all containers including stopped? | `docker ps -a` |
| How do you follow container logs in real time? | `docker logs -f <name>` |
| How do you get a shell in a running container? | `docker exec -it <name> bash` |
| How do you see CPU/memory live for all containers? | `docker stats` |
| How do you copy a file out of a container? | `docker cp <name>:/path/file ./` |
| How do you see what ports a container exposes? | `docker port <name>` |
| How do you remove all stopped containers? | `docker container prune` |
| How do you see Docker disk usage? | `docker system df` |
| How do you clean all unused Docker resources? | `docker system prune -a` |
| How do you find the IP of a container? | `docker inspect <name> --format '{{.NetworkSettings.IPAddress}}'` |
| How do you see env vars set on a container? | `docker inspect <name> --format '{{json .Config.Env}}'` |
| How do you watch real-time Docker daemon events? | `docker events` |
| How do you run a one-off debug container that deletes itself? | `docker run -it --rm ubuntu bash` |
| How do you check why a container keeps crashing? | `docker ps -a` (exit code) + `docker logs` |

---

*Next note:* `04-Docker-Networking.md`
