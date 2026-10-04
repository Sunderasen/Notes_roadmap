# 09 — Docker Logs & Troubleshooting

**Tags:** #docker #logs #troubleshooting #debugging #sre  
**Vault:** `SRE-2027/08-Docker/09-Docker-Logs-Troubleshooting.md`

---

## Concept

Docker captures everything a container writes to stdout and stderr as logs. Troubleshooting always follows the same order: is it running → why did it stop → what did it say → what is it using → get inside.

---

## Key Points

- `docker logs` captures **stdout + stderr** only — logs to files inside container are invisible
- `-f` → follow live (like `tail -f`)
- `--tail N` → last N lines
- `--since` → time-scoped logs (e.g. `--since 10m`, `--since 2025-09-01`)
- Default log driver: `json-file` — logs stored at `/var/lib/docker/containers/<id>/<id>-json.log`
- Logs grow forever by default — set `--log-opt max-size=10m --log-opt max-file=3`
- `docker stats` → live CPU, memory, network, disk I/O
- `docker exec -it <name> sh` → get inside running container
- `docker inspect` → full config dump — ports, mounts, env, network, exit code
- `docker events` → real-time daemon event stream (start, die, kill, oom...)

---

## Diagram

```
Container is broken — what do you do?
──────────────────────────────────────────────────
Step 1: docker ps              Is it running?
         └── No → docker ps -a  Did it exit? What code?

Step 2: docker logs --tail 100 <name>
         └── What did the app say before dying?

Step 3: docker stats <name>
         └── Is it OOMing? CPU throttled?

Step 4: docker inspect <name>
         └── Wrong env? Bad mount? Wrong port?

Step 5: docker exec -it <name> sh
         └── Get inside and poke around

Step 6: docker events --since 1h
         └── What did the daemon see?
```

---

## Example

```bash
# Logs
docker logs myapp
docker logs -f myapp                       # follow
docker logs --tail 100 myapp               # last 100 lines
docker logs --since 10m myapp              # last 10 minutes
docker logs --since 10m -f myapp           # scoped + follow

# Stats
docker stats                               # all containers live
docker stats --no-stream                   # snapshot

# Inspect specific fields
docker inspect myapp --format '{{.State.ExitCode}}'
docker inspect myapp --format '{{json .Config.Env}}'
docker inspect myapp --format '{{json .Mounts}}'
docker inspect myapp --format '{{.NetworkSettings.IPAddress}}'

# Get inside
docker exec -it myapp sh

# Copy log file out of container
docker cp myapp:/var/log/app/error.log ./

# Real-time events
docker events --since 1h
docker events --filter container=myapp
docker events --filter event=die           # only deaths
```

---

## SRE Relevance

This is your incident response toolkit. You'll run these commands in sequence every time a container is misbehaving. Know them cold — especially `docker logs`, `docker stats`, `docker inspect`, and `docker exec`.

---

## Quick Revision

| Question | Answer |
|---|---|
| What does `docker logs` capture? | stdout and stderr only |
| How to follow logs in real time? | `docker logs -f <name>` |
| How to see logs from last 10 minutes? | `docker logs --since 10m <name>` |
| How to check live resource usage? | `docker stats` |
| How to get a shell in running container? | `docker exec -it <name> sh` |
| How to find a container's IP? | `docker inspect <name> --format '{{.NetworkSettings.IPAddress}}'` |
| How to copy a file out of a container? | `docker cp <name>:/path/file ./` |
| Where are json-file logs stored on host? | `/var/lib/docker/containers/<id>/<id>-json.log` |
| How to watch real-time daemon events? | `docker events` |
| How to prevent logs filling disk? | `--log-opt max-size=10m --log-opt max-file=3` |
