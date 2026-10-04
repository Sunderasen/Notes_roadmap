# 06 — Docker Networking

**Tags:** #docker #networking #bridge #dns  
**Vault:** `SRE-2027/08-Docker/06-Docker-Networking.md`

---

## Concept

Docker networking controls how containers communicate with each other, the host, and the outside world. Each container gets its own network namespace. The network driver determines how that namespace connects to everything else.

---

## Key Points

- **bridge** (default) — private virtual network on host; containers need port mapping to be reachable
- **custom bridge** — same as bridge but with **automatic DNS** (containers reach each other by name)
- **host** — container shares host network stack; no port mapping needed; no isolation
- **none** — no network; loopback only; maximum isolation
- **overlay** — multi-host networking for Docker Swarm
- Default bridge has **no DNS** — containers must use IPs
- Custom bridge has **embedded DNS at `127.0.0.11`** — use container name as hostname
- Port mapping (`-p`) creates an iptables NAT rule on the host
- Always create a custom network for multi-container apps

---

## Diagram

```
Custom bridge network (mynet)
─────────────────────────────────────────────
  ┌──────────┐          ┌──────────┐
  │  web     │          │   db     │
  │ :80      │◄────────►│ :5432    │
  │172.20.0.2│  by name │172.20.0.3│
  └──────────┘   (DNS)  └──────────┘
        │
        │ -p 8080:80 (NAT)
        ▼
  host:8080 → outside world

  web → ping db   ✅  (same network, DNS works)
  web → ping db   ❌  (default bridge, no DNS)
```

---

## Example

```bash
# Create custom network
docker network create mynet

# Run containers on it — they find each other by name
docker run -d --name api --network mynet alpine sleep 300
docker run -d --name db  --network mynet alpine sleep 300

# api can reach db by name
docker exec -it api ping db        # works
docker exec -it api wget -qO- http://db:5432

# Inspect who's on the network
docker network inspect mynet
```

---

## SRE Relevance

90% of container connectivity issues come from containers being on different networks or using the default bridge (no DNS). Always verify with `docker network inspect` and `docker exec + ping`.

---

## Quick Revision

| Question | Answer |
|---|---|
| Default network driver? | bridge |
| Why does DNS fail on default bridge? | No embedded DNS — containers must use IPs |
| Where is Docker's DNS server? | `127.0.0.11` inside containers on custom networks |
| How do containers find each other by name? | Custom bridge network — Docker resolves container name |
| `host` network driver does what? | Container shares host network — no isolation, no port mapping needed |
| `none` network driver does what? | No network at all — loopback only |
| How to check container connectivity? | `docker exec -it <name> ping <other-name>` |
| How does port mapping work internally? | Docker writes iptables NAT rules |
