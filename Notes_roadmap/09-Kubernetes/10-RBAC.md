---
tags: [kubernetes, rbac, eks, sre, cka]
---
# Kubernetes RBAC

> **Remember:** Authentication = **Who are you?** Authorization = **What can you do?**

## Concept
RBAC (Role-Based Access Control) controls which Kubernetes API actions users, groups, and ServiceAccounts can perform.

## Definition
RBAC is a Kubernetes authorization mechanism. **Roles** and **ClusterRoles** define permissions. **RoleBindings** and **ClusterRoleBindings** give those permissions to identities.

| Resource | Purpose |
|---|---|
| Role | Permission rules within a namespace |
| ClusterRole | Reusable rules, including cluster-scoped resources |
| RoleBinding | Grants Role or ClusterRole permissions within one namespace |
| ClusterRoleBinding | Grants ClusterRole permissions cluster-wide |
| ServiceAccount | Identity used by workloads |

**Memory trick:** Role = **WHAT**, Binding = **WHO**, Namespace = **WHERE**.

## Why We Need It
A developer may need to read Pods in `dev`, but should not be able to delete workloads in `production`. RBAC helps enforce **least privilege**.

## How It Works
```text
kubectl get pods -n dev
        |
        v
Kubernetes API server
        |
        v
Authentication: who is calling?
        |
        v
Authorization: may this identity LIST Pods in dev?
        |
        +-- Allowed -> request continues
        +-- Denied  -> Forbidden (403)
```
RBAC permissions are **additive**. It has no explicit deny rules.

## YAML Example — Read Pods and Logs in dev

**Role (`role.yaml`):**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
```

**RoleBinding (`rolebinding.yaml`):**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-pod-reader
  namespace: dev
subjects:
  - kind: User
    name: developer1
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Field meanings:**
- `apiGroups: [""]` = core API group (Pods belong here).
- `resources` = resource or subresource the rule applies to.
- `verbs` = allowed operations; `get` and `list` are different.
- `pods/log` = separate permission needed to read Pod logs.
- `subjects` = who receives permissions.
- `roleRef` = which Role or ClusterRole is granted.

A RoleBinding **does not create a user**. `developer1` must match the authenticated Kubernetes username.

## Role vs ClusterRole
A **Role** defines permissions in one namespace. A **ClusterRole** can define permissions for cluster-scoped resources (such as Nodes) or namespace-scoped resources (such as Pods).

```text
ClusterRole + RoleBinding        -> permission in binding's namespace
ClusterRole + ClusterRoleBinding -> permission across the cluster
```

Creating a Role or ClusterRole alone does **not** grant access.

## ServiceAccount Example
A workload can use a ServiceAccount as its Kubernetes API identity.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitoring-sa
  namespace: dev
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: monitoring-pod-reader
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: monitoring-sa
    namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

A Pod can select this identity inside its `spec`:
```yaml
spec:
  serviceAccountName: monitoring-sa
  containers:
    - name: app
      image: nginx:1.27
```
The ServiceAccount must exist in the same namespace as the Pod.

## What to Learn — Commands
```bash
kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml
kubectl get roles,rolebindings -n dev
kubectl get clusterroles
kubectl get clusterrolebindings
kubectl get serviceaccounts -n dev
kubectl describe role pod-reader -n dev
kubectl describe rolebinding developer-pod-reader -n dev

# Check your current permissions
kubectl auth can-i list pods -n dev
kubectl auth can-i delete pods -n production
kubectl auth can-i get pods --subresource=log -n dev

# Test another identity; requires impersonation permission
kubectl auth can-i list pods --as=developer1 -n dev
kubectl auth can-i list pods --as=system:serviceaccount:dev:monitoring-sa -n dev
```

## RBAC on Amazon EKS
EKS has **AWS IAM authentication/access management** in addition to Kubernetes RBAC.

```text
IAM user / role
      |
      v
EKS authentication and identity mapping
      |
      v
Kubernetes user / groups
      |
      v
Authorization: Kubernetes RBAC and/or EKS access policies
```

- **IAM** = AWS identity and AWS permissions.
- **EKS access entry** = connects an IAM principal to EKS cluster access and may associate Kubernetes groups or EKS access policies.
- **Kubernetes RBAC** = roles and bindings inside Kubernetes.
- **EKS access policies** = EKS-managed authorization, not Kubernetes Role objects.
- Older clusters may use the `aws-auth` ConfigMap; newer clusters can use access entries. Check the authentication mode.

**Commands from your EKS practice:**
```bash
aws sts get-caller-identity
aws eks update-kubeconfig --region ap-south-1 --name learn-kratos
kubectl get nodes
aws eks describe-cluster --name learn-kratos --region ap-south-1 \
  --query 'cluster.accessConfig.authenticationMode'
aws eks list-access-entries --cluster-name learn-kratos --region ap-south-1
```
These AWS CLI commands require appropriate IAM permissions. `aws sts get-caller-identity` working does **not** prove Kubernetes access.

## Real-World Example — Forbidden in Production
**Problem:**
```text
Error from server (Forbidden):
pods is forbidden: User "developer1" cannot list resource "pods"
in API group "" in the namespace "production"
```

**Investigate:**
```bash
kubectl auth can-i list pods --as=developer1 -n production
kubectl get rolebindings -n production
kubectl describe rolebinding <binding-name> -n production
kubectl describe role <role-name> -n production
```
Possible causes: wrong identity, missing binding, missing `list` verb, or binding in the wrong namespace. Confirm approved access before changing anything. **Do not grant `cluster-admin` just to fix Forbidden.**

## Troubleshooting

### Forbidden (403)
Authentication likely succeeded, but authorization denied the action. Check the **identity, verb, resource, and namespace**. Inspect Roles and bindings.

### Unauthorized or credentials error
Authentication may have failed. On EKS, check:
```bash
aws sts get-caller-identity
aws eks update-kubeconfig --region ap-south-1 --name learn-kratos
kubectl config current-context
```
Then inspect EKS access entries and authentication mode if needed.

### Can list Pods but cannot read logs
Pod logs use the `pods/log` subresource. Add `get` on `pods/log` to an appropriate Role.

### Works in dev but not production
A RoleBinding in `dev` does not grant permissions in `production`.

### `kubectl auth can-i --as` fails
Your current identity may lack **impersonation** permission; that is separate from the tested user's permissions.

## Interview Questions

**What is RBAC?**  
Kubernetes authorization that controls API operations for users, groups, and ServiceAccounts.

**Role vs ClusterRole?**  
A Role defines permissions within one namespace. A ClusterRole can define reusable rules or rules for cluster-scoped resources.

**RoleBinding vs ClusterRoleBinding?**  
A RoleBinding grants access within one namespace. A ClusterRoleBinding grants access cluster-wide.

**Can a RoleBinding reference a ClusterRole?**  
Yes. It grants that ClusterRole's namespace-scoped permissions only in the binding's namespace.

**Why can a user list Pods but not read logs?**  
Reading logs needs `get` permission on the `pods/log` subresource.

**How is EKS IAM different from RBAC?**  
IAM identifies AWS principals and controls AWS API access. EKS authenticates/maps the identity. Kubernetes RBAC and/or EKS access policies authorize Kubernetes actions.

**How do you troubleshoot Forbidden?**  
Identify user, verb, resource, namespace; check `kubectl auth can-i`; inspect bindings and roles; apply least-privilege changes if approved.

## Quick Revision
- Authentication = **Who are you?**
- Authorization = **What can you do?**
- Role = namespace permissions.
- ClusterRole = reusable or cluster-scoped rules.
- RoleBinding = grant permissions in a namespace.
- ClusterRoleBinding = grant permissions cluster-wide.
- ServiceAccount = workload identity.
- `kubectl auth can-i` = test access.
- `pods/log` = permission for logs.
- EKS access entry is **not** a RoleBinding.
- Least privilege = only required permissions.

> **Remember:** Role = **WHAT**, Binding = **WHO**, Namespace = **WHERE**.
