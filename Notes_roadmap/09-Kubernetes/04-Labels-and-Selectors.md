# Labels and Selectors

Tags: #kubernetes #labels #selector #sre #cka

## Concept
Labels are tags on objects. Selectors find objects with those tags.

## Definition — interview answer
A label is a key/value pair in metadata. Controllers and Services use selectors to choose the Pods they manage or send traffic to.

## Why We Need It
Pod names change when replicas are replaced. Labels let a Service find the right Pods without depending on names or IPs.

## How It Works
```text
nginx Pod: app=nginx, group=web ----+
                                    +--> Service selector: group=web
httpd Pod: app=httpd, group=web ----+
```
A Service selector with multiple different keys uses **AND**: every listed label must match.

## Working YAML
Add a shared label to both Pods:
```yaml
# Nginx Pod metadata
metadata:
  name: nginx
  namespace: dev
  labels:
    app: nginx
    group: web
```
```yaml
# HTTPD Pod metadata
metadata:
  name: httpd
  namespace: dev
  labels:
    app: httpd
    group: web
```
Service selector:
```yaml
spec:
  selector:
    group: web
```
These are **snippets**, not standalone manifests. See [[08-Services-and-DNS]] for a complete Service manifest.

**Wrong:**
```yaml
selector:
  app: nginx
  app: httpd
```
This duplicates a YAML key. It does not mean OR. Depending on validation, the manifest can be rejected or one value can replace the other.

## What to Learn
```bash
kubectl get pods -n dev --show-labels
kubectl get pods -n dev -l group=web
kubectl describe svc cluster-ip -n dev
kubectl get endpointslices -n dev -l kubernetes.io/service-name=cluster-ip
```

## Real-World Example
You expected one Service to send traffic to both Nginx and Apache. Only HTTPD responded because your selector repeated `app`. A shared `group: web` label fixes selection. **Production caution:** Mixing different applications behind one Service can give users different responses; normally keep separate Services unless behavior is intentionally identical.

## Troubleshooting
**Service has no endpoints:**
1. Check actual Service selector: `kubectl describe svc cluster-ip -n dev`.
2. Check Pod labels: `kubectl get pods -n dev --show-labels`.
3. Check Pods are Ready: `kubectl get pods -n dev`.
4. Check EndpointSlices: `kubectl get endpointslices -n dev -l kubernetes.io/service-name=cluster-ip`.

## Interview Questions
**Q: Labels vs selectors?**  
A: Labels describe resources; selectors match resources.

**Q: Do two selector keys mean OR?**  
A: No, they mean AND for normal Service selectors.

**Q: Can a Service select Pods from another namespace?**  
A: A normal selector-based Service selects Pods in its own namespace.

**Q: Why not select Pods by name?**  
A: Replacement Pod names can change; labels stay consistent through the template.

## Quick Revision
- Label = identify; selector = find.
- Duplicate YAML keys are not OR conditions.
- Service selector + Pod labels + readiness = backend selection.
