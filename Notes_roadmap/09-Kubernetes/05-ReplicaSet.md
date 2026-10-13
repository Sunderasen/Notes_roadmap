# ReplicaSet

Tags: #kubernetes #replicaset #sre #cka

## Concept
A ReplicaSet keeps a chosen number of matching Pods running.

## Definition
It compares the desired replica count with the number of matching Pods and creates or removes Pods as needed.

## How It Works
```text
ReplicaSet (replicas: 3)
   |-- Pod A
   |-- Pod B
   |-- Pod C
```
If a managed Pod is deleted, the ReplicaSet creates a replacement.

## YAML Example
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: sp-replica
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        app: nginx
        tier: frontend
    spec:
      containers:
        - name: cont1
          image: httpd:2.4
          ports:
            - containerPort: 80
```

- `metadata.name`: ReplicaSet name.
- `spec.replicas`: Number of Pods wanted.
- `spec.selector.matchLabels`: Which Pods belong to this ReplicaSet.
- `spec.template`: Blueprint for Pods it creates.
- `template.metadata.labels`: Labels on created Pods; must match the selector.
- `template.spec.containers`: Containers in each Pod.

**Naming tip:** `app: nginx` is allowed here, but confusing because the image is `httpd`. Prefer labels that describe the real workload.

## SRE Relevance
If a Pod disappears, the ReplicaSet can restore the count. A ReplicaSet does not guarantee the application is healthy.

## Commands
```bash
kubectl apply -f replicaset.yaml
kubectl get rs
kubectl get pods --show-labels
kubectl describe rs sp-replica
kubectl scale rs sp-replica --replicas=5
kubectl explain replicaset.spec.template.spec.containers
```

Changing only `replicas: 3` to `replicas: 5` scales the **same ReplicaSet**; it does not create a new ReplicaSet.

## Quick Revision
- ReplicaSet = keeps Pod count.
- Selector must match template labels.
- Template = new Pod blueprint.
- Scaling changes Pod count.
- Deployments normally manage ReplicaSets in production.
