# Deployments and Rolling Updates

Tags: #kubernetes #deployment #rollout #sre #cka

## Concept
A Deployment manages application updates and ReplicaSets.

## Definition
A Deployment describes the desired application Pods and can update them gradually.

## How It Works
```text
Deployment
   |-- Old ReplicaSet -> old Pods
   |-- New ReplicaSet -> new Pods
```

During a rolling update, Kubernetes gradually starts Pods with the new template and removes old ones.

## Example
You saw Pod names change from one ReplicaSet hash to another during a rollout. That is expected: a Deployment created a new ReplicaSet for the updated Pod template.

**Important:** Updating a Deployment's Pod template can create a new ReplicaSet. Changing only the replica count usually scales the existing active ReplicaSet.

## SRE Relevance
During a bad release, check rollout status and undo the release when appropriate. Verify the application after rollback.

## Commands
```bash
kubectl get deployments -n production
kubectl get rs -n production
kubectl rollout status deployment/sp-deployment -n production
kubectl rollout history deployment/sp-deployment -n production
kubectl rollout undo deployment/sp-deployment -n production
kubectl describe deployment sp-deployment -n production
```

For the default RollingUpdate settings, `maxSurge` and `maxUnavailable` are 25% each. Their actual Pod counts depend on rounding and the replica count.

## Quick Revision
- Deployment manages ReplicaSets.
- Rolling update replaces Pods gradually.
- New Pod template can mean new ReplicaSet.
- `rollout status` checks progress.
- `rollout undo` rolls back to the previous revision when available.
- Always check spelling and namespace (`sp-deployment`, not `sp-depolyment`).
