# Kubernetes Troubleshooting

Tags: #kubernetes #troubleshooting #sre

Keep adding new problems here after each practice session.

## Pod exists but `kubectl exec` says NotFound

**Problem**
```bash
kubectl exec -it httpd -- /bin/bash -n dev
# Error: pods "httpd" not found
```

**Why it happened**
`-n dev` was placed after `--`. Kubernetes searched the **default** namespace. Everything after `--` is treated as the command inside the container.

**Check**
```bash
kubectl get pods -n dev
```

**Fix**
```bash
kubectl exec -it httpd -n dev -- /bin/sh
# Or use /bin/bash if installed
```

**Remember:** kubectl options go before `--`.

---

## Service reaches only HTTPD, not Nginx

**Problem**
```yaml
selector:
  app: nginx
  app: httpd
```

**Why it happened**
The `app` key is duplicated. This is invalid YAML mapping structure; a parser may reject it or keep one value.

**Check**
```bash
kubectl describe svc cluster-ip -n dev
kubectl get pods -n dev --show-labels
kubectl get endpoints cluster-ip -n dev
```

**Fix**
Give both Pods a common label, such as `group: web`, then use:
```yaml
selector:
  group: web
```

**Remember:** Multiple different selector keys use AND logic. A Service does not support OR matching with two values of the same key.

---

## ReplicaSet YAML rejected: unknown field

**Problem**
```yaml
port:
  - containerport: 80
```

**Why it happened**
Kubernetes YAML field names are exact and case-sensitive.

**Fix**
```yaml
ports:
  - containerPort: 80
```

**Check**
```bash
kubectl explain pod.spec.containers.ports
kubectl apply --dry-run=server -f replicaset.yaml
```

**Remember:** `ports` is a list; `containerPort` uses capital `P`.

---

## Rollback says Deployment NotFound

**Problem**
```bash
kubectl rollout undo deployment/sp-depolyment -n production
```

**Why it happened**
The resource name was misspelled, or the namespace was wrong.

**Check**
```bash
kubectl get deployments -n production
```

**Fix**
```bash
kubectl rollout undo deployment/sp-deployment -n production
```

**Remember:** Copy exact names from `kubectl get`.

---

## Service exists but requests fail

**Possible causes**
- Service selector does not match Pod labels.
- Backend Pod is not Ready.
- `targetPort` does not match the listening application port.
- Network access to NodePort is blocked or unavailable.

**Check**
```bash
kubectl get svc -n dev
kubectl describe svc cluster-ip -n dev
kubectl get pods -n dev --show-labels
kubectl get endpointslices -n dev -l kubernetes.io/service-name=cluster-ip
kubectl describe pod <pod-name> -n dev
```

**Fix**
Correct the selector, restore backend health, or fix port/network settings based on the evidence.

**Remember:** Service exists ≠ healthy backend exists.

---

## Pod not starting

**Check**
```bash
kubectl get pods -n dev
kubectl describe pod <pod-name> -n dev
kubectl logs <pod-name> -n dev
kubectl logs <pod-name> -n dev --previous
```

**Look for**
- `Pending`: scheduling, image pull, or startup work may still be in progress.
- `ImagePullBackOff`: image download is failing.
- `CrashLoopBackOff`: a container keeps failing and restarting.

**Remember:** Events explain many startup problems; logs explain what the application printed.
