# Docker Networking

**Tags:** #docker #networking #bridge #dns #sre  
**Vault Path:** `SRE-2027/08-Docker/04-Docker-Networking.md`

---

## 1. Concept

Every container needs to communicate — with the host, with other containers, or with the outside world. Docker networking controls **how** that communication flows. As an SRE, you'll debug connectivity failures, set up isolated networks for security, and trace how traffic moves between services. This is foundational for understanding Kubernetes networking later.

---

## 2. Definition — Network Drivers

Docker networking is built on **drivers**. Each driver creates a different type of network with different connectivity rules.

```
┌──────────────────────────────────────────────────────────────────┐
│                        DOCKER HOST                               │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    bridge (docker0)                       │   │
│  │                   172.17.0.0/16                          │   │
│  │                                                          │   │
│  │  ┌───────────┐   ┌───────────┐   ┌───────────┐          │   │
│  │  │Container 1│   │Container 2│   │Container 3│          │   │
│  │  │172.17.0.2 │   │172.17.0.3 │   │172.17.0.4 │          │   │
│  │  └───────────┘   └───────────┘   └───────────┘          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                          │ NAT                                    │
│                          ▼                                        │
│                   eth0 (host NIC)                                 │
│                   192.168.1.100                                   │
└──────────────────────────────────────────────────────────────────┘
                          │
                          ▼
                     External Network / Internet
```

---

## 3. Network Types

### bridge (Default)

The default network driver. Docker creates a virtual network (`docker0`) on the host. Containers get IPs in a private subnet and communicate through this bridge.

```bash
# Default bridge — containers NOT accessible by name (no DNS)
docker run -d --name web nginx
docker run -d --name db mysql
# web CANNOT reach db by hostname on default bridge

# Check it
docker network ls           # see "bridge" network
docker network inspect bridge
```

**Default bridge limitations:**
- No automatic DNS — containers must use IPs to communicate
- All containers on the same bridge can reach each other (no isolation)
- `:latest` anti-pattern — IPs change on restart

**Custom bridge — the right way:**

```bash
# Create a custom bridge network
docker network create mynet

# Run containers on it
docker run -d --name web --network mynet nginx
docker run -d --name db --network mynet mysql

# web CAN reach db by hostname now — Docker provides DNS
docker exec -it web ping db             # works!
docker exec -it web curl http://api:8080
```

**Custom bridge advantages:**
- Automatic DNS — containers reach each other by **container name**
- Network isolation — only containers on the same network can communicate
- Multiple networks possible — fine-grained security segmentation

---

### host

Container shares the **host's network stack** directly. No network isolation — container sees all host interfaces, uses host ports.

```bash
docker run -d --network host nginx
# nginx now listens on host port 80 directly — no -p mapping needed
# curl http://localhost:80 works from host immediately
```

**When to use:**
- Maximum network performance (no NAT overhead)
- Containers that need to bind to many ports
- Network monitoring tools that need raw access to host network

**Risks:**
- Container can access all host network services
- Port conflicts — container and host share the same port space
- Less portable — tied to host network config

```bash
# See that there's no separate network namespace
docker run --rm --network host busybox ip addr
# Output will show host's network interfaces
```

---

### none

Container has **no network** at all. Completely isolated. Only loopback interface.

```bash
docker run -d --network none myapp
# Container has no eth0, no internet, no container-to-container
# Only localhost (127.0.0.1) works inside
```

**When to use:**
- Batch processing jobs that don't need network
- Maximum security isolation
- Testing network-independent code paths

---

### overlay (Swarm / multi-host)

Allows containers on **different Docker hosts** to communicate as if on the same network. Used with Docker Swarm.

```bash
# Requires Docker Swarm mode
docker swarm init
docker network create --driver overlay --attachable myoverlay
docker service create --network myoverlay --name web nginx
```

**Relevance:** This is Docker's answer to multi-host networking. In production you'll use Kubernetes CNI plugins (Calico, Flannel, Cilium) instead — same concept, different implementation.

---

### macvlan

Assigns a **MAC address** to a container, making it appear as a physical device on the network. Container gets an IP directly from the physical network (no NAT).

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  mymacvlan

docker run -d --network mymacvlan --ip 192.168.1.50 nginx
```

**When to use:**
- Legacy apps that expect a real MAC/IP on the physical network
- Network monitoring (promiscuous mode)
- Avoiding NAT for performance

---

## 4. Driver Comparison

| Feature | bridge | host | none | overlay |
|---|---|---|---|---|
| Isolation | Partial | None | Full | Partial |
| DNS between containers | Custom only | N/A | N/A | Yes |
| Multi-host | No | No | No | Yes |
| NAT | Yes | No | N/A | Yes |
| Port mapping needed | Yes | No | N/A | No |
| Best for | Dev + Prod (single host) | Performance | Security | Swarm |

---

## 5. Docker DNS — How Containers Find Each Other

On a **custom bridge network**, Docker runs an embedded DNS server at `127.0.0.11`.

```
Container "web" wants to reach "db":
  1. web does: curl http://db:5432
  2. OS resolver in web queries 127.0.0.11 (Docker DNS)
  3. Docker DNS looks up "db" → 172.20.0.3
  4. web connects to 172.20.0.3:5432
```

```bash
# Check DNS from inside a container
docker exec -it web cat /etc/resolv.conf
# Output: nameserver 127.0.0.11

docker exec -it web nslookup db        # should resolve
docker exec -it web ping db            # should work on custom network

# Default bridge — DNS does NOT work
docker run -d --name web nginx
docker run -d --name db nginx
docker exec -it web ping db            # FAILS — use IP instead
```

**Critical rule:** Always use a **custom bridge network** in Compose and multi-container setups. The default bridge has no DNS.

---

## 6. Port Mapping

Port mapping (`-p`) creates a NAT rule: traffic hitting `host:port` is forwarded to `container:port`.

```bash
# Syntax: -p host_port:container_port
docker run -p 8080:80 nginx         # all interfaces → container 80
docker run -p 443:443 nginx         # HTTPS
docker run -p 127.0.0.1:8080:80 nginx   # localhost only (more secure)
docker run -p 0.0.0.0:8080:80 nginx     # explicit all interfaces

# Multiple ports
docker run -p 80:80 -p 443:443 nginx

# Random host port
docker run -p 80 nginx              # Docker assigns random host port
docker port <container>             # find what port was assigned

# See all mappings
docker port web
docker inspect web --format '{{json .NetworkSettings.Ports}}'
```

**How it works under the hood:**

```
Incoming: host:8080
  ↓
iptables NAT rule (DOCKER chain)
  ↓
Container eth0: 172.17.0.2:80
```

Docker writes `iptables` rules automatically. You can verify:
```bash
sudo iptables -t nat -L DOCKER -n
```

---

## 7. Container-to-Container Communication

```bash
# Pattern 1: Same custom network (recommended)
docker network create backend
docker run -d --name api --network backend myapi
docker run -d --name db  --network backend postgres
# api can reach db at: postgres://db:5432

# Pattern 2: Multiple networks (for tiered architectures)
docker network create frontend
docker network create backend

docker run -d --name web --network frontend nginx
docker run -d --name api --network frontend --network backend myapi
docker run -d --name db  --network backend postgres

# web → api: YES (both on frontend)
# api → db:  YES (both on backend)
# web → db:  NO  (different networks — isolated)
# This is network segmentation — frontend can't directly hit DB

# Attach a running container to a second network
docker network connect backend api
```

---

## 8. Networking in Docker Compose

Compose automatically creates a **custom bridge network** for each project and connects all services to it. Services reach each other by service name.

```yaml
version: "3.9"

services:
  web:
    image: nginx
    ports:
      - "8080:80"
    networks:
      - frontend

  api:
    image: myapi:v2
    networks:
      - frontend
      - backend

  db:
    image: postgres:15
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true      # no external internet access
```

`internal: true` = network has no external routing. DB containers can't make outbound calls — security hardening.

---

## 9. SRE Relevance

| Situation | Command / Action |
|---|---|
| Container can't reach another | `docker network inspect` → are they on the same network? |
| DNS not resolving | Check if using custom bridge, not default |
| Port not accessible from host | `docker port web` → verify mapping exists |
| Investigate container IP | `docker inspect web --format '{{.NetworkSettings.IPAddress}}'` |
| Security: isolate DB network | Use `internal: true` in Compose or separate network |
| Debug connectivity from inside | `docker exec -it web ping db`, `curl`, `nslookup` |
| See iptables rules Docker created | `sudo iptables -t nat -L DOCKER -n` |

**Common incident causes:**
1. Container on default bridge → DNS fails → service can't find dependency
2. Port mapping missing → external traffic can't reach container
3. Two containers on different custom networks → can't communicate
4. `internal: true` network → container can't pull updates / reach external APIs

---

## 10. Commands — Quick Reference

```bash
# === LIST ===
docker network ls
docker network ls --filter driver=bridge

# === CREATE ===
docker network create mynet
docker network create --driver bridge mynet
docker network create --subnet 172.20.0.0/16 --gateway 172.20.0.1 mynet
docker network create --internal mynet              # no external access

# === INSPECT ===
docker network inspect mynet
docker network inspect mynet --format '{{json .Containers}}'
docker network inspect bridge                       # default bridge

# === CONNECT / DISCONNECT ===
docker network connect mynet web
docker network connect --ip 172.20.0.10 mynet web   # specific IP
docker network disconnect mynet web

# === REMOVE ===
docker network rm mynet
docker network prune                                # remove unused

# === DEBUG FROM INSIDE CONTAINER ===
docker exec -it web ping db
docker exec -it web nslookup db
docker exec -it web curl http://db:8080/health
docker exec -it web cat /etc/resolv.conf            # check DNS server
docker exec -it web ip addr                         # container's IPs
docker exec -it web ip route                        # routing table

# === PORT MAPPING ===
docker run -p 8080:80 nginx
docker port web
docker inspect web --format '{{json .NetworkSettings.Ports}}'
```

---

## 11. Quick Revision

| Question | Answer |
|---|---|
| What is the default Docker network driver? | bridge |
| Why does DNS fail on the default bridge? | Default bridge has no embedded DNS — containers must use IPs |
| What network driver gives container access to host network directly? | host |
| What network driver provides zero network access? | none |
| What network driver enables multi-host communication? | overlay |
| Where is Docker's embedded DNS server? | `127.0.0.11` inside containers on custom networks |
| How do containers on a custom bridge find each other? | By container name — Docker DNS resolves it |
| How do you make a network with no external internet access? | `--internal` flag or `internal: true` in Compose |
| How does port mapping work internally? | Docker writes iptables NAT rules |
| How do you attach a running container to a second network? | `docker network connect <network> <container>` |
| Best practice for multi-container apps? | Create a custom bridge network — never rely on default bridge |
| What does `docker network inspect` show you? | Driver, subnet, gateway, connected containers with IPs |

---

*Next note:* `05-Docker-Volumes.md`
