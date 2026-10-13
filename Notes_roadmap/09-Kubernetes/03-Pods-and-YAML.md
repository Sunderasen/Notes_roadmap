# Pods and Pod YAML

Tags: #kubernetes #pods #yaml #sre #cka

## Concept
A Pod is the smallest unit Kubernetes schedules.

## Definition
A Pod holds one or more containers that share a network identity and can share storage.

## How It Works
One Pod normally runs one main application container. Containers in the same Pod share the Pod IP and can reach each other through `localhost`.

## YAML Example
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

| Field | Remember |
|---|---|
| `apiVersion` | Kubernetes API version |
| `kind` | Resource type |
| `metadata.name` | Pod name |
| `metadata.namespace` | Where the Pod lives |
| `metadata.labels` | Tags used to find the Pod |
| `spec.containers` | Containers to run |
| `name` | Container name |
| `image` | Container image |
| `containerPort` | Port declared for the container; does not expose it outside the Pod |

## Pod Lifecycle
- **Pending:** Pod accepted but not fully running yet; may be waiting for scheduling, image, or setup.
- **Running:** Pod is assigned to a node and at least one container is running, starting, or restarting.
- **Succeeded:** All containers finished successfully and will not restart.
- **Failed:** All containers ended, and at least one failed.
- **Unknown:** Kubernetes cannot determine the Pod phase.

`CrashLoopBackOff` and `ImagePullBackOff` are **kubectl status reasons**, not Pod phases. **Running does not always mean Ready.**

## SRE Relevance
A Pod can be Running but fail readiness checks. Always check `READY`, events, and logs.

## Commands
```bash
kubectl apply -f pod.yaml
kubectl get pods -n dev -o wide
kubectl describe pod httpd -n dev
kubectl logs httpd -n dev
kubectl logs -f httpd -n dev
kubectl logs httpd -n dev --previous
kubectl exec -it httpd -n dev -- /bin/sh
kubectl delete pod httpd -n dev
```

**Note:** A standalone Pod is not recreated automatically after deletion unless another controller manages it.

## Quick Revision
- Pod name and container name are different fields.
- Pod = scheduled unit; container = running application.
- Check `READY`, not only `STATUS`.
- `kubectl logs` reads container stdout/stderr.
- `--previous` helps after a container restart.
