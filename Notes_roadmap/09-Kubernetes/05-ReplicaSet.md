# ReplicaSet

Tags: #kubernetes #replicaset #sre #cka

## Concept
A ReplicaSet keeps the requested number of matching Pods present.

## Definition — interview answer
A ReplicaSet is a Kubernetes controller that watches Pods matching its selector and creates or removes Pods to match `spec.replicas`.

## Why We Need It
One Pod may disappear when a node fails or a Pod is deleted. A ReplicaSet restores the desired Pod count. In production, a Deployment usually manages ReplicaSets for us.

## How It Works
```text
ReplicaSet sp-replica: replicas=3
  |- Pod A
  |- Pod B
  |- Pod C
Delete Pod B -> controller creates replacement Pod D
```
A ReplicaSet does not guarantee three **healthy** applications, only the desired count of managed Pods.

## Working YAML
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: sp-replica
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        app: httpd
        tier: frontend
    spec:
      containers:
        - name: cont1
          image: httpd:2.4
          ports:
            - containerPort: 80
```

| Field | Meaning |
|---|---|
| `replicas: 3` | Keep three matching Pods |
| `selector.matchLabels` | How ReplicaSet finds Pods |
| `template` | Blueprint used to create Pods |
| `template.metadata.labels` | Labels put on new Pods; must match selector |
| `template.spec.containers` | Containers each new Pod runs |

The outer `spec` configures the ReplicaSet. `template.spec` configures each Pod.

## What to Learn
```bash
kubectl apply -f replicaset.yaml
kubectl get rs -n dev
kubectl get pods -n dev -l tier=frontend
kubectl describe rs sp-replica -n dev
kubectl scale rs sp-replica --replicas=5 -n dev
kubectl apply --dry-run=server -f replicaset.yaml
```
**Note:** Avoid using the same selector for unrelated ReplicaSets/Deployments; controllers can conflict over Pods.

## Real-World Example
A worker node fails and one ReplicaSet-owned Pod disappears. The controller creates a replacement; the scheduler places it on an available node. If there is no spare capacity, the replacement may remain Pending.

## Troubleshooting
**ReplicaSet says 3 desired but fewer ready:**
```bash
kubectl get rs sp-replica -n dev
kubectl get pods -n dev -l tier=frontend -o wide
kubectl describe rs sp-replica -n dev
kubectl describe pod <pod> -n dev
```
Look for scheduling, image pull, or app startup failures.

**Manifest rejected with unknown field:** Use exact field names:
```yaml
ports:
  - containerPort: 80
```
Not `port:` or `containerport:`. Use `kubectl explain pod.spec.containers.ports`.

## Interview Questions
**Q: What happens if replicas changes from 3 to 5?**  
A: The same ReplicaSet creates two more Pods, assuming no other matching Pods affect the count.

**Q: ReplicaSet vs Deployment?**  
A: ReplicaSet maintains Pod count; Deployment manages ReplicaSets and application rollouts.

**Q: Why must selector match template labels?**  
A: Otherwise the controller cannot reliably manage the Pods it creates; the API rejects a mismatched ReplicaSet template.

**Q: Does ReplicaSet place one Pod on every node?**  
A: No. The scheduler decides placement; DaemonSet is for one Pod per eligible node.

## Quick Revision
- ReplicaSet = desired Pod count.
- `selector` matches `template.metadata.labels`.
- Scaling a ReplicaSet does not create a new ReplicaSet.
- Usually manage through Deployment.
