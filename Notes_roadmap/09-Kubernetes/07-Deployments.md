# Deployments and Rolling Updates

Tags: #kubernetes #deployment #rollout #sre #cka

## Concept
A Deployment keeps application Pods running and lets us update them with controlled rollouts.

## Definition — interview answer
A Deployment manages ReplicaSets. When its Pod template changes, it can create a new ReplicaSet and gradually shift Pods from the old version to the new one. It supports rollout history and rollback.

## Why We Need It
Directly changing standalone Pods is hard to manage. Deployments make replica count, application updates, and rollback repeatable.

## How It Works
```text
Deployment (3 replicas)
  old ReplicaSet: 3 old Pods
       update image
  new ReplicaSet: new Pods start
  old ReplicaSet: old Pods scale down
```
A rolling update tries to maintain availability, but a bad app or failed readiness can still cause problems.

## Working YAML
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sp-deployment
  namespace: dev
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: sp-web
  template:
    metadata:
      labels:
        app: sp-web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```
- `replicas`: desired Pods.
- `selector` and `template.labels`: must match.
- `maxSurge: 1`: can temporarily create one extra Pod during update.
- `maxUnavailable: 0`: do not intentionally reduce available Pods below desired count during rollout.
- `template.spec.containers.image`: changing this starts a new Pod-template rollout.

## What to Learn
```bash
kubectl apply -f deployment.yaml
kubectl get deploy,rs,pods -n dev
kubectl set image deployment/sp-deployment nginx=nginx:1.28 -n dev
kubectl rollout status deployment/sp-deployment -n dev
kubectl rollout history deployment/sp-deployment -n dev
kubectl rollout undo deployment/sp-deployment -n dev
```
Image tags shown are examples; choose images available to your cluster. For your actual `production` Deployment, use `-n production` and verify the name before rollback.

## Real-World Example
You watched old Pod names with one ReplicaSet hash terminate while new Pods with another hash started. That is a rolling update. If new Pods fail readiness, check rollout status, new Pod events, and logs before deciding to roll back.

## Troubleshooting
**Rollout stuck:**
```bash
kubectl rollout status deployment/sp-deployment -n dev
kubectl get pods -n dev -l app=sp-web
kubectl describe deployment sp-deployment -n dev
kubectl describe pod <new-pod> -n dev
kubectl logs <new-pod> -n dev
```
Look for image pull errors, readiness failures, and capacity problems.

**Rollback says NotFound:**
```bash
kubectl get deployments -n production
# Correct spelling matters:
kubectl rollout undo deployment/sp-deployment -n production
```
Rollback needs an earlier revision; check `rollout history` first.

## Interview Questions
**Q: Why does Deployment create a new ReplicaSet?**  
A: A change to the Pod template represents a new version that can be rolled out gradually.

**Q: Does scaling from 3 to 5 create a new ReplicaSet?**  
A: Usually no. It scales the active ReplicaSet(s) without changing the Pod template.

**Q: What are maxSurge and maxUnavailable?**  
A: They control how many extra Pods can exist and how many desired Pods can be unavailable during a rolling update.

**Q: How do you roll back a bad release?**  
A: Check the Deployment and revision history, run `kubectl rollout undo`, then verify rollout status and application health.

## Quick Revision
- Deployment -> ReplicaSets -> Pods.
- Pod-template change -> new rollout.
- Replica count change -> scaling.
- Check `rollout status`, events, logs; rollback only after confirming target.
