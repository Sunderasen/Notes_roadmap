# Namespaces

Tags: #kubernetes #namespace #sre #cka

## Concept
A namespace is a folder-like boundary for Kubernetes resource names and organization.

## Definition — interview answer
A namespace groups namespaced resources within one cluster. It lets `dev` and `production` each have a Pod named `nginx`. Namespaces alone do **not** isolate network traffic or guarantee resource limits.

## Why We Need It
Teams can share a cluster without mixing up similarly named resources. Namespace-aware commands reduce the risk of changing the wrong environment.

## How It Works
```text
Cluster
  dev        -> nginx, httpd, cluster-ip Service
  production -> nginx, payment-api
```
A Pod can be named `nginx` in both namespaces. Nodes are cluster-scoped; they do not live inside a namespace.

## Working YAML
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.27
```
Save as `namespace-demo.yaml`, then:
```bash
kubectl apply -f namespace-demo.yaml
kubectl get pods -n dev
```
Do not delete the namespace as cleanup if it contains other practice resources.

## What to Learn
```bash
kubectl get ns
kubectl get pods -n dev
kubectl get pods -A
kubectl config current-context
kubectl config set-context --current --namespace=dev
```
The last command changes the **default namespace in your current context**. Explicit `-n` still overrides it. Check your context before using it.

## Real-World Example
You ran:
```bash
kubectl exec -it httpd -- /bin/bash -n dev
```
Kubernetes searched for `httpd` in the default namespace because `-n dev` came **after** `--`. Correct:
```bash
kubectl exec -it httpd -n dev -- /bin/sh
```
`--` separates kubectl options from the command inside the container.

## Troubleshooting
**Symptom: `pods "httpd" not found`, but `get pods -n dev` shows it.**
```bash
kubectl get pods -A | grep httpd
kubectl exec -n dev httpd -- hostname
```
Fix the namespace. If the shell fails with `executable file not found`, try `/bin/sh`; minimal images may not include Bash.

## Interview Questions
**Q: Can two namespaces have a Pod with the same name?**  
A: Yes. The namespace is part of the resource identity.

**Q: Does a namespace block traffic between Pods?**  
A: No, not by itself. Network isolation needs a supported NetworkPolicy setup.

**Q: Why use `-n` in production commands?**  
A: It makes the target environment explicit and reduces mistakes.

**Q: Are nodes namespaced?**  
A: No. Nodes are cluster-scoped.

## Quick Revision
- Namespace = grouping and naming boundary.
- `-n dev` = choose namespace; `-A` = all namespaces.
- Always verify context + namespace before changes.
- `--` means the remaining arguments belong to the container command.
