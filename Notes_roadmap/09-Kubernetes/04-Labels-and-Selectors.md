# Labels and Selectors

Tags: #kubernetes #labels #selectors #sre

## Concept
Labels identify resources. Selectors find matching resources.

## Definition
A label is a key/value pair, such as `app: nginx`. A selector is a rule used by Services and controllers to choose resources.

## How It Works
Pod:
```yaml
metadata:
  labels:
    app: nginx
    group: web
```

Service:
```yaml
spec:
  selector:
    group: web
```

The Service selects Pods with `group: web`.

## Example
You tried:
```yaml
selector:
  app: nginx
  app: httpd
```

This repeats the same YAML key. It is invalid; a parser may reject it or keep only one value.

To select both Pods, give each a common label:
```yaml
# Nginx Pod
labels:
  app: nginx
  group: web

# HTTPD Pod
labels:
  app: httpd
  group: web

# Service
selector:
  group: web
```

**Multiple different selector keys mean AND**, not OR.

## SRE Relevance
A wrong Service selector can send traffic to the wrong Pods or leave the Service with no endpoints.

## Commands
```bash
kubectl get pods -n dev --show-labels
kubectl get pods -n dev -l group=web
kubectl describe svc cluster-ip -n dev
kubectl get endpoints cluster-ip -n dev
```

## Quick Revision
- Labels = identify.
- Selectors = match.
- `app: nginx` and `app: httpd` cannot both be values of one `app` key.
- Service selectors match all listed key/value pairs.
- Check Pod labels and Service endpoints together.
