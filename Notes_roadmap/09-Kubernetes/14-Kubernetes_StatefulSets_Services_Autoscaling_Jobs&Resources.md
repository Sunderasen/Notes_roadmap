#kubernetes #sre #cka #statefulset #autoscaling #jobs #resources

## 1. StatefulSets

### Concept

A StatefulSet manages applications that need **stable Pod names, stable network identities, and persistent storage**.

Unlike a Deployment, a StatefulSet gives each Pod a predictable identity.

### Definition

A StatefulSet is a Kubernetes workload controller that manages Pods with stable identities and supports dedicated persistent storage for each Pod.

### Why It Is Needed

- Stable Pod names such as `redis-0`, `redis-1`.
    
- Dedicated storage for individual Pods.
    
- Ordered Pod creation and deletion.
    
- Stable DNS names through a headless Service.
    

**Important:** A StatefulSet does not automatically configure database replication or clustering.

### How It Works

```
StatefulSet: redis
    |
    |-- redis-0 --> PVC-0 --> PV-0
    |-- redis-1 --> PVC-1 --> PV-1
    |-- redis-2 --> PVC-2 --> PV-2
    |
Headless Service: redis-headless
```

Each Pod receives its own PVC through `volumeClaimTemplates`.

### YAML / Configuration

```
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
    - name: redis
      port: 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis-headless
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7.4
          ports:
            - containerPort: 6379
          volumeMounts:
            - name: redis-data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        storageClassName: standard
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

**Important fields**

|Field|Purpose|
|---|---|
|`serviceName`|Associates the StatefulSet with its governing Service|
|`replicas`|Desired number of Pods|
|`volumeClaimTemplates`|Creates individual PVCs for Pods|
|`clusterIP: None`|Creates a headless Service|
|`storageClassName`|Selects the storage provisioner|

In Kind, `standard` typically uses the local-path provisioner.

### Important Commands

```
kubectl get statefulsets
kubectl get pods -l app=redis
kubectl get pvc
kubectl get pv
kubectl describe statefulset redis
kubectl rollout status statefulset/redis
```

Pod DNS name:

```
redis-0.redis-headless.default.svc.cluster.local
```

### Troubleshooting

**Pod stuck in Pending**

```
kubectl describe pod redis-0
kubectl get pvc
kubectl describe pvc redis-data-redis-0
kubectl get storageclass
```

Check PVC binding, available storage, and scheduling events.

**Pod stuck during rollout**

By default, StatefulSets use ordered rollout behavior. A Pod that is not Ready can prevent later Pods from being created or updated.

```
kubectl get pods -l app=redis
kubectl describe pod redis-0
kubectl logs redis-0
```

### Interview Questions

**Q: Deployment vs StatefulSet?**

Deployment manages interchangeable Pods. StatefulSet manages Pods with stable identities and optional dedicated storage.

**Q: What happens when a StatefulSet Pod is deleted?**

The controller recreates the Pod with the same name and can reuse its existing PVC.

**Q: Does deleting a StatefulSet delete its PVCs?**

By default, StatefulSet PVCs are retained. PVC retention policies can change this behavior.

---

## 2. ExternalName Service

### Concept

ExternalName lets applications use a Kubernetes Service name that points to another DNS hostname.

It is **DNS aliasing**, not load balancing or proxying.

### Definition

ExternalName is a Kubernetes Service type that maps a Service DNS name to an external hostname using a DNS CNAME record.

### Why It Is Needed

- Reference an external service using a Kubernetes DNS name.
    
- Reduce hardcoded external hostnames in application configurations.
    
- Change the external DNS target through Service configuration.
    

### How It Works

```
Application Pod
      |
      | DNS query: external-api
      v
Kubernetes DNS
      |
      | CNAME
      v
api.example.com
      |
      v
External DNS resolution
      |
      v
Destination IP
```

The application connects to the resolved destination IP. Traffic does not pass through an ExternalName Service proxy.

### YAML / Configuration

```
apiVersion: v1
kind: Service
metadata:
  name: external-api
spec:
  type: ExternalName
  externalName: api.example.com
```

`api.example.com` is an illustrative hostname.

### Important Commands

```
kubectl get svc external-api
kubectl describe svc external-api
kubectl run netshoot \
  --image=nicolaka/netshoot \
  --restart=Never \
  --command -- sleep 3600
kubectl exec -it netshoot -- sh
```

Inside Netshoot:

```
nslookup external-api.default.svc.cluster.local
nslookup api.example.com
```

### Important Limitations

- ExternalName does not create a ClusterIP.
    
- It does not create normal Service Endpoints.
    
- It does not perform health checks or load balancing.
    
- It does not rewrite HTTP `Host` headers or TLS SNI names.
    
- External DNS resolution must work.
    

**HTTP/TLS warning:** Connecting to an ExternalName alias can fail when the upstream server expects a different hostname or certificate name.

### Troubleshooting

```
kubectl get svc external-api -o yaml
kubectl -n kube-system get configmap coredns -o yaml
kubectl -n kube-system logs deployment/coredns
```

If you see `SERVFAIL`, investigate upstream DNS resolution rather than assuming the Service is misconfigured.

### Interview Questions

**Q: ExternalName vs ClusterIP?**

ClusterIP provides a virtual Service IP and routes traffic to backends. ExternalName provides a DNS CNAME alias.

**Q: Can ExternalName point directly to an IP?**

No. `externalName` is intended for a DNS hostname, not an IP address.

---

## 3. Horizontal Pod Autoscaler (HPA)

### Concept

HPA automatically changes the **number of Pods** based on observed metrics.

### Definition

Horizontal Pod Autoscaler is a Kubernetes controller that adjusts replica counts for supported scalable workloads such as Deployments and StatefulSets.

### Why It Is Needed

- Increase replicas when demand grows.
    
- Reduce replicas when demand decreases.
    
- Avoid manually adjusting replica counts.
    

### How It Works

```
Metrics Server / Metrics API
          |
          v
      HPA Controller
          |
          | Compare current and target metrics
          v
     Deployment Scale
          |
     2 Pods → 4 Pods
```

For CPU utilization, HPA uses CPU usage relative to requested CPU.

Simplified formula:

```
Desired replicas =
Current replicas × (Current utilization / Target utilization)
```

Kubernetes also applies tolerance, readiness handling, stabilization, and scaling policies.

### YAML / Configuration

Example Deployment container resources:

```
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

HPA:

```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

**Important fields**

|Field|Meaning|
|---|---|
|`scaleTargetRef`|Workload to scale|
|`minReplicas`|Minimum desired replicas|
|`maxReplicas`|Maximum desired replicas|
|`averageUtilization`|Target CPU usage relative to requests|

For CPU utilization-based HPA, CPU requests must be configured for the relevant containers.

### Important Commands

```
kubectl get hpa
kubectl describe hpa web-hpa
kubectl top pods
kubectl top nodes
kubectl get deployment web
kubectl get apiservice v1beta1.metrics.k8s.io
```

**Kind requirement:** Metrics Server is not generally installed by default. Install and validate a compatible Metrics Server setup before running CPU-based HPA labs.

### Troubleshooting

**HPA shows** `**<unknown>**`

Check:

```
kubectl describe hpa web-hpa
kubectl top pods
kubectl get apiservice v1beta1.metrics.k8s.io
```

Possible causes:

- Metrics Server missing or unhealthy.
    
- CPU requests missing.
    
- Metrics unavailable for Pods.
    
- Metrics API communication errors.
    

**HPA does not scale**

Check actual utilization, target thresholds, stabilization windows, and whether maximum replicas have been reached.

### Interview Questions

**Q: Does HPA increase CPU limits?**

No. HPA primarily changes replica counts.

**Q: Can HPA scale on memory?**

Yes, and it can also use supported custom and external metrics.

---

## 4. Vertical Pod Autoscaler (VPA)

### Concept

VPA adjusts or recommends the **CPU and memory resources assigned to containers**.

HPA scales the number of Pods. VPA changes resource sizing.

### Definition

Vertical Pod Autoscaler is an additional Kubernetes autoscaling component that recommends and, depending on its configuration and version, applies container resource changes.

### Why It Is Needed

- Identify underallocated resources.
    
- Reduce wasted resource requests.
    
- Improve workload sizing.
    

### How It Works

```
Pod Resource Usage
        |
        v
   VPA Recommender
        |
        v
CPU / Memory Recommendations
        |
        v
Optional Resource Updates
```

### YAML / Configuration

Example VPA resource using the commonly deployed `autoscaling.k8s.io/v1` API:

```
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: "Off"
```

`Off` mode produces recommendations without automatically changing Pod resources.

VPA is an add-on, not a built-in controller installed by default in Kind. The VPA CRDs and controllers must be installed first.

### Important Commands

```
kubectl get vpa
kubectl describe vpa web-vpa
kubectl get crd | grep verticalpodautoscaler
```

Inspect recommendations:

```
kubectl get vpa web-vpa -o yaml
```

### HPA vs VPA

|Feature|HPA|VPA|
|---|---|---|
|Changes|Replica count|Container resources|
|Scaling|Horizontal|Vertical|
|Metrics|CPU, memory, custom/external|Resource usage history|
|Extra installation|Metrics pipeline commonly required|VPA controllers and CRDs|
|Main concern|Enough replicas|Correct resource sizing|

**Important:** HPA and VPA can conflict when both control CPU or memory for the same workload. Use carefully coordinated metrics and policies.

### Troubleshooting

If VPA is not working, inspect its installation, CRDs, controller logs, target references, and recommendation status.

### Interview Questions

**Q: Does VPA automatically increase node capacity?**

No. It adjusts or recommends Pod resources. A separate node autoscaling mechanism may be needed.

**Q: Can VPA restart Pods?**

Some update modes can require Pod recreation, while supported in-place update behavior depends on the Kubernetes and VPA versions and configuration.

---

## 5. Jobs and CronJobs

### Concept

A Job runs a task until the required number of successful executions is reached.

A CronJob creates Jobs according to a schedule.

### Definition

- **Job:** Manages Pods that perform finite tasks.
    
- **CronJob:** Creates Jobs on a recurring schedule.
    

### Why They Are Needed

- Batch processing.
    
- One-time maintenance tasks.
    
- Scheduled scripts.
    
- Periodic cleanup or reports.
    

### How Jobs Work

```
Job Created
     |
     v
Pod Starts
     |
     |-- Exit 0 → Success
     |
     |-- Exit nonzero → Failure
                           |
                           v
                     Retry policy
```

### Job YAML

```
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 2
  ttlSecondsAfterFinished: 300
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: busybox:1.36
          command:
            - sh
            - -c
            - echo "Task started"; sleep 10; echo "Task completed"
```

**Important fields**

|Field|Meaning|
|---|---|
|`completions`|Required successful Pod completions|
|`parallelism`|Maximum desired concurrent Pods|
|`backoffLimit`|Failure retry threshold|
|`restartPolicy`|`Never` or `OnFailure`|
|`ttlSecondsAfterFinished`|Cleanup delay after Job finishes|
|`activeDeadlineSeconds`|Maximum active execution duration|

With `restartPolicy: Never`, a failed Job Pod is not restarted in place; the Job controller may create a replacement Pod.

### CronJob YAML

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scheduled-job
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: worker
              image: busybox:1.36
              command:
                - sh
                - -c
                - date
```

Runs approximately every five minutes, subject to scheduling conditions.

`concurrencyPolicy: Forbid` prevents overlapping Jobs created by this CronJob.

CronJobs should be designed to tolerate missed or duplicate executions.

### Important Commands

```
kubectl get jobs
kubectl get cronjobs
kubectl describe job batch-job
kubectl logs job/batch-job
kubectl get pods -l job-name=batch-job
kubectl describe cronjob scheduled-job
```

### Troubleshooting

**Job repeatedly fails**

```
kubectl describe job batch-job
kubectl get pods -l job-name=batch-job
kubectl logs job/batch-job
```

Check:

- Container exit codes.
    
- Application errors.
    
- Missing configuration.
    
- Failed image pulls.
    
- Resource limits.
    
- `BackoffLimitExceeded`.
    

### Interview Questions

**Q: Job vs Deployment?**

A Deployment maintains continuously running Pods. A Job runs tasks until completion.

**Q: What does** `**backoffLimit: 2**` **mean?**

It sets the Job failure retry threshold. Exact Pod counts can depend on restart policy and execution behavior.

**Q: CronJob vs Job?**

A CronJob schedules Jobs; a Job manages execution to completion.

---

## 6. Resource Requests and Limits

### Concept

Requests and limits control how Kubernetes schedules Pods and constrains container resource consumption.

### Definition

**Requests:** Resources Kubernetes uses for scheduling and resource allocation decisions.

**Limits:** Resource consumption boundaries enforced by the container runtime and Linux kernel, depending on resource type.

### Why They Are Needed

- Avoid overloading nodes.
    
- Improve scheduling decisions.
    
- Prevent uncontrolled resource consumption.
    
- Support capacity planning and autoscaling.
    

### How It Works

```
Pod requests 200m CPU, 256Mi memory
              |
              v
Kubernetes Scheduler
              |
              v
Node with sufficient available
allocatable requested capacity
              |
              v
Pod starts with configured limits
```

### YAML / Configuration

```
apiVersion: v1
kind: Pod
metadata:
  name: resource-lab
spec:
  containers:
    - name: nginx
      image: nginx:stable
      resources:
        requests:
          cpu: "200m"
          memory: "256Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
```

### CPU Units

```
1000m = 1 CPU
500m  = 0.5 CPU
200m  = 0.2 CPU
```

### Memory Units

```
1Mi = 1,048,576 bytes
1Gi = 1024Mi
```

### CPU vs Memory Limits

|Resource|When limit is exceeded|
|---|---|
|CPU|Container is typically throttled|
|Memory|Container may be terminated by OOM killer|

CPU throttling slows execution.

Memory OOM termination can cause the container to restart according to its restart policy.

### Requests vs Limits

|Feature|Request|Limit|
|---|---|---|
|Used for scheduling|Yes|Not directly|
|Minimum reserved capacity for scheduling|Yes|No|
|Restricts runtime usage|No|Yes|
|Influences HPA CPU utilization|Yes|No|

Requests are scheduling reservations, not guarantees of constant performance.

### Important Commands

```
kubectl describe pod resource-lab
kubectl top pod resource-lab
kubectl top nodes
kubectl describe node
kubectl get resourcequota
kubectl get limitrange
```

### Troubleshooting

**Pod Pending**

```
kubectl describe pod resource-lab
```

Look for:

```
Insufficient cpu
Insufficient memory
```

This means the scheduler could not find sufficient allocatable capacity for the Pod's requests.

**OOMKilled**

```
kubectl describe pod resource-lab
kubectl get pod resource-lab \
  -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
```

An `OOMKilled` result indicates the container was terminated due to an out-of-memory condition. Investigate memory usage, application behavior, and configured limits.

**CPU throttling**

Investigate CPU usage and throttling metrics. `kubectl top` alone does not directly report throttling.

### Interview Questions

**Q: What happens if a Pod requests 2 CPUs but the node only has 1 CPU available for scheduling?**

The Pod remains Pending unless another suitable node can satisfy its scheduling requirements.

**Q: What happens if CPU usage exceeds the limit?**

The container is typically throttled rather than killed.

**Q: What happens if memory usage exceeds the limit?**

The container may be OOM-killed.

**Q: Are requests and limits mandatory?**

Not universally. They may be supplied by namespace LimitRanges or required by policies.

---

## 7. Quick Revision

|Topic|Remember|
|---|---|
|StatefulSet|Stable Pod identity and dedicated persistent storage|
|ExternalName|DNS CNAME alias, not a proxy|
|HPA|Changes replica count|
|VPA|Recommends or changes container resources|
|Job|Runs a finite task to completion|
|CronJob|Schedules Jobs|
|CPU request|Used for scheduling and utilization calculations|
|CPU limit|Can cause throttling|
|Memory request|Used for scheduling|
|Memory limit|Can result in OOM termination|

## 8. SRE Troubleshooting Cheat Sheet

```
# Workloads
kubectl get statefulsets
kubectl get deployments
kubectl get pods -o wide

# Storage
kubectl get pvc,pv
kubectl get storageclass

# DNS
kubectl get svc
kubectl -n kube-system get configmap coredns

# Autoscaling
kubectl get hpa
kubectl describe hpa <name>
kubectl top pods

# Batch workloads
kubectl get jobs,cronjobs
kubectl describe job <name>
kubectl logs job/<name>

# Resources
kubectl describe pod <name>
kubectl top nodes
kubectl describe node <node-name>
```

## 9. Hands-On Practice Checklist

- Deploy a three-replica StatefulSet in Kind.
    
- Verify each Pod has its own PVC.
    
- Delete one StatefulSet Pod and verify its stable identity and storage.
    
- Create an ExternalName Service and inspect its DNS CNAME.
    
- Install and validate Metrics Server before testing HPA.
    
- Scale a Deployment using HPA.
    
- Explore VPA recommendations after installing the VPA add-on.
    
- Create a successful Job and a failing Job.
    
- Schedule a CronJob.
    
- Configure CPU/memory requests and limits.
    
- Troubleshoot a Pending Pod caused by insufficient resource requests.
    

## Final Revision

**StatefulSet = Identity + Storage**

**ExternalName = DNS Alias**

**HPA = More or Fewer Pods**

**VPA = Bigger or Smaller Resource Allocation**

**Job = Run Until Complete**

**CronJob = Run on Schedule**

**Requests = Scheduling**

**Limits = Runtime Boundaries**