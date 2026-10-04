# 05 — Container Lifecycle

**Tags:** #docker #lifecycle #states #signals  
**Vault:** `SRE-2027/08-Docker/05-Container-Lifecycle.md`

---

## Concept

A container moves through defined states from creation to deletion. Knowing the states and exit codes tells you exactly what happened to a container during an incident — without guessing.

---

## Key Points

- `created` → exists, never started
- `running` → process active
- `paused` → frozen (SIGSTOP), still in memory
- `exited` → process ended; still visible in `docker ps -a`
- `dead` → removal failed; rare kernel/storage issue
- `docker stop` = SIGTERM → waits 10s → SIGKILL
- `docker kill` = SIGKILL immediately — use only when frozen
- Exit code `0` = clean exit, `1` = app error, `137` = OOM/force killed, `143` = graceful stop
- `--restart=always` = auto-restart including after daemon restart
- `--restart=unless-stopped` = auto-restart but respects manual `docker stop`

---

## Diagram

```
docker create
     │
     ▼
  CREATED
     │ docker start / docker run
     ▼
  RUNNING ◄─────────────── docker restart
     │         │        │
  docker     docker   process
  pause      stop     exits
     │         │        │
  PAUSED    EXITED    EXITED
     │         │
  docker     docker start
  unpause       │
     │         ▼
     └──► RUNNING

  Any state → docker rm → DELETED
```

---

## Example

```bash
# Full lifecycle
docker run -d --name app alpine sleep 300   # running
docker pause app                             # paused
docker unpause app                           # running
docker stop app                              # exited (143)
docker start app                             # running
docker kill app                              # exited (137)
docker rm app                                # deleted

# Find why a container stopped
docker ps -a                                 # see exit code in STATUS
docker logs app                              # see last output
docker inspect app --format '{{.State.ExitCode}}'
```

---

## SRE Relevance

Exit codes are your first clue in any container incident. `137` = OOM or force kill → check memory limits. `1` = app crashed → check logs immediately.

---

## Quick Revision

| Question | Answer |
|---|---|
| What signal does `docker stop` send first? | SIGTERM |
| What if container doesn't stop in 10s? | Docker sends SIGKILL |
| Exit code 137 means? | Killed by SIGKILL — OOM or `docker kill` |
| Exit code 143 means? | Killed by SIGTERM — graceful `docker stop` |
| Exit code 0 means? | Clean exit |
| `docker ps` vs `docker ps -a`? | Running only vs all states |
| `--restart=always` vs `--restart=unless-stopped`? | Always: ignores manual stop; unless-stopped: respects it |
| How to delete container and its anonymous volumes? | `docker rm -v <name>` |
