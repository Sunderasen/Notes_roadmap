# DaemonSet

Tags: #kubernetes #daemonset #sre #cka

## Concept
A DaemonSet puts a Pod on every eligible node, or on a chosen set of nodes.

## Definition — interview answer
A DaemonSet controller ensures each eligible node runs a copy of a Pod. When a new eligible node joins, the controller arranges a Pod for it.

## Why We Need It
Some software needs to run **on nodes**, not as a fixed number of application replicas. Examples: log collection and node monitoring agents.

## How It Works
```text
Nodes:         node-a    node-b    node-c
DaemonSet:     agent     agent     agent
New node-d ->  agent also runs on node-d
```
Eligibility can depend on node labels, taints/tolerations, and scheduling rules. Not every DaemonSet must run on every node.

## Working YAML
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-web-demo
  namespace: dev
spec:
  selector:
    matchLabels:
      app: node-web-demo
  template:
    metadata:
      labels:
        app: node-web-demo
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
```
This is a learning demo, not a typical reason to use a DaemonSet. On your single-node Minikube cluster, expect **one Pod if the node is eligible**.

## What to Learn
```bash
kubectl apply -f daemonset.yaml
kubectl get ds -n dev
kubectl get pods -n dev -o wide -l app=node-web-demo
kubectl describe ds node-web-demo -n dev
kubectl delete ds node-web-demo -n dev
```

## Real-World Example
A new worker node is added during scaling. The DaemonSet places a logging agent on it automatically. Without that agent, logs from applications on the new node might not reach the central log system.

## Troubleshooting
**DaemonSet Pod missing from a node:**
```bash
kubectl get ds -n dev
kubectl describe ds node-web-demo -n dev
kubectl get pods -n dev -o wide -l app=node-web-demo
kubectl describe node <node-name>
```
Compare desired/current/ready counts. Check node eligibility, scheduling events, and container startup errors.

## Interview Questions
**Q: ReplicaSet vs DaemonSet?**  
A: ReplicaSet maintains a count; DaemonSet covers eligible nodes.

**Q: What happens when a new node joins?**  
A: A DaemonSet Pod is scheduled there if the node is eligible.

**Q: Why use DaemonSet for log agents?**  
A: Each node needs its own agent to collect local logs.

**Q: Why only one DaemonSet Pod in Minikube?**  
A: Your cluster has one eligible node.

## Quick Revision
- DaemonSet = per eligible node.
- ReplicaSet = fixed desired count.
- Typical use: logging, monitoring, networking agents.
