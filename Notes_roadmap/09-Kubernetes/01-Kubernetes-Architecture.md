# Kubernetes Architecture

Tags: #kubernetes #architecture #sre #cka

## Concept
Kubernetes runs applications in containers and works to keep them in the state we asked for.

## Definition — interview answer
Kubernetes is a container orchestration system. We declare the desired state, such as three application Pods. Kubernetes schedules Pods, watches their state, and uses controllers to correct differences. It does not guarantee an application has no bugs or that every request succeeds.

## Why We Need It
With plain Docker, someone must decide where to run containers and replace them when a machine fails. Kubernetes automates much of that work across nodes.

## How It Works
```text
kubectl apply -> API server -> stored state in etcd
                      |
            Controller notices missing Pods
                      |
             Scheduler chooses a node
                      |
               Kubelet on that node
                      |
             Container runtime starts Pod
```

| Component | Job | When to suspect it |
|---|---|---|
| API server | Receives and validates API requests | `kubectl` cannot talk to cluster |
| etcd | Stores cluster state | Control plane cannot reliably read/write state |
| Controller manager | Runs controllers that reconcile resources | Desired and actual resources differ |
| Scheduler | Picks node for unscheduled Pods | Pod stays Pending with scheduling events |
| Kubelet | Manages Pods assigned to its node | Node/Pod startup issues |
| containerd (runtime) | Starts containers | Image/runtime startup errors |

**Important:** The scheduler selects a node; it does not directly start the container. The kubelet and runtime do that.

## Working Example
Create a simple Deployment:
```bash
kubectl create deployment web --image=nginx:1.27 --replicas=3 -n dev
kubectl get deploy,rs,pods -n dev -o wide
```
If `dev` does not exist, create it first: `kubectl create namespace dev`. Delete a Deployment-owned Pod and watch a replacement appear:
```bash
kubectl delete pod <web-pod-name> -n dev
kubectl get pods -n dev -w
```
Cleanup: `kubectl delete deployment web -n dev`.

## What to Learn
- `kubectl get nodes -o wide`: check node health and IPs.
- `kubectl get pods -A`: find workloads across namespaces.
- `kubectl describe pod`: read scheduling and startup events.
- Desired state vs actual state; scheduler vs kubelet vs controller.

## Real-World Example
A release requests five Pods, but only three run. First check the Deployment's desired/available count. Then describe the Pending Pods. If events say `Insufficient memory`, the scheduler cannot place them. Restarting Nginx will not solve a node-capacity problem.

## Troubleshooting
**Symptom: Pod remains Pending.**
```bash
kubectl get pods -n dev -o wide
kubectl describe pod <pod> -n dev
kubectl get nodes -o wide
```
Read Events at the bottom. If there is a scheduling error, investigate node capacity, selectors, or taints. If it is waiting for an image, check image name and pull events instead.

## Interview Questions
**Q: What happens after `kubectl apply`?**  
A: API server validates and stores the resource. A controller notices desired Pods, the scheduler assigns them to nodes, and kubelets start containers through the runtime.

**Q: Scheduler vs kubelet?**  
A: Scheduler chooses the node; kubelet manages Pods on that node.

**Q: If a Pod is deleted, who replaces it?**  
A: Its controller (such as a ReplicaSet) creates a replacement Pod. A standalone Pod has no such controller.

**Q: Does Running mean the app is healthy?**  
A: No. Check readiness and application behavior.

## Quick Revision
- API server = entry point; etcd = stored cluster state.
- Controller = notices and corrects differences.
- Scheduler = chooses node; kubelet = runs assigned Pods.
- Start troubleshooting with `get`, then `describe`, then logs.
