# Services and Kubernetes DNS

Tags: #kubernetes #service #dns #networking #sre #cka

## Concept
A Service gives Pods a stable way to reach an application, even when backend Pods change.

## Definition
A Service uses a selector to find backend Pods. Kubernetes provides a stable Service address and routes traffic to eligible endpoints.

## How It Works
```text
Client Pod -> Service DNS / ClusterIP -> selected ready Pod(s)
```

### ClusterIP
Used for communication **inside the cluster**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cluster-ip
  namespace: dev
spec:
  type: ClusterIP
  selector:
    group: web
  ports:
    - port: 80
      targetPort: 80
```

- `port`: Port clients use on the Service.
- `targetPort`: Port on the backend Pod.
- `selector`: Labels used to find backend Pods.

### NodePort
Makes a Service reachable through a port on a node, subject to networking and firewall rules.

Example from your practice:
```text
Service port: 80
NodePort:     30001
Node IP:      192.168.49.2
Test URL:     http://192.168.49.2:30001
```

On Minikube, direct node-IP access may depend on the driver and host networking. Port-forwarding is a useful alternative.

### Service FQDN
Standard format:
```text
<service>.<namespace>.svc.<cluster-domain>
```

Your example:
```text
cluster-ip.dev.svc.cluster.local
```

This assumes the common cluster domain `cluster.local`. From another Pod in `dev`, `cluster-ip` often works as a short name.

## SRE Relevance
A Service can exist while having no usable backends. Check the selector, Pod readiness, endpoint addresses, and target port.

## Commands
```bash
kubectl get svc -n dev
kubectl describe svc cluster-ip -n dev
kubectl get endpoints cluster-ip -n dev
kubectl get endpointslices -n dev -l kubernetes.io/service-name=cluster-ip
kubectl get pods -n dev --show-labels
kubectl port-forward -n dev svc/cluster-ip 8080:80
# In another terminal:
curl http://localhost:8080
```

DNS test using a temporary Pod:
```bash
kubectl run dns-test -n dev --rm -it --restart=Never   --image=busybox:1.36 --   nslookup cluster-ip.dev.svc.cluster.local
```

## Quick Revision
- ClusterIP = internal Service address.
- NodePort = Service exposed through a node port.
- Service selects Pods using labels.
- `port` = Service port; `targetPort` = backend port.
- FQDN = `service.namespace.svc.cluster.local` in a default-domain cluster.
- Service DNS is normally for use inside the cluster.
