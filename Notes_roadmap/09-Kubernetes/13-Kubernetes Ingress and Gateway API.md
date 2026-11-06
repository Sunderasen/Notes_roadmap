
**Category:** Kubernetes Networking  
**Environment:** Kind / Minikube / AWS EKS  
**Tags:** #kubernetes #ingress #gateway-api #networking #sre

---

# 1. Concept

Kubernetes Ingress and Gateway API manage **incoming traffic to applications running inside a Kubernetes cluster**.

Both allow routing requests based on information such as:

- Domain name (hostname)
    
- URL path
    
- HTTP request properties
    

They sit logically between external clients and Kubernetes Services.

```
Client
   |
   v
External IP / Load Balancer
   |
   v
Ingress Controller / Gateway Implementation
   |
   v
Kubernetes Service
   |
   v
Application Pods
```

**Important:** Ingress and Gateway API are configuration APIs. They do not forward traffic themselves. A compatible controller must implement their rules.

---

# 2. Definition

## Ingress

Ingress is a Kubernetes API resource that defines HTTP/HTTPS routing rules for incoming traffic.

An **Ingress Controller** reads those rules and configures the actual routing infrastructure.

## Gateway API

Gateway API is a collection of Kubernetes networking APIs designed for more flexible traffic management.

Its main resources are:

|Resource|Responsibility|
|---|---|
|`GatewayClass`|Identifies the Gateway controller implementation|
|`Gateway`|Defines listeners, ports, protocols, and TLS configuration|
|`HTTPRoute`|Defines HTTP request matching and routing rules|
|`GRPCRoute`|Defines gRPC routing|
|`TCPRoute` / `TLSRoute`|Define additional routing types where supported|

Gateway API uses Custom Resource Definitions (CRDs), unlike the built-in Kubernetes Ingress API.

---

# 3. Why It Is Needed

## Why not use only a LoadBalancer Service?

A `LoadBalancer` Service exposes a Kubernetes Service through an external load balancer.

However, a Service does not define HTTP routing rules based on hostname, URL path, or HTTP headers.

Ingress and Gateway API provide these routing capabilities.

For example:

```
example.com/       → frontend-service
example.com/api    → api-service
admin.example.com  → admin-service
```

The same entry point can serve multiple applications.

## Why was Gateway API introduced?

Ingress has limitations:

- Many advanced features require controller-specific annotations.
    
- Complex traffic management is not standardized.
    
- Infrastructure and application routing responsibilities are less clearly separated.
    
- The API is relatively limited to HTTP/HTTPS routing.
    

Gateway API provides standardized resources for richer routing and clearer ownership.

**Note:** Ingress remains supported, but its API is frozen and is not receiving new features.

---

# 4. How It Works

## 4.1 Ingress request flow

```
Client
  |
  | GET /api/users
  | Host: example.com
  v
Ingress Controller
  |
  | Matches hostname + path
  v
api-service
  |
  v
API Pod
```

Steps:

1. Client resolves the domain to the ingress entry point.
    
2. Client sends an HTTP request.
    
3. The Ingress controller checks the hostname and URL path.
    
4. The matching rule selects a Kubernetes Service.
    
5. Traffic is forwarded to an eligible backend Pod.
    

## 4.2 Gateway API request flow

```
Client
  |
  v
Gateway Listener
  |
  v
HTTPRoute
  |
  | Match hostname / path / headers
  | Apply supported filters
  v
Kubernetes Service
  |
  v
Application Pod
```

Steps:

1. `GatewayClass` identifies the implementation.
    
2. `Gateway` defines where traffic enters.
    
3. `HTTPRoute` attaches to a Gateway using `parentRefs`.
    
4. The route matches incoming requests.
    
5. Matching traffic is forwarded to the backend Service.
    

## 4.3 Understanding the Host header

The HTTP `Host` header tells the server which hostname the client wants to access.

Example:

```
curl -H "Host: example.com" http://127.0.0.1:8080/api
```

The TCP connection goes to `127.0.0.1:8080`, but the HTTP request identifies `example.com` as its target hostname.

This allows the controller to match hostname-based routing rules.

**DNS and Host are different:**

- DNS resolves a hostname to an IP address.
    
- The HTTP Host header identifies the requested hostname.
    
- The controller uses the Host header (or HTTP/2 `:authority`) for hostname-based routing.
    

Adding a local hosts-file entry can change where a domain resolves, but it does not create routing rules inside Kubernetes.

For HTTPS, TLS SNI and certificate validation also matter.

## 4.4 URL path matching vs URL rewriting

These are different operations.

**Path matching:**

```
Client requests /api/users
Route matches /api
Backend receives /api/users
```

**URL rewriting:**

```
Client requests /api/users
Route matches /api
Rewrite removes /api prefix
Backend receives /users
```

A routing rule does not automatically rewrite the request URL.

Gateway API provides an `URLRewrite` filter for supported implementations.

---

# 5. YAML / Configuration

## 5.1 Ingress YAML

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080

          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

### Important fields

|Field|Meaning|
|---|---|
|`ingressClassName`|Selects the Ingress controller class|
|`rules.host`|Hostname to match|
|`path`|URL path to match|
|`pathType: Prefix`|Matches the specified path prefix|
|`backend.service.name`|Destination Kubernetes Service|
|`backend.service.port.number`|Service port, not necessarily container port|

The `nginx` class must exist and be handled by an installed controller.

## 5.2 GatewayClass

A GatewayClass selects the implementation.

Example:

```
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: example-class
spec:
  controllerName: example.com/gateway-controller
```

**Important:** This is illustrative. `controllerName` must match an actual installed Gateway controller. Creating this YAML does not install one.

## 5.3 Gateway

```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: app-gateway
spec:
  gatewayClassName: example-class
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Same
```

### Important fields

|Field|Meaning|
|---|---|
|`gatewayClassName`|GatewayClass used by the Gateway|
|`listeners.name`|Listener identifier|
|`protocol`|HTTP, HTTPS, or another supported protocol|
|`port`|Listener port|
|`allowedRoutes`|Controls which Routes may attach|

## 5.4 HTTPRoute

```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
spec:
  parentRefs:
    - name: app-gateway

  hostnames:
    - example.com

  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: api-service
          port: 8080

    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: frontend-service
          port: 80
```

### Important fields

|Field|Meaning|
|---|---|
|`parentRefs`|Gateway or listener to attach to|
|`hostnames`|Hostnames to match|
|`rules.matches`|Request matching conditions|
|`PathPrefix`|Path prefix matching|
|`backendRefs`|Destination Services|
|`filters`|Supported request or response transformations|

## 5.5 Gateway API URL rewriting

Example HTTPRoute rule:

```
rules:
  - matches:
      - path:
          type: PathPrefix
          value: /api
    filters:
      - type: URLRewrite
        urlRewrite:
          path:
            type: ReplacePrefixMatch
            replacePrefixMatch: /
    backendRefs:
      - name: api-service
        port: 8080
```

Request behavior:

```
Incoming: /api/users
Backend:  /users
```

This requires a controller that supports the `URLRewrite` feature.

## 5.6 Gateway API traffic splitting

```
rules:
  - backendRefs:
      - name: app-v1
        port: 80
        weight: 90
      - name: app-v2
        port: 80
        weight: 10
```

This configures an approximate 90/10 traffic distribution between the two Services.

Weights are relative, not guarantees for a small number of requests.

---

# 6. Where to Use Ingress vs Gateway API

## Use Ingress when:

- You need straightforward hostname and path-based HTTP/HTTPS routing.
    
- Your cluster already uses a supported Ingress controller.
    
- Advanced routing features are unnecessary.
    
- You want a relatively simple configuration.
    
- Existing tooling and operational processes are built around Ingress.
    

## Use Gateway API when:

- You are designing a new platform and have a compatible implementation.
    
- Multiple applications share common gateway infrastructure.
    
- You need standardized header matching, traffic splitting, or request rewriting.
    
- Infrastructure and application teams need separate ownership.
    
- You want richer traffic-routing capabilities.
    
- Your implementation supports the Gateway API features you require.
    

## Comparison

|Feature|Ingress|Gateway API|
|---|---|---|
|HTTP routing|Yes|Yes|
|Hostname routing|Yes|Yes|
|Path routing|Yes|Yes|
|Header-based routing|Controller-specific|Standard HTTPRoute feature|
|Traffic splitting|Controller-specific|Standard HTTPRoute feature|
|URL rewriting|Often controller-specific|HTTPRoute filter|
|Configuration complexity|Lower|Higher|
|Resource separation|Limited|GatewayClass, Gateway, Route|
|Controller required|Yes|Yes|
|Kubernetes built-in API|Yes|CRDs required|
|New feature development|Frozen|Actively developed|

### SRE recommendation

**Ingress is not obsolete.** Continue using it where it meets requirements and is well supported.

For new environments needing more advanced traffic management, evaluate Gateway API first, but confirm controller compatibility and feature support before choosing it.

---

# 7. Important Commands

## Ingress

```
kubectl get ingress -A
kubectl describe ingress app-ingress
kubectl get ingressclass
kubectl get svc
kubectl get endpointslices
kubectl explain ingress.spec
```

## Gateway API

```
kubectl get gatewayclass
kubectl get gateway -A
kubectl get httproute -A

kubectl describe gateway app-gateway
kubectl describe httproute app-route

kubectl get gateway app-gateway -o yaml
kubectl get httproute app-route -o yaml
```

## Check whether Gateway API is installed

```
kubectl get crds | grep gateway.networking.k8s.io
```

## Test hostname routing

```
curl -H "Host: example.com" http://127.0.0.1:8080/
```

This assumes the controller entry point is accessible on local port 8080.

For normal DNS-based testing:

```
curl -v http://example.com/api
```

For local DNS override without editing `/etc/hosts`:

```
curl --resolve example.com:80:127.0.0.1 \
  http://example.com/api
```

---

# 8. Troubleshooting

## Common problems

|Problem|Possible cause|
|---|---|
|Ingress exists but traffic fails|Controller missing or incorrect IngressClass|
|Gateway not programmed|Controller or infrastructure configuration issue|
|HTTPRoute not accepted|Attachment permissions or listener mismatch|
|HTTP 404|Hostname or path doesn't match|
|HTTP 503|Backend Service or endpoints unavailable|
|Wrong application receives traffic|Incorrect routing rules|
|URL rewrite not working|Unsupported filter or incorrect rewrite|
|HTTPS certificate error|Incorrect certificate, hostname, or TLS configuration|

## Ingress troubleshooting sequence

```
kubectl get ingress -A
kubectl describe ingress app-ingress
kubectl get ingressclass
kubectl get svc
kubectl get endpointslices
```

Check controller logs when needed.

## Gateway API troubleshooting sequence

```
kubectl get gatewayclass
kubectl describe gateway app-gateway
kubectl describe httproute app-route
kubectl get svc
kubectl get endpointslices
```

### Gateway API status conditions

|Condition|Meaning|
|---|---|
|`Accepted`|Resource or Route attachment accepted|
|`Programmed`|Gateway configuration has been programmed|
|`ResolvedRefs`|Referenced backends and other objects resolved successfully|

Inspect the relevant resource's `status.conditions` for the actual reason when a condition is false.

## Kind / Minikube vs EKS

**Kubernetes-native:**

- Ingress and Gateway API resources describe desired routing.
    
- A compatible controller implements the configuration.
    

**Kind / Minikube:**

- Install a suitable controller.
    
- Expose the controller through local ports, port forwarding, or other supported networking.
    
- Local DNS or hosts-file entries may be needed for hostname testing.
    

**AWS EKS:**

- The controller determines how Kubernetes routing maps to AWS infrastructure.
    
- An AWS load balancer may be created depending on the implementation.
    
- Verify IAM permissions, networking, security groups, and controller logs.
    
- AWS load balancers can incur charges even when application traffic is low.
    

Do not assume every Gateway API controller supports the same features.

---

# 9. Interview Questions

**Q1. What is Kubernetes Ingress?**

Ingress is a Kubernetes API resource defining HTTP/HTTPS routing rules to Services. An Ingress controller implements those rules.

**Q2. What is Gateway API?**

Gateway API is a collection of Kubernetes networking APIs providing more flexible routing and separation between infrastructure and application configuration.

**Q3. Difference between Ingress and Gateway API?**

Ingress is simpler and focused on HTTP/HTTPS routing. Gateway API offers more structured resources and standardized advanced routing features.

**Q4. Can Ingress work without an Ingress controller?**

The resource can exist, but it will not provide working traffic routing without a compatible controller.

**Q5. What are GatewayClass, Gateway, and HTTPRoute?**

- GatewayClass selects the implementation.
    
- Gateway defines traffic listeners.
    
- HTTPRoute defines request matching and backend routing.
    

**Q6. What is the difference between hostname routing and path routing?**

Hostname routing matches the requested domain. Path routing matches the URL path.

**Q7. Does path matching automatically rewrite the URL?**

No. Rewriting requires an explicit supported configuration.

**Q8. How does an HTTPRoute connect to a Gateway?**

Using `parentRefs`, subject to listener and route attachment rules.

**Q9. Can Gateway API replace an AWS load balancer?**

Not necessarily. Gateway API defines routing configuration; a controller may provision or configure AWS load balancers to implement it.

**Q10. When should you choose Gateway API instead of Ingress?**

When advanced routing, shared gateway infrastructure, or clearer team ownership is needed and the selected controller supports the required features.

---

# 10. Quick Revision

- **Service:** Stable endpoint for Pods.
    
- **Ingress:** HTTP/HTTPS routing rules.
    
- **Ingress Controller:** Implements Ingress rules.
    
- **GatewayClass:** Selects Gateway implementation.
    
- **Gateway:** Defines traffic entry points and listeners.
    
- **HTTPRoute:** Matches requests and selects backends.
    
- **Host header:** Identifies the requested hostname.
    
- **DNS:** Resolves hostnames to IP addresses.
    
- **Path matching:** Selects a route based on the URL path.
    
- **URL rewriting:** Changes the path forwarded to the backend.
    
- **Traffic splitting:** Distributes requests across multiple backends.
    
- **Troubleshooting:** Check controller, Gateway/Ingress status, routing rules, Services, and endpoints.
    

**Final takeaway:** Use Ingress for simpler, established HTTP routing. Prefer Gateway API when you need more flexible, standardized traffic management and have a compatible controller.