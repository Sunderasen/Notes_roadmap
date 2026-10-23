---
tags: [kubernetes, scheduling, affinity, taints, tolerations, sre]
---
# 09 — Kubernetes Pod Scheduling

## 1. Concept
The **scheduler** chooses a suitable node for a new Pod. The **kubelet** on that node then runs it. Scheduling depends on resource requests, labels, affinity, taints, and storage topology.

## 2. Why It Is Needed
Control where workloads run, keep dependent Pods close, spread replicas, and reserve nodes for selected workloads.

## 3. Node Labels and `nodeSelector`
Labels are key-value metadata on nodes. `nodeSelector` requires exact matches.

```bash
kubectl get nodes --show-labels
kubectl label node <node-name> color=red
kubectl get nodes -L color
```

```yaml
spec:
  nodeSelector:
    color: red
```

A Pod stays **Pending** if no eligible node matches.

## 4. Node Affinity
Node affinity uses **node labels** and supports hard and soft rules.

### Required (hard rule)
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: color
                operator: In
                values: [red]
```

### Preferred (soft rule)
```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: color
                operator: In
                values: [red]
```

- `required...`: node **must** match.
- `preferred...`: scheduler **prefers** a match but may choose another eligible node.
- `IgnoredDuringExecution`: changing a label later does not automatically evict an already running Pod.
- `nodeSelectorTerms` are ORed; expressions within one term are ANDed.

## 5. Pod Affinity and Anti-Affinity
These match **labels on other Pods**, not labels on nodes. `topologyKey` defines the boundary.

### Pod affinity: place near matching Pods
```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: backend
          topologyKey: kubernetes.io/hostname
```

### Pod anti-affinity: avoid matching Pods
```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: web
          topologyKey: kubernetes.io/hostname
```

- `kubernetes.io/hostname`: same/different **node**.
- `topology.kubernetes.io/zone`: same/different **zone**.
- Required anti-affinity may leave replicas Pending if insufficient eligible nodes exist.

## 6. Taints and Tolerations
**Taints** repel Pods from nodes; **tolerations** allow Pods to tolerate matching taints. A toleration does **not** attract a Pod to that node.

```bash
kubectl taint nodes <node-name> color=red:NoSchedule
kubectl describe node <node-name>
kubectl taint nodes <node-name> color=red:NoSchedule-
```

```yaml
spec:
  tolerations:
    - key: color
      operator: Equal
      value: red
      effect: NoSchedule
```

| Taint effect | Behavior |
|---|---|
| `NoSchedule` | Prevents new non-tolerating Pods from scheduling |
| `PreferNoSchedule` | Soft preference to avoid the node |
| `NoExecute` | Prevents placement and evicts existing non-tolerating Pods |

With `NoExecute`, `tolerationSeconds: 60` allows a matching Pod to remain for up to 60 seconds after the taint applies.

## 7. Combining Affinity and Tolerations
Use **required node affinity** to target a labeled node and a **toleration** to permit entry if that node is tainted.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: red-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: color
                operator: In
                values: [red]
  tolerations:
    - key: color
      operator: Equal
      value: red
      effect: NoSchedule
  containers:
    - name: nginx
      image: nginx:stable
```

**Field placement:** `affinity`, `tolerations`, and `containers` are all under the Pod's `spec`.

## 8. Kubernetes vs EKS
- **Kubernetes-native:** scheduler, labels, affinity, taints, tolerations.
- **EKS/AWS:** worker nodes have zone labels such as `topology.kubernetes.io/zone=ap-south-1a`. EBS volumes are AZ-scoped; the Pod and volume must be in compatible zones.

## 9. Important Commands
```bash
kubectl get pods -o wide
kubectl get nodes --show-labels
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get nodes -o custom-columns='NAME:.metadata.name,TAINTS:.spec.taints'
kubectl describe pod <pod-name>
kubectl describe node <node-name>
```

## 10. Troubleshooting
**Pod Pending?** Start with `kubectl describe pod <pod-name>` and inspect Events.

| Event clue | Check |
|---|---|
| `didn't match Pod's node affinity/selector` | Node labels and affinity rules |
| `untolerated taint` | Node taints and Pod tolerations |
| `Insufficient cpu` / `memory` | Resource requests vs allocatable capacity |
| `volume node affinity conflict` | PV topology vs node zone |

Avoid changing constraints blindly; identify which scheduler requirement is failing first.

## 11. Interview Questions
1. **nodeSelector vs node affinity?** Exact matches vs richer required/preferred label expressions.
2. **Node affinity vs Pod affinity?** Node labels vs labels of existing Pods.
3. **Does toleration force placement?** No; it only removes a taint-based restriction.
4. **`NoSchedule` vs `NoExecute`?** New scheduling restriction vs restriction plus possible eviction.
5. **What does `IgnoredDuringExecution` mean?** Label changes do not automatically evict the running Pod.

## 12. Quick Revision
- **Affinity:** attract/require placement.
- **Anti-affinity:** separate Pods.
- **Taint:** repel Pods.
- **Toleration:** permit despite taint.
- **Required:** hard rule; **Preferred:** soft rule.
- **Pending Pod:** check `kubectl describe pod` Events first.
