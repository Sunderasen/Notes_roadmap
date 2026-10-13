# Pods and Pod YAML

Tags: #kubernetes #pod #yaml #sre #cka

## Concept
A Pod is the smallest unit Kubernetes schedules. Usually one Pod contains one application container.

## Definition — interview answer
A Pod wraps one or more containers that share the Pod network (including one Pod IP and localhost) and can share volumes. Kubernetes schedules the **Pod**, not each container separately.

## Why We Need It
Kubernetes needs a unit to schedule, restart containers in, and attach network/storage settings to. Pods are replaceable; do not rely on a Pod IP staying the same.

## How It Works
```text
Node
  Pod: httpd (Pod IP 10.244.x.y)
    Container: httpd -> listens on port 80
```
A Pod's `Running` phase does **not** mean its application is ready for traffic.

## Working YAML
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: httpd
  namespace: dev
  labels:
    app: httpd
spec:
  containers:
    - name: httpd
      image: httpd:2.4
      ports:
        - containerPort: 80
```

| Field | What it means |
|---|---|
| `apiVersion: v1` | API group/version for Pod |
| `kind: Pod` | Create a Pod |
| `metadata.name` | Pod's name |
| `metadata.namespace` | Pod's namespace |
| `metadata.labels` | Searchable key/value tags |
| `spec.containers` | List of containers in Pod |
| `containers[].name` | Container name (not Pod name) |
| `image` | Image to run; pinned version is clearer than `latest` |
| `containerPort` | Describes the container port; does not publish it |

## Pod Phases vs Status Reasons
| Pod phase | Meaning |
|---|---|
| Pending | Accepted but not fully started |
| Running | Assigned to a node; container is running, starting, or restarting |
| Succeeded | All containers finished successfully |
| Failed | All containers stopped; at least one failed |
| Unknown | State cannot be determined |

`CrashLoopBackOff` and `ImagePullBackOff` are **status reasons shown by kubectl**, not Pod phases. A container may restart while the Pod object remains.

## What to Learn
```bash
kubectl apply -f httpd.yaml
kubectl get pods -n dev -o wide
kubectl describe pod httpd -n dev
kubectl logs httpd -n dev
kubectl logs -f httpd -n dev
kubectl logs httpd -n dev --previous
kubectl exec -it httpd -n dev -- /bin/sh
```
For multi-container Pods use `-c <container-name>` with `logs` or `exec`.

## Real-World Example
An API Pod says `Running`, but users get 503. Check `READY` and events, then the application logs. A process can be running while failing its readiness check or unable to serve requests.

## Troubleshooting
**Pod keeps restarting:**
```bash
kubectl get pod httpd -n dev
kubectl describe pod httpd -n dev
kubectl logs httpd -n dev --previous
```
Look for application errors, OOMKilled, and probe failures. `--previous` shows the last terminated container's logs if available.

**No logs?** `kubectl logs` reads stdout/stderr. An app writing only to a file may not appear there.

**Pod deleted and not recreated?** Standalone Pods have no ReplicaSet/Deployment to replace them.

## Interview Questions
**Q: Pod vs container?**  
A: A container runs the process. A Pod is the Kubernetes unit that groups and schedules container(s).

**Q: Can two containers in a Pod use localhost?**  
A: Yes, they share the Pod network namespace.

**Q: Does `containerPort: 80` expose the Pod outside the cluster?**  
A: No. Use a Service or another supported network path.

**Q: Difference between restart and replacement?**  
A: Kubelet can restart a container inside the same Pod; a controller creates a new Pod when a managed Pod disappears.

**Q: What do you check first in CrashLoopBackOff?**  
A: Describe the Pod and read current/previous container logs.

## Quick Revision
- Pod IP can change; Service gives a stable endpoint.
- Running ≠ Ready.
- Pod name and container name are separate.
- Logs + events explain most first-level Pod failures.
