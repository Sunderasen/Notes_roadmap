# Container Lifecycle

**Tags:** #docker #containers #lifecycle #states #sre  
**Vault Path:** `SRE-2027/08-Docker/02-Container-Lifecycle.md`

---

## 1. Concept

A container is not just "running" or "stopped." It moves through a defined set of **states** from creation to deletion. Understanding the full lifecycle tells you exactly what is happening when a container behaves unexpectedly — why it restarted, why it exited with code 137, or why it shows as `Exited (1)` in `docker ps -a`.

---

## 2. Definition — The State Machine

```
                        docker create
                             │
                             ▼
                       ┌──────────┐
                       │ CREATED  │  ← exists, not started yet
                       └────┬─────┘
                            │ docker start
                            ▼
          ┌─────────────────────────────────┐
          │                                 │
          │           RUNNING               │◀──── docker restart
          │                                 │
          └──┬────────────┬────────────┬────┘
             │            │            │
        docker pause  docker stop  docker kill
             │            │            │
             ▼            ▼            ▼
        ┌─────────┐  ┌─────────┐  ┌─────────┐
        │ PAUSED  │  │ STOPPED │  │ STOPPED │
        └────┬────┘  │(Exited) │  │(Exited) │
             │       └────┬────┘  └────┬────┘
       docker unpause     │            │
             │            └─────┬──────┘
             ▼                  │
          RUNNING          docker start
                                │
                                ▼
                            RUNNING

          Any stopped state:
                       docker rm
                            │
                            ▼
                         DELETED  (gone from docker ps -a)
```

---

## 3. States — Definitions

### CREATED
Container has been created (`docker create`) but never started. The writable layer exists, config is set, but no process is running. Rarely used directly — `docker run` skips this and goes straight to running.

```bash
docker create --name web nginx      # creates but does NOT start
docker ps -a                        # shows as "Created"
docker start web                    # now it runs
```

### RUNNING
A container process (PID 1 inside the container) is active. The container is consuming resources and responding to signals.

```bash
docker run -d --name web nginx      # created + started
docker ps                           # shows "Up X minutes"
```

### PAUSED
All processes inside are **frozen** using `SIGSTOP` / cgroup freezer. Container is still in memory, still has its network, but nothing executes. Useful for snapshotting or temporary suspension.

```bash
docker pause web
docker ps        # shows "Up X minutes (Paused)"
docker unpause web
```

### STOPPED / EXITED
The container's main process (PID 1) has ended — either normally, due to an error, or from a signal. Container still exists on disk (`docker ps -a` shows it) but no process is running.

```bash
docker stop web     # sends SIGTERM, waits 10s, then SIGKILL
docker ps -a        # shows "Exited (0) 5 seconds ago"
```

Exit codes tell you **why** it stopped:

| Exit Code | Meaning |
|---|---|
| `0` | Clean exit — process finished normally |
| `1` | General error — application crashed or config issue |
| `2` | Shell built-in misuse |
| `125` | Docker itself failed (invalid flag, permission issue) |
| `126` | Command found but not executable |
| `127` | Command not found inside container |
| `130` | Killed by Ctrl+C (SIGINT) |
| `137` | Killed by SIGKILL (OOM killer or `docker kill`) |
| `139` | Segmentation fault |
| `143` | Killed by SIGTERM (graceful stop via `docker stop`) |

**SRE shortcut:**
- Exit `0` = intended stop
- Exit `1` = app error, check logs
- Exit `137` = OOM killed or force killed — check memory limits
- Exit `143` = gracefully stopped

### DEAD
A rare state where Docker tried to remove the container but failed. Usually a kernel or storage issue. Requires manual cleanup.

```bash
docker ps -a    # shows "Dead"
docker rm -f <container>   # force remove
```

### DELETED
Container is fully gone — no longer in `docker ps -a`. The writable layer is deleted. Named volumes survive; anonymous volumes may be deleted depending on flags.

```bash
docker rm web              # remove stopped container
docker rm -f web           # force remove running container
docker run --rm nginx      # auto-delete when stopped
```

---

## 4. Creating a Container — `docker run` Deep Dive

`docker run` = `docker create` + `docker start` + optionally `docker attach`

```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARGS]
```

### Most Important Flags

```bash
# --- EXECUTION MODE ---
-d                        # detached — run in background
-it                       # interactive + pseudo-TTY — for shell access
--rm                      # delete container when it exits

# --- IDENTITY ---
--name web                # assign a name (else Docker generates one)

# --- NETWORKING ---
-p 8080:80                # map host port 8080 → container port 80
-p 127.0.0.1:8080:80      # bind to specific host interface
--network mynet           # attach to a custom network
--hostname app01          # set container hostname

# --- ENVIRONMENT ---
-e APP_ENV=production     # set single env variable
--env-file .env           # load env vars from file

# --- RESOURCES ---
--memory="512m"           # max memory (k, m, g)
--memory-swap="1g"        # total memory + swap
--cpus="1.5"              # CPU quota (1.5 = 1.5 cores)
--cpu-shares=512          # relative CPU weight (default 1024)

# --- RESTART POLICY ---
--restart=no              # never restart (default)
--restart=always          # always restart
--restart=on-failure      # restart only on non-zero exit
--restart=on-failure:3    # restart up to 3 times
--restart=unless-stopped  # restart always unless manually stopped

# --- STORAGE ---
-v mydata:/app/data       # named volume
-v /host/path:/app/path   # bind mount
--mount type=tmpfs,...    # tmpfs

# --- USER / SECURITY ---
--user 1000:1000          # run as specific UID:GID
--read-only               # read-only root filesystem
--cap-drop ALL            # drop all Linux capabilities
--cap-add NET_ADMIN       # add specific capability
```

### Examples

```bash
# Basic web server
docker run -d --name web -p 8080:80 nginx

# Interactive debugging container (deleted on exit)
docker run -it --rm ubuntu bash

# App with env file, resource limits, restart policy
docker run -d \
  --name api \
  --env-file .env \
  --memory="256m" \
  --cpus="0.5" \
  --restart=on-failure:3 \
  -p 3000:3000 \
  myapi:v2

# Read-only container (security hardening)
docker run -d \
  --name secure-app \
  --read-only \
  --tmpfs /tmp \
  myapp:v2
```

---

## 5. Stopping a Container — SIGTERM vs SIGKILL

This is critical knowledge for SREs. There is a meaningful difference.

```
docker stop <name>
  │
  ├── Sends SIGTERM to PID 1 inside container
  │   └── Well-behaved apps: flush buffers, close connections, exit cleanly
  │
  ├── Waits 10 seconds (default grace period)
  │   └── Change with: docker stop -t 30 <name>
  │
  └── If still running after timeout → sends SIGKILL (force kill)

docker kill <name>
  │
  └── Sends SIGKILL immediately — no grace period
      └── Use only when container is frozen/unresponsive
```

**Why this matters in production:**
- A database container needs SIGTERM to flush the write-ahead log before dying
- A web server needs SIGTERM to drain active requests
- SIGKILL can corrupt data or cause missed requests
- Always try `docker stop` first; only escalate to `docker kill` if it hangs

```bash
docker stop web              # graceful, 10s timeout
docker stop -t 30 web        # graceful, 30s timeout
docker kill web              # immediate SIGKILL
docker kill --signal SIGHUP web   # send custom signal
```

---

## 6. Restart Policies — Production Behaviour

```bash
# Never restart (default — containers stay stopped after exit)
--restart=no

# Always restart, even after daemon restart
--restart=always

# Restart only on failure (exit code != 0)
--restart=on-failure

# Restart up to N times on failure
--restart=on-failure:5

# Like always, but respects manual docker stop
--restart=unless-stopped
```

**Practical rules:**
- Production services: `--restart=always` or `--restart=unless-stopped`
- One-off jobs: `--restart=no` (default)
- Flaky services: `--restart=on-failure:3` — fail loud after 3 attempts instead of looping forever
- `unless-stopped` is preferred over `always` when you want `docker stop` to stick through daemon restarts

---

## 7. Container Lifecycle — SRE Incident Workflow

```
STEP 1: Is the container running?
        docker ps
        └── Not listed → docker ps -a (check if stopped/exited)

STEP 2: Why did it stop?
        docker ps -a
        └── Look at STATUS column: "Exited (137)" → OOMkilled
        └── "Exited (1)"   → application error
        └── "Exited (0)"   → clean exit (maybe a job that completed)

STEP 3: What did it say before dying?
        docker logs web
        docker logs --tail 200 web
        docker logs --since 30m web

STEP 4: Check resource usage (if still running but slow)
        docker stats web

STEP 5: Get inside (if still running)
        docker exec -it web bash

STEP 6: Restart with fix
        docker stop web
        docker rm web
        docker run ... (with fix applied)
```

---

## 8. Ephemeral vs Persistent Containers

```
Ephemeral containers (stateless)        Persistent containers (stateful)
──────────────────────────────          ────────────────────────────────
Web servers                             Databases (MySQL, Postgres)
API services                            Message queues (RabbitMQ, Kafka)
Workers / job runners                   Caches (Redis with persistence)
Batch processes                         File servers

Design principle:                       Design principle:
Kill and replace freely.                Use named volumes.
State lives elsewhere.                  Backup volumes before rm.
--rm flag is your friend.               Never use --rm.
```

---

## 9. Commands — Quick Reference

```bash
# === CREATE / RUN ===
docker create --name web nginx          # create without starting
docker start web                        # start created container
docker run -d --name web nginx          # create + start (most common)
docker run -it --rm ubuntu bash         # interactive, delete on exit

# === STOP / KILL ===
docker stop web                         # SIGTERM + 10s timeout
docker stop -t 30 web                   # SIGTERM + 30s timeout
docker kill web                         # immediate SIGKILL
docker kill --signal SIGHUP web         # custom signal

# === PAUSE / RESUME ===
docker pause web
docker unpause web

# === RESTART ===
docker restart web
docker restart -t 5 web                 # 5s timeout before kill

# === REMOVE ===
docker rm web                           # remove stopped container
docker rm -f web                        # force remove running container
docker container prune                  # remove all stopped containers

# === INSPECT STATE ===
docker ps                               # running only
docker ps -a                            # all states
docker ps -a --filter status=exited     # only exited containers
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Image}}"
docker inspect web                      # full config + state
docker inspect web --format '{{.State.Status}}'   # just the state
docker inspect web --format '{{.State.ExitCode}}' # exit code
```

---

## 10. Quick Revision

| Question | Answer |
|---|---|
| What are the container states? | Created, Running, Paused, Stopped/Exited, Dead, Deleted |
| What does `docker run` actually do? | `docker create` + `docker start` |
| What is exit code 137? | Killed by SIGKILL — OOM killer or `docker kill` |
| What is exit code 143? | Killed by SIGTERM — graceful stop |
| What is exit code 0? | Clean exit — process ended normally |
| What signal does `docker stop` send first? | SIGTERM, then SIGKILL after timeout |
| How do you change `docker stop` timeout? | `docker stop -t 30 web` |
| What does `docker pause` do? | Freezes all processes (SIGSTOP / cgroup freezer) |
| What restart policy for production services? | `--restart=always` or `--restart=unless-stopped` |
| How do you auto-delete a container on exit? | `--rm` flag in `docker run` |
| Container exited — how do you find why? | `docker ps -a` for exit code, `docker logs` for output |
| What flag prevents container from writing to its filesystem? | `--read-only` |

---

*Next note:* `03-Basic-Docker-Commands.md`
