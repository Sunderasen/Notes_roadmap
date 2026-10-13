# DaemonSet

Tags: #kubernetes #daemonset #sre

## Concept
A DaemonSet runs a Pod on every eligible node, or on a selected group of nodes.

## Definition
When a matching node joins, Kubernetes makes sure it gets a DaemonSet Pod. When a node leaves, that node's Pod goes away with it.

## How It Works
```text
DaemonSet: logging-agent
  Node A -> logging-agent Pod
  Node B -> logging-agent Pod
  Node C -> logging-agent Pod
```

A ReplicaSet controls **how many Pods**. A DaemonSet controls **which eligible nodes have a Pod**.

## Example
A log collection agent is needed on each worker node. When a new node is added, the DaemonSet places an agent Pod there.

## SRE Relevance
Common for node log agents, monitoring agents, and some networking components.

## Commands
```bash
kubectl get daemonsets -A
kubectl get ds -n kube-system
kubectl describe ds <daemonset-name> -n <namespace>
kubectl get pods -o wide -n <namespace>
```

## Quick Revision
- ReplicaSet = desired count.
- DaemonSet = Pod on each eligible node.
- Useful for logging and monitoring agents.
- A new eligible node gets a DaemonSet Pod.
