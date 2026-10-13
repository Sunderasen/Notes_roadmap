# Kubernetes Architecture

Tags: #kubernetes #architecture #sre #cka

## Concept
Kubernetes runs containers and helps keep applications running.

## Definition
Kubernetes compares **what we want** (desired state) with **what is running** (actual state). Controllers work to fix differences.

## How It Works
```text
kubectl -> API server -> etcd (cluster data)
                 |
                 +-> Controller manager (checks desired state)
                 +-> Scheduler (chooses a node)
                              |
                              v
                         Kubelet on node
                              |
                         Container runtime
                              |
                             Pod
```

- **API server:** Entry point for Kubernetes requests.
- **etcd:** Stores cluster configuration and state.
- **Controller manager:** Runs controllers that notice and correct differences.
- **Scheduler:** Chooses a suitable node for a new Pod.
- **Kubelet:** Agent on each node; makes sure assigned Pods run.
- **Container runtime:** Runs containers (for example, containerd).

## Example
You ask for three replicas. One Pod disappears. The controller notices only two matching Pods and creates a replacement. The scheduler picks a node, and the kubelet starts it.

## SRE Relevance
During an outage, find **where** the failure happened: API access, scheduling, image pull, container startup, or application readiness.

## Commands
```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
kubectl describe node <node-name>
```

## Quick Revision
- API server = entry point.
- etcd = cluster state.
- Scheduler = picks node.
- Kubelet = runs assigned Pods.
- Controller = works toward desired state.
