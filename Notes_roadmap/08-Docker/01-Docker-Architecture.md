# Docker Architecture

**Tags:** #docker #architecture #daemon #sre  
**Vault Path:** `SRE-2027/08-Docker/01-Docker-Architecture.md`

---

## 1. Concept

Docker is a **client-server application**. When you type `docker run`, you are not directly running a container — you are sending an instruction to a background service (the daemon) that does the actual work. Understanding this architecture explains why things fail, how remote Docker works, and what each moving part is responsible for.

---

## 2. Definition — The Big Picture

```
┌─────────────────────────────────────────────────────────────────┐
│                        DOCKER HOST                              │
│                                                                 │
│   ┌─────────────┐        ┌──────────────────────────────────┐  │
│   │ Docker CLI  │        │         Docker Daemon            │  │
│   │  (client)  │──REST──▶│         (dockerd)                │  │
│   │            │        │                                  │  │
│   │ docker run │        │  ┌──────────┐  ┌─────────────┐  │  │
│   │ docker ps  │        │  │  Images  │  │  Containers │  │  │
│   │ docker logs│        │  └──────────┘  └─────────────┘  │  │
│   └─────────────┘        │                                  │  │
│                           │  ┌──────────┐  ┌─────────────┐  │  │
│                           │  │ Networks │  │   Volumes   │  │  │
│                           │  └──────────┘  └─────────────┘  │  │
│                           └──────────────────────────────────┘  │
│                                      │                          │
│                                      ▼                          │
│                           ┌──────────────────┐                  │
│                           │   containerd     │  (container      │
│                           │                  │   runtime)       │
│                           └────────┬─────────┘                  │
│                                    │                            │
│                           ┌────────▼─────────┐                  │
│                           │    runc / shim   │  (OCI runtime)   │
│                           └────────┬─────────┘                  │
│                                    │                            │
│                    ┌───────────────┼───────────────┐            │
│                    ▼               ▼               ▼            │
│              ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│              │Container1│  │Container2│  │Container3│          │
│              └──────────┘  └──────────┘  └──────────┘          │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                       ┌────────────────────────┐
                       │    Docker Registry      │
                       │  (DockerHub / ECR / GCR)│
                       └────────────────────────┘
```

---

## 3. Components — Definitions

### Docker CLI (Client)
- The tool you interact with: `docker`, `docker compose`
- Sends commands to the daemon over a **REST API**
- By default talks to the local daemon via a **Unix socket**: `/var/run/docker.sock`
- Can be pointed at a **remote daemon** using `DOCKER_HOST` env variable

```bash
# Talk to a remote Docker daemon
export DOCKER_HOST=tcp://192.168.1.100:2376
docker ps   # now running against the remote host
```

### Docker Daemon (`dockerd`)
- The background service that does all the real work
- Listens on a Unix socket (default) or TCP port
- Manages: images, containers, networks, volumes
- Talks to `containerd` to actually run containers
- Config file: `/etc/docker/daemon.json`

```bash
# Check if daemon is running
systemctl status docker

# View daemon logs
journalctl -u docker -f

# Daemon config
cat /etc/docker/daemon.json
```

### containerd
- The **container runtime** — handles the actual lifecycle of containers
- Pulled out of Docker as a separate project, now a CNCF standard
- Kubernetes also uses containerd directly (bypasses dockerd)
- Manages: image pulling, container creation, storage, networking hooks

### runc
- The **low-level OCI runtime** — actually creates the container process
- Calls Linux kernel features: namespaces, cgroups, seccomp, capabilities
- `containerd` calls `runc` to spawn the container
- You never interact with `runc` directly

### Docker Registry
- Storage for Docker images (layers + metadata)
- **DockerHub**: default public registry (`docker.io`)
- **ECR** (AWS), **GCR** (Google), **ACR** (Azure): cloud registries
- **Harbor**, **Nexus**: self-hosted private registries
- `docker pull` = download from registry; `docker push` = upload to registry

---

## 4. How a `docker run` Works — Step by Step

```
You type: docker run nginx

Step 1: CLI → Daemon
        Docker CLI sends REST API call to dockerd
        POST /containers/create

Step 2: Daemon checks image cache
        Is nginx image already pulled locally?
        └── YES → skip to Step 4
        └── NO  → Step 3

Step 3: Daemon pulls image from registry
        Contacts DockerHub (or configured registry)
        Downloads image layers in parallel
        Stores in /var/lib/docker/overlay2/

Step 4: Daemon asks containerd to create container
        containerd prepares rootfs from image layers
        Sets up writable layer on top (copy-on-write)

Step 5: containerd calls runc
        runc creates Linux namespaces:
          - PID namespace  (isolated process tree)
          - NET namespace  (isolated network stack)
          - MNT namespace  (isolated filesystem)
          - UTS namespace  (isolated hostname)
          - IPC namespace  (isolated IPC)
          - USER namespace (optional, isolated UIDs)
        runc sets up cgroups (CPU/mem limits)
        runc spawns container process

Step 6: Container is running
        Daemon tracks it, assigns IP, sets up networking
        You get the container ID back
```

---

## 5. Key Linux Primitives Docker Uses

Docker is not magic — it's a clean interface over Linux kernel features.

| Primitive | What it does | Docker use |
|---|---|---|
| **Namespaces** | Isolate processes from each other | Each container gets its own PID, NET, MNT, UTS, IPC namespace |
| **cgroups** | Limit and account resource usage | CPU/memory limits on containers |
| **Union Filesystem** | Stack layers into one view | Image layers + writable container layer (overlay2) |
| **seccomp** | Filter system calls | Restricts which syscalls a container can make |
| **Capabilities** | Fine-grained root privileges | Drop dangerous capabilities from containers |

```bash
# See namespaces of a running container
docker inspect <container> | grep -i pid
ls -la /proc/<pid>/ns/

# See cgroups
cat /sys/fs/cgroup/memory/docker/<container_id>/memory.usage_in_bytes
```

---

## 6. Image Layers — How Storage Works

```
Image: myapp:v2
─────────────────────────────────
Layer 4 (R/W): Container layer    ← writes go here, dies with container
Layer 3 (R):   COPY app/ .        ← your app code
Layer 2 (R):   RUN pip install    ← dependencies
Layer 1 (R):   FROM python:3.11   ← base image
─────────────────────────────────
Driver: overlay2 (default on Linux)
```

- Each layer is a **content-addressed** tar archive stored in `/var/lib/docker/overlay2/`
- Layers are **shared** across images — if two images use the same base, it's stored once
- The **writable layer** is unique per container — deleted when container is removed
- This is why images are fast to pull when you already have some layers cached

```bash
# See image layers
docker history myapp:v2

# See where Docker stores everything
ls /var/lib/docker/
```

---

## 7. Docker Socket — The Power and the Risk

The Docker socket `/var/run/docker.sock` is how the CLI talks to the daemon.

```bash
# If you mount the socket into a container, that container can control Docker
docker run -v /var/run/docker.sock:/var/run/docker.sock docker docker ps

# This is effectively root on the host — a critical security concern
# Never do this in production unless you understand the implications
```

**SRE relevance:** CI systems (Jenkins, GitLab Runner) often mount the Docker socket to build images inside containers — this is a common attack surface. In secure environments, use **Docker-in-Docker (dind)** or **Kaniko** instead.

---

## 8. SRE Relevance

| Situation | What architecture knowledge helps with |
|---|---|
| Container won't start | Know that `containerd` and `runc` are below `dockerd` — check `journalctl -u containerd` |
| Docker daemon is down | All containers keep running — daemon manages, but doesn't host them |
| Disk full on host | Know images live in `/var/lib/docker/overlay2/` — target that for cleanup |
| Remote Docker access | `DOCKER_HOST` or Docker Context — point CLI at remote daemon |
| Security audit | Understand socket exposure, capabilities, namespace isolation |
| Kubernetes transition | K8s uses `containerd` directly — Docker knowledge transfers cleanly |

**Important:** If `dockerd` crashes, **running containers keep running**. The daemon is the management plane, not the execution plane. This is why Kubernetes doesn't need Docker.

---

## 9. Commands — Quick Reference

```bash
# === DAEMON ===
systemctl status docker          # Is daemon running?
systemctl restart docker         # Restart daemon
journalctl -u docker -f          # Follow daemon logs
cat /etc/docker/daemon.json      # Daemon config

# === SYSTEM INFO ===
docker info                      # Full daemon info: runtime, storage driver, OS
docker version                   # Client and daemon versions
docker system df                 # Disk usage by images/containers/volumes/cache
docker system events             # Real-time daemon events

# === STORAGE ===
docker info | grep -i storage    # Storage driver in use
ls /var/lib/docker/              # Docker's data root
docker history <image>           # Image layers
docker inspect <image>           # Full image metadata

# === REMOTE DAEMON ===
export DOCKER_HOST=tcp://host:2376
docker context create remote --docker "host=tcp://host:2376"
docker context use remote
docker context ls
```

---

## 10. Quick Revision

| Question | Answer |
|---|---|
| What is dockerd? | The Docker daemon — manages images, containers, networks, volumes |
| What is containerd? | The container runtime that dockerd delegates to |
| What is runc? | Low-level OCI runtime that actually creates container processes |
| What Linux features does Docker use? | Namespaces, cgroups, union filesystem (overlay2), seccomp, capabilities |
| Where does Docker store images on disk? | `/var/lib/docker/overlay2/` |
| What is the Docker socket? | `/var/run/docker.sock` — Unix socket for CLI ↔ daemon communication |
| If dockerd crashes, do containers stop? | No — they keep running; daemon is management plane only |
| What is a Docker registry? | Storage for images — DockerHub, ECR, GCR, ACR, Harbor |
| What is the default registry? | DockerHub (`docker.io`) |
| How does CLI communicate with daemon? | REST API over Unix socket or TCP |
| What is overlay2? | The default union filesystem driver — stacks image layers |
| How are image layers shared? | Content-addressed — same layer used by multiple images stored once |

---

*Next note:* `02-Container-Lifecycle.md`
