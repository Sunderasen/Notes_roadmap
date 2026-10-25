# 12 - Kubernetes Probes, Container Types, ConfigMaps & Secrets

**Topic:** Kubernetes Workload Configuration and Troubleshooting  
**Environment:** Minikube / AWS EKS  
**Category:** Kubernetes  
**Tags:** #kubernetes #sre #probes #containers #configmap #secrets #troubleshooting

---

# 1. Kubernetes Probes

## 1.1 Concept

Kubernetes uses probes to check the condition of applications running inside containers.

A Pod being in the `Running` state does not necessarily mean its application is healthy or ready to receive traffic.

## 1.2 Definition

A probe is a health check performed by the kubelet to determine whether a container has started, is healthy, or is ready to serve requests.

There are three types:

| Probe | Purpose | Failure action |
|---|---|---|
| `startupProbe` | Has the application finished starting? | Restart container after failure threshold |
| `livenessProbe` | Is the application still healthy? | Restart container after failure threshold |
| `readinessProbe` | Can the application serve traffic? | Mark container not ready; remove Pod from eligible Service endpoints |

## 1.3 Why It Is Needed

- Detect applications that are running but unresponsive.
- Avoid sending traffic to applications that are not ready.
- Automatically restart unhealthy containers.
- Allow slow-starting applications sufficient initialization time.

## 1.4 How It Works

The kubelet periodically executes configured health checks.

Kubernetes supports three common probe mechanisms:

| Mechanism | Description |
|---|---|
| `httpGet` | Checks an HTTP endpoint |
| `tcpSocket` | Checks whether a TCP connection can be established |
| `exec` | Executes a command inside the container |

Kubernetes also supports gRPC health probes.

**Important distinction:**

- Liveness failure → container restart.
- Readiness failure → Pod becomes NotReady; no restart caused by readiness failure alone.
- Startup probe → disables liveness and readiness checks until startup succeeds.

## 1.5 YAML / Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-lab
spec:
  containers:
    - name: nginx
      image: nginx:stable
      ports:
        - containerPort: 80

      startupProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 5
        failureThreshold: 12

      readinessProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 5
        failureThreshold: 3

      livenessProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 10
        failureThreshold: 3
```

### Important fields

| Field | Meaning |
|---|---|
| `initialDelaySeconds` | Delay before initial probing |
| `periodSeconds` | Interval between probe executions |
| `timeoutSeconds` | Maximum time allowed for each probe |
| `failureThreshold` | Consecutive failures before declaring failure |
| `successThreshold` | Consecutive successes required to recover; must be 1 for liveness and startup probes |

## 1.6 Important Commands

```bash
kubectl get pods
kubectl describe pod probe-lab
kubectl logs probe-lab
kubectl get pod probe-lab -o yaml
kubectl explain pod.spec.containers.livenessProbe
```

## 1.7 Troubleshooting

| Symptom | Possible cause | Investigation |
|---|---|---|
| Frequent restarts | Liveness probe failing | `kubectl describe pod` |
| Running but NotReady | Readiness probe failing | Check events and endpoint |
| Restarting during startup | Startup probe insufficient | Check startup duration and thresholds |
| HTTP probe returns 404 | Incorrect path | Verify application endpoint |
| TCP probe fails | Application not listening | Check listening port and logs |

**SRE note:** A successful TCP probe proves the port accepts connections, not that the application is functioning correctly.

## 1.8 Interview Questions

**Q1. What is the difference between readiness and liveness probes?**

Readiness determines whether a container can receive traffic. Liveness determines whether Kubernetes should restart it.

**Q2. Does readiness probe failure restart a container?**

No. It marks the container not ready.

**Q3. Why use a startup probe?**

To protect slow-starting applications from premature liveness failures.

## 1.9 Quick Revision

- Startup = Has the application started?
- Liveness = Should Kubernetes restart it?
- Readiness = Should Kubernetes send traffic?
- Probes are executed by the kubelet.
- Bad probe configuration can cause unnecessary outages.

---

# 2. Kubernetes Container Types

## 2.1 Concept

A Kubernetes Pod can contain multiple containers that work together.

Different container types serve different purposes during startup, normal execution, and troubleshooting.

## 2.2 Definition

| Container type | Purpose |
|---|---|
| Application container | Runs the main application |
| Init container | Performs startup preparation |
| Native sidecar container | Runs alongside application containers and supports their operation |
| Ephemeral container | Provides temporary troubleshooting tools |

## 2.3 Why It Is Needed

- Separate startup preparation from application execution.
- Add supporting functionality without modifying application code.
- Troubleshoot minimal container images without rebuilding them.
- Share network and storage resources between cooperating containers.

## 2.4 How It Works

### Application container

Runs the main workload and is defined under `spec.containers`.

### Init container

Runs before regular application containers.

- Runs to completion.
- Executes sequentially with other regular init containers.
- Failure can block application startup.

### Native sidecar container

Defined under `spec.initContainers` with:

```yaml
restartPolicy: Always
```

It starts during initialization and continues running alongside the application.

Unlike regular init containers, it does not need to finish before the application starts.

### Ephemeral container

Added to an existing Pod for troubleshooting.

- Not intended for application workloads.
- Cannot be removed individually after being added.
- Does not automatically inherit another container's filesystem or volume mounts.
- Can inspect targeted container processes when supported by the runtime.

## 2.5 YAML / Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: container-types-lab
spec:
  volumes:
    - name: shared-logs
      emptyDir: {}

  initContainers:
    - name: init-setup
      image: busybox:1.36
      command: ["sh", "-c", "echo Initialization completed"]

    - name: log-sidecar
      image: busybox:1.36
      restartPolicy: Always
      command:
        - sh
        - -c
        - |
          touch /logs/app.log
          exec tail -f /logs/app.log
      volumeMounts:
        - name: shared-logs
          mountPath: /logs

  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          while true; do
            echo "Application running" >> /logs/app.log
            sleep 10
          done
      volumeMounts:
        - name: shared-logs
          mountPath: /logs
```

### Important fields

| Field | Meaning |
|---|---|
| `containers` | Regular application containers |
| `initContainers` | Initialization containers and native sidecars |
| `restartPolicy: Always` | Makes an init container behave as a native sidecar |
| `emptyDir` | Temporary volume shared by containers in the Pod |
| `volumeMounts` | Makes the shared volume accessible inside each container |

## 2.6 Important Commands

```bash
kubectl get pod container-types-lab

kubectl describe pod container-types-lab

kubectl logs container-types-lab -c init-setup

kubectl logs container-types-lab -c log-sidecar

kubectl exec -it container-types-lab -c app -- sh
```

### Debug an existing container

```bash
kubectl debug -it container-types-lab \
  --image=busybox:1.36 \
  --target=app \
  --container=debugger \
  -- sh
```

Inside the debugger:

```bash
ps
cat /proc/1/cmdline | tr '\0' ' '
```

The `--target` option requests access to the target container's process namespace. Actual visibility depends on container runtime support.

## 2.7 Troubleshooting

| Symptom | Possible cause |
|---|---|
| Pod stuck in `Init:CrashLoopBackOff` | Init container repeatedly failing |
| App cannot find generated configuration | Init container wrote to wrong location or volume |
| Sidecar cannot read logs | Incorrect shared volume or mount path |
| Debug container cannot see processes | Targeting/runtime namespace limitations |
| Logs disappear after Pod removal | `emptyDir` is temporary |

## 2.8 Interview Questions

**Q1. Init container vs sidecar container?**

An ordinary init container runs to completion before the application starts. A native sidecar starts during initialization and continues running.

**Q2. Can two containers in a Pod communicate over localhost?**

Yes. Containers in the same Pod share the Pod network namespace.

**Q3. Why use `kubectl debug` instead of `kubectl exec`?**

To introduce debugging tools when the application image lacks utilities such as a shell, `ps`, or networking tools.

## 2.9 Quick Revision

- Application = Main workload.
- Init = Preparation before application startup.
- Native sidecar = Supporting container running alongside application.
- Ephemeral = Troubleshooting.
- Same Pod = Shared network namespace.
- Shared files require appropriate volume mounts.

---

# 3. Kubernetes ConfigMaps

## 3.1 Concept

ConfigMaps separate application configuration from container images.

The same image can be reused across development, testing, and production with different configuration values.

## 3.2 Definition

A ConfigMap is a Kubernetes API object used to store non-sensitive configuration as key-value pairs or files.

## 3.3 Why It Is Needed

- Avoid hardcoding application settings.
- Reuse container images across environments.
- Change configuration without rebuilding images.
- Manage application configuration independently.

## 3.4 How It Works

1. Create a ConfigMap.
2. Reference it from a Pod.
3. Kubernetes provides its values through environment variables or mounted files.
4. The application reads the configuration.

### ConfigMap consumption methods

| Method | Configuration | Update behavior |
|---|---|---|
| Individual environment variable | `configMapKeyRef` | Requires container restart/replacement |
| Multiple environment variables | `envFrom.configMapRef` | Requires container restart/replacement |
| Mounted files | `volumes.configMap` | Files update eventually, except `subPath` mounts |

Applications must reread or reload updated mounted files to use new values.

## 3.5 YAML / Configuration

### Create a ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sre-app-config
data:
  APP_ENV: "development"
  LOG_LEVEL: "INFO"
```

### Inject environment variables

```yaml
envFrom:
  - configMapRef:
      name: sre-app-config
```

### Mount ConfigMap as a volume

```yaml
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
      volumeMounts:
        - name: app-config
          mountPath: /etc/app-config
          readOnly: true

  volumes:
    - name: app-config
      configMap:
        name: sre-app-config
```

The container receives files:

```text
/etc/app-config/APP_ENV
/etc/app-config/LOG_LEVEL
```

## 3.6 Important Commands

```bash
kubectl get configmaps

kubectl describe configmap sre-app-config

kubectl get configmap sre-app-config -o yaml

kubectl exec configmap-lab -- printenv APP_ENV

kubectl exec volume-mount-lab -- ls /etc/app-config

kubectl exec volume-mount-lab -- cat /etc/app-config/APP_ENV
```

## 3.7 Troubleshooting

| Error | Cause | Investigation |
|---|---|---|
| `CreateContainerConfigError` | Missing ConfigMap or key | `kubectl describe pod` |
| Environment value unchanged | Container has old injected value | Restart/recreate workload |
| Mounted file missing | Incorrect ConfigMap key or mount | Inspect volume configuration |
| Mounted file not updating | `subPath` mount or propagation delay | Inspect volume configuration |
| Application uses old configuration | Application does not reload file | Check application behavior |

## 3.8 Interview Questions

**Q1. Why use ConfigMaps instead of hardcoding values?**

To separate configuration from the application image and reuse the same image across environments.

**Q2. Does updating a ConfigMap update running container environment variables?**

No. The container must be restarted or replaced.

**Q3. Can ConfigMaps be mounted as volumes?**

Yes. Each key can appear as a file inside the mounted directory.

## 3.9 Quick Revision

- ConfigMap = Non-sensitive configuration.
- `data` = Configuration key-value pairs.
- `envFrom` = Import multiple keys as environment variables.
- `configMapKeyRef` = Import one key.
- `volumes.configMap` = Mount configuration as files.
- Environment variables do not update automatically.

---

# 4. Kubernetes Secrets

## 4.1 Concept

Secrets provide a Kubernetes mechanism for storing and distributing sensitive configuration.

Examples include database passwords, API tokens, and credentials.

## 4.2 Definition

A Secret is a Kubernetes API object designed to hold sensitive data.

A common Secret type is `Opaque`, used for general-purpose key-value credentials.

## 4.3 Why It Is Needed

- Avoid embedding credentials inside container images.
- Separate credentials from application configuration.
- Control access through Kubernetes RBAC.
- Support credential updates independently of application images.

**Security warning:** Kubernetes Secrets are not automatically secure simply because they are stored in a Secret object. Proper access controls and encryption configuration are required.

## 4.4 How It Works

Secrets support two common input fields:

| Field | Purpose |
|---|---|
| `data` | Base64-encoded values |
| `stringData` | Plain-text input converted by Kubernetes into `data` |

### Base64 vs encryption

Base64 is encoding, not encryption.

```bash
echo -n "lab-password-123" | base64
```

Decoding requires no encryption key:

```bash
echo "bGFiLXBhc3N3b3JkLTEyMw==" | base64 -d
```

**Production protection:**

- Restrict Secret access using RBAC.
- Enable or verify encryption at rest.
- Avoid committing actual credentials to Git.
- Avoid printing passwords in logs or terminal output.
- Consider external secret-management solutions where appropriate.

## 4.5 YAML / Configuration

### Create a Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  DB_USERNAME: "admin"
  DB_PASSWORD: "lab-password-123"
```

*Dummy credentials for learning only.*

### Inject one Secret key as an environment variable

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: DB_PASSWORD
```

### Mount a Secret as files

```yaml
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
      volumeMounts:
        - name: db-secret
          mountPath: /etc/db-secret
          readOnly: true

  volumes:
    - name: db-secret
      secret:
        secretName: db-credentials
```

Mounted files:

```text
/etc/db-secret/DB_USERNAME
/etc/db-secret/DB_PASSWORD
```

### Secret update behavior

| Consumption method | Update behavior |
|---|---|
| Environment variable | Requires container restart/replacement |
| Mounted volume | Files update eventually |
| Mounted using `subPath` | Does not receive automatic updates |

Even when mounted files update, the application may require a configuration reload or restart.

## 4.6 Important Commands

```bash
kubectl get secrets

kubectl describe secret db-credentials

kubectl get secret db-credentials -o jsonpath='{.data}' 
```

The last command displays encoded Secret data; use it cautiously because encoded values are reversible.

For safer key inspection:

```bash
kubectl describe secret db-credentials
```

To verify mounted filenames without exposing credentials:

```bash
kubectl exec volume-mount-lab -- ls -l /etc/db-secret
```

## 4.7 Troubleshooting

### Error: `CreateContainerConfigError`

Start with:

```bash
kubectl describe pod secret-lab
```

Example event:

```text
Error: couldn't find key DB_PASSWORD in Secret default/db-credentials
```

Inspect the Secret:

```bash
kubectl describe secret db-credentials
```

Check whether the referenced key exists.

### Common problems

| Problem | Possible cause |
|---|---|
| Secret not found | Wrong name or namespace |
| Key not found | `secretKeyRef.key` mismatch |
| Old password still used | Environment variable not refreshed or application caching |
| Secret file missing | Wrong mount path or key |
| Credential exposed | Environment or logs printed sensitive data |

## 4.8 Interview Questions

**Q1. ConfigMap vs Secret?**

ConfigMaps are for non-sensitive configuration. Secrets are intended for sensitive information and require appropriate security controls.

**Q2. Is Base64 encryption?**

No. Anyone can decode Base64 without a key.

**Q3. What happens when a Secret used as an environment variable changes?**

The running container retains its existing value until restarted or replaced.

**Q4. How do you troubleshoot a missing Secret key?**

Use `kubectl describe pod`, inspect the error, and compare the requested key with the Secret's available keys using `kubectl describe secret`.

## 4.9 Quick Revision

- Secret = Sensitive configuration.
- `Opaque` = General-purpose Secret type.
- `data` = Base64-encoded values.
- `stringData` = Plain-text input.
- `secretKeyRef` = Inject one Secret key.
- `secretRef` under `envFrom` = Inject multiple keys.
- Secret volumes = Mount credentials as files.
- Base64 does not provide security.

---

# 5. Container Lifecycle Hooks

## 5.1 Concept

Lifecycle hooks allow Kubernetes to execute actions at particular stages of a container's lifecycle.

## 5.2 Definition

| Hook | Purpose |
|---|---|
| `postStart` | Executes after container creation |
| `preStop` | Executes before normal container termination |

`postStart` is not guaranteed to execute before the container's main process starts.

## 5.3 Why It Is Needed

- Perform container startup actions.
- Support graceful shutdown.
- Give applications time to finish active requests.

## 5.4 How It Works

During graceful Pod termination:

1. Kubernetes starts the termination grace-period countdown.
2. The kubelet executes the `preStop` hook, if configured.
3. After the hook finishes, the runtime sends the container termination signal, normally `SIGTERM`.
4. The application can perform graceful shutdown.
5. If the grace period expires, the container may be forcefully terminated with `SIGKILL`.

If `preStop` is still running when the grace period expires, the kubelet may request a one-time two-second extension.

## 5.5 YAML / Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-lab
spec:
  terminationGracePeriodSeconds: 45
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
      lifecycle:
        postStart:
          exec:
            command: ["sh", "-c", "echo started > /tmp/start.log"]
        preStop:
          exec:
            command: ["sh", "-c", "sleep 10"]
```

`terminationGracePeriodSeconds` is a Pod-level field.

**Important:** Sleeping in `preStop` does not itself guarantee that the application finishes active requests. The application must support graceful shutdown.

## 5.6 Important Commands

```bash
kubectl describe pod lifecycle-lab

kubectl get pod lifecycle-lab -o yaml

kubectl delete pod lifecycle-lab
```

Deletion is a disruptive action. Use only on disposable lab workloads or after checking production impact.

## 5.7 Troubleshooting

- Hook command fails → inspect Pod events.
- Shutdown takes too long → check grace period and application behavior.
- Requests fail during termination → investigate readiness changes, connection draining, and graceful shutdown.
- Container receives `SIGKILL` → investigate whether the grace period was exceeded.

## 5.8 Interview Questions

**Q1. Difference between `postStart` and `preStop`?**

`postStart` runs after container creation. `preStop` runs during graceful termination before the termination signal.

**Q2. Does `preStop` time count toward the grace period?**

Yes.

**Q3. If a hook takes 10 seconds and shutdown needs 20 seconds, is a 20-second grace period sufficient?**

No. At least 30 seconds is needed, preferably with a buffer.

## 5.9 Quick Revision

- `postStart` = Startup lifecycle action.
- `preStop` = Pre-termination action.
- `terminationGracePeriodSeconds` = Total graceful termination budget.
- `SIGTERM` = Request graceful shutdown.
- `SIGKILL` = Forceful termination.

---

# 6. SRE Troubleshooting Cheat Sheet

| Symptom | First command | What to inspect |
|---|---|---|
| Pod stuck in `Pending` | `kubectl describe pod <pod>` | Scheduling events |
| `CreateContainerConfigError` | `kubectl describe pod <pod>` | Missing Secret, ConfigMap, or key |
| `CrashLoopBackOff` | `kubectl logs <pod> --previous` | Previous container failure |
| Readiness failing | `kubectl describe pod <pod>` | Probe events and endpoint |
| Container restarts | `kubectl describe pod <pod>` | Last state, exit code, OOM |
| Init container fails | `kubectl logs <pod> -c <init-container>` | Initialization errors |
| Sidecar missing logs | `kubectl describe pod <pod>` | Shared volume mounts |
| ConfigMap value outdated | `kubectl exec <pod> -- printenv APP_ENV` | Environment update behavior |
| Secret mount missing | `kubectl describe pod <pod>` | Secret name, key, mount |
| YAML unknown field | `kubectl explain pod.spec.containers` | YAML hierarchy and indentation |

## Resource requests and limits

A common editor warning is:

> One or more containers do not have resources — this can cause noisy neighbor issues.

Example configuration:

```yaml
resources:
  requests:
    cpu: "10m"
    memory: "16Mi"
  limits:
    cpu: "100m"
    memory: "64Mi"
```

- CPU requests help scheduling and influence CPU allocation under contention.
- CPU limits can cause throttling.
- Memory requests help scheduling.
- Memory limits can lead to OOM kills if exceeded.

Missing resources are not necessarily a YAML error, but appropriate resource configuration is important in production.

---

# 7. Final Interview Revision

| Question | Short answer |
|---|---|
| Liveness vs readiness? | Restart decision vs traffic eligibility |
| Why startup probe? | Protect slow-starting applications |
| Init vs sidecar? | One completes before app startup; one keeps running |
| Why ephemeral containers? | Troubleshoot existing Pods |
| ConfigMap vs Secret? | Non-sensitive vs sensitive configuration |
| Is Base64 encryption? | No |
| Does updating `envFrom` values update running containers? | No |
| Do mounted ConfigMaps update? | Eventually, except `subPath` |
| Does an application automatically reload mounted configuration? | Depends on the application |
| Where are Pod volumes defined? | `spec.volumes` |
| Where are volume mounts defined? | `spec.containers[].volumeMounts` |
| What is `preStop`? | Hook executed during graceful termination |
| What does `CreateContainerConfigError` indicate? | Container configuration cannot be resolved |
| First troubleshooting command? | Usually `kubectl describe pod` |

## Final Quick Revision

**Probes:** Startup, liveness, readiness.

**Container types:** Application, init, native sidecar, ephemeral.

**ConfigMap:** Non-sensitive configuration, consumed through environment variables or volumes.

**Secret:** Sensitive configuration, with Base64 encoding but not automatic protection from unauthorized access.

**Volumes:** Defined at Pod level, mounted at container level.

**Lifecycle:** `postStart`, `preStop`, graceful termination.

**SRE principle:** Inspect Pod events, container states, logs, and configuration before making changes.