# kubectl Cheat Sheet

Tags: #kubernetes #kubectl #sre #cka

Keep adding commands as you practice. Replace `<...>` placeholders.

## Cluster and Nodes
```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl describe node <node-name>
```

## Namespaces
```bash
kubectl get ns
kubectl get pods -n dev
kubectl get pods -A
```

## Pods
```bash
kubectl get pods -n dev -o wide
kubectl describe pod <pod-name> -n dev
kubectl get pods -n dev --show-labels
kubectl get pods -n dev -l app=httpd
kubectl delete pod <pod-name> -n dev
```

## Logs
```bash
kubectl logs <pod-name> -n dev
kubectl logs -f <pod-name> -n dev
kubectl logs <pod-name> -c <container-name> -n dev
kubectl logs <pod-name> -n dev --previous
```

## Execute Commands in Containers
```bash
kubectl exec -it <pod-name> -n dev -- /bin/sh
kubectl exec -it <pod-name> -n dev -- /bin/bash
kubectl exec -n dev <pod-name> -- cat /etc/resolv.conf
```
**Remember:** Options such as `-n dev` go **before** `--`.

## YAML and Validation
```bash
kubectl apply -f manifest.yaml
kubectl apply --dry-run=server -f manifest.yaml
kubectl explain pod.spec.containers
kubectl run demo --image=nginx --dry-run=client -o yaml
kubectl create deployment demo --image=nginx --dry-run=client -o yaml
```

## ReplicaSets and DaemonSets
```bash
kubectl get rs -n dev
kubectl describe rs <name> -n dev
kubectl scale rs <name> --replicas=3 -n dev
kubectl get ds -A
```

## Deployments and Rollouts
```bash
kubectl get deployments -n production
kubectl get rs -n production
kubectl rollout status deployment/<name> -n production
kubectl rollout history deployment/<name> -n production
kubectl rollout undo deployment/<name> -n production
```

## Services and DNS
```bash
kubectl get svc -n dev
kubectl describe svc cluster-ip -n dev
kubectl get endpoints cluster-ip -n dev
kubectl get endpointslices -n dev -l kubernetes.io/service-name=cluster-ip
kubectl port-forward -n dev svc/cluster-ip 8080:80
```

## Troubleshooting Order
```text
1. Correct cluster and namespace?
2. Resource exists?
3. Pod READY and STATUS?
4. Events in kubectl describe?
5. Container logs?
6. Labels match selectors?
7. Service endpoints and ports correct?
8. DNS and network path working?
```

## LoadBalancer / NodePort
```bash
kubectl get svc -n dev
kubectl describe svc <service-name> -n dev
kubectl get nodes -o wide
kubectl get endpointslices -n dev -l kubernetes.io/service-name=<service-name>
minikube service <service-name> -n dev --url
```
