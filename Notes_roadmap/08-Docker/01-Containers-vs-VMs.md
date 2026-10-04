# 01 — Containers vs VMs

**Tags:** #docker #containers #vms  
**Vault:** `SRE-2027/08-Docker/01-Containers-vs-VMs.md`

---

## Concept

VMs virtualise hardware — each gets its own OS kernel. Containers virtualise the OS — they share the host kernel and isolate processes using namespaces and cgroups. Containers are faster, smaller, and cheaper to run.

---

## Key Points

- VM = hardware emulation → full OS per VM → heavy (~GBs, minutes to boot)
- Container = process isolation → shared kernel → light (~MBs, seconds to start)
- Containers use **namespaces** (isolation) + **cgroups** (resource limits)
- VMs offer stronger isolation — separate kernel per VM
- Containers offer better density — run 10x more on the same host
- Both can coexist — containers often run inside VMs in cloud environments

---

## Diagram

```
VMs                                  Containers
──────────────────────────           ──────────────────────────
┌────────┐ ┌────────┐               ┌──────┐ ┌──────┐ ┌──────┐
│ App A  │ │ App B  │               │App A │ │App B │ │App C │
├────────┤ ├────────┤               ├──────┴─┴──────┴─┴──────┤
│  OS    │ │  OS    │               │       Docker Engine     │
├────────┴─┴────────┤               ├────────────────────────┤
│    Hypervisor     │               │      Host OS Kernel     │
├───────────────────┤               ├────────────────────────┤
│    Hardware       │               │       Hardware         │
└───────────────────┘               └────────────────────────┘
  Each VM: ~1-10GB                    Each container: ~10-200MB
  Boot: minutes                       Start: seconds
```

---

## Example

```bash
# Container starts in milliseconds
docker run --rm alpine echo "hello"

# Check container uses host kernel
docker run --rm alpine uname -r   # same as: uname -r on host
```

---

## SRE Relevance

Containers are the unit of deployment in modern SRE — you need to understand why they replaced VMs for most workloads and where VMs still win (stronger isolation, different kernel).

---

## Quick Revision

| Question | Answer |
|---|---|
| What do containers share with the host? | The OS kernel |
| What provides container isolation? | Namespaces |
| What limits container resources? | cgroups |
| VM boot time vs container start time? | Minutes vs seconds |
| Where do VMs still win? | Stronger isolation (different kernel per VM) |
| Typical VM size vs container size? | GBs vs MBs |
