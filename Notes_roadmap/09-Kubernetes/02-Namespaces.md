# Kubernetes Namespaces

Tags: #kubernetes #namespace #sre

## Concept
A namespace is a way to organize Kubernetes resources.

## Definition
Namespaces let teams or environments use the same cluster without mixing resource names.

## How It Works
A Pod called `nginx` can exist in both `dev` and `production`.

```text
Cluster
  dev -> nginx Pod
  production -> nginx Pod
```

Namespaces do **not** automatically block network traffic between Pods.

## Example
Your `httpd` and `nginx` Pods were created in `dev`. A command without `-n dev` searched the default namespace and returned `NotFound`.

## SRE Relevance
Always check the namespace before changing or debugging production resources.

## Commands
```bash
kubectl get namespaces
kubectl get pods -n dev
kubectl get pods -A
kubectl get pods --all-namespaces
kubectl config view --minify
```

## Quick Revision
- Namespace = resource grouping.
- `-n dev` = work in `dev`.
- `-A` = list across namespaces.
- Same Pod name is allowed in different namespaces.
- Namespace is not automatic network isolation.
