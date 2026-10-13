# Services, NodePort, LoadBalancer and DNS

Tags: #kubernetes #service #networking #dns #sre #cka

## Concept
A Service gives clients a stable way to reach changing Pods.

## Definition — interview answer
A Kubernetes Service is a stable network endpoint for a set of backend Pods. A selector typically finds the Pods; EndpointSlices track usable backend addresses. The Service can be internal (`ClusterIP`) or exposed using `NodePort` or a `LoadBalancer` integration.

## Why We Need It
Pod IPs change when Pods are replaced. A Service name and ClusterIP can stay stable while backends change.

## How It Works
```text
Client -> cluster-ip.dev.svc.cluster.local:80
               |
         ClusterIP Service
               |
        EndpointSlice entries
               |
         Ready matching Pods
```
**Service selects Pods in its own namespace.** A Service does not fix a broken backend app.

## Working YAML — ClusterIP
Assume your Nginx Pod has label `app: nginx` in `dev`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: cluster-ip
  namespace: dev
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
```
- `port`: port used by Service clients.
- `targetPort`: destination port on Pod.
- `selector`: chooses matching Pods.
- `containerPort` in a Pod manifest is descriptive; the app must actually listen on the target port.

## NodePort
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
  namespace: dev
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30001
```
Test from a machine that can reach the node:
```bash
kubectl get nodes -o wide
curl http://<node-ip>:30001
```
Your Minikube node IP was `192.168.49.2`; its reachability depends on the Minikube driver/network setup. NodePort usually uses port range `30000–32767`, unless the cluster is configured differently.

## LoadBalancer and NodePort
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
  namespace: dev
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```
By default, Kubernetes normally allocates a NodePort for a LoadBalancer Service. `kubectl get svc -n dev` may show `80:31xxx/TCP`.

**Can you curl the NodePort?** Yes, if a NodePort was allocated, the node address is reachable, the network allows it, and the Service has working backends. Some load-balancer implementations route directly to Pods rather than through NodePort.

A LoadBalancer Service can use `spec.allocateLoadBalancerNodePorts: false` to avoid automatic NodePort allocation if the load-balancer implementation supports direct routing. Existing allocated NodePorts are not automatically removed just by changing this field.

On Minikube, an `EXTERNAL-IP` may remain Pending until you use a supported load-balancer solution such as `minikube tunnel`. NodePort can still be tested separately.

## Kubernetes DNS / FQDN
Standard Service DNS name:
```text
<service>.<namespace>.svc.<cluster-domain>
```
Your Service (with default cluster domain):
```text
cluster-ip.dev.svc.cluster.local
```
Inside the `dev` namespace, `cluster-ip` usually works as a short name. This internal DNS name normally does **not** resolve from your laptop/host without extra DNS setup.

## What to Learn
```bash
kubectl get svc -n dev
kubectl describe svc cluster-ip -n dev
kubectl get pods -n dev --show-labels
kubectl get endpointslices -n dev -l kubernetes.io/service-name=cluster-ip
kubectl port-forward svc/cluster-ip -n dev 8080:80
# In another terminal:
curl http://localhost:8080
```
DNS check from a temporary Pod:
```bash
kubectl run dns-test -n dev --rm -it --restart=Never   --image=busybox:1.36 -- nslookup cluster-ip.dev.svc.cluster.local
```

## Real-World Example
You gave a Service this selector:
```yaml
selector:
  app: nginx
  app: httpd
```
It selected only one backend in your test. This is not a valid way to select both. Give both Pods `group: web` and select `group: web`. In real production, separate Nginx and Apache Services are often safer because they may return different content.

## Troubleshooting
**Service returns connection errors:**
```bash
kubectl describe svc cluster-ip -n dev
kubectl get pods -n dev --show-labels
kubectl get endpointslices -n dev -l kubernetes.io/service-name=cluster-ip
kubectl describe pod <backend-pod> -n dev
```
Check selector -> readiness -> targetPort -> app logs -> network path, in that order.

**NodePort fails from host:** Confirm node IP, NodePort, endpoints, firewall, and Minikube networking. Try `minikube service web-nodeport -n dev --url` if available, or `kubectl port-forward`.

**DNS fails:** Verify Service name/namespace, test from a Pod, and inspect DNS configuration (`/etc/resolv.conf`) before blaming the app.

## Interview Questions
**Q: Why use Service instead of Pod IP?**  
A: Pod IPs change; the Service gives a stable endpoint.

**Q: ClusterIP vs NodePort vs LoadBalancer?**  
A: ClusterIP is internal; NodePort opens a port on nodes; LoadBalancer integrates with an external/internal load balancer implementation.

**Q: Does LoadBalancer create NodePort?**  
A: By default, yes; it can be disabled when the implementation supports it.

**Q: What is your Service FQDN?**  
A: `cluster-ip.dev.svc.cluster.local` if the cluster domain is `cluster.local`.

**Q: Service exists but no traffic—what next?**  
A: Check labels/selectors, EndpointSlices, Pod readiness, ports, then app/network logs.

## Quick Revision
- Service = stable address; Pods = replaceable backends.
- `port` is Service port; `targetPort` is Pod destination port.
- Default FQDN = `service.namespace.svc.cluster.local`.
- LoadBalancer usually gets a NodePort, but network reachability still matters.
- Check EndpointSlices when a Service has no working backends.
