---
tags: [kubernetes, storage, pv, pvc, storageclass, aws, ebs, sre]
---
# 11 — Kubernetes Volumes and Storage

## 1. Concept
A Kubernetes **volume** provides a filesystem location that containers can mount. Container writable-layer data is not reliable across container recreation. **Not all volumes are persistent**; lifetime depends on the volume type.

## 2. Why It Is Needed
Share data between containers in a Pod, store temporary files, and preserve application data beyond Pod replacement using persistent storage.

## 3. Basic Volume Types

### `emptyDir`
Created when a Pod is assigned to a node; survives container restarts **within that Pod**, but its data is removed when the Pod is removed from the node.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
      volumeMounts:
        - name: scratch
          mountPath: /data
  volumes:
    - name: scratch
      emptyDir: {}
```

### `hostPath`
Mounts a path from the **worker node**. Data is tied to that node, not portable to another node. Host access creates security risks; avoid for general application persistence.

```yaml
volumes:
  - name: host-data
    hostPath:
      path: /mnt/app-data
      type: DirectoryOrCreate
```

## 4. PV, PVC, and Pod
- **PersistentVolume (PV):** cluster-scoped storage resource.
- **PersistentVolumeClaim (PVC):** namespaced request for storage.
- **Pod:** references the PVC, not usually the PV or cloud volume ID.

```text
Pod → PVC → PV → Storage backend (for example, AWS EBS)
```

A 2Gi PVC can bind to a 3Gi PV and report **3Gi capacity**, because it claims the whole PV. A 5Gi PV cannot satisfy an 8Gi PVC request.

## 5. Static Provisioning with AWS EBS
In static provisioning, create the EBS volume and PV yourself. The AWS EBS CSI driver must be installed and authorized.

### PV (existing EBS volume)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-pv
spec:
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-xxxxxxxxxxxxxxxxx
    fsType: ext4
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values: [ap-south-1a]
```

### PVC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: ""
  volumeName: test-pv
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

### Pod mounting the PVC
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ebs-demo-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: my-pvc
```

**Key fields:**
- `csi.driver`: storage plugin name.
- `volumeHandle`: existing EBS volume ID.
- `nodeAffinity`: PV's allowed storage topology.
- `storageClassName: ""`: explicitly requests no StorageClass; avoids default-class selection.
- `volumeName`: targets a specific PV.
- `volumeMounts.mountPath`: location inside container.
- `volumes[].persistentVolumeClaim.claimName`: PVC reference.

The scheduler considers PV topology. If no compatible node exists in the EBS AZ, the Pod cannot run successfully with that volume.

## 6. StorageClass and Dynamic Provisioning
A **StorageClass** describes how a provisioner should create storage. With dynamic provisioning, the CSI driver creates a volume when a PVC requests it.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
```

PVC using this class:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  storageClassName: ebs-gp3
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 3Gi
```

**Flow:** PVC → StorageClass → CSI provisioner → EBS volume + PV → Pod.

- `WaitForFirstConsumer`: waits for a consuming Pod so storage can be created in a suitable AZ.
- `Delete`: dynamically provisioned backing storage is normally deleted when the claim is deleted; **data-loss risk**.
- `allowVolumeExpansion`: enables expansion when supported by the driver and volume type.
- Standard EKS EBS CSI provisioner is `ebs.csi.aws.com`; **EKS Auto Mode uses a different provisioner**.

**Cost:** AWS EBS volumes incur charges even when Pods are stopped. Inspect and clean up lab resources carefully.

## 7. Access Modes
| Mode | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) | Read-write mount from one node; multiple Pods on that same node may use it |
| `ReadOnlyMany` (ROX) | Read-only from multiple nodes |
| `ReadWriteMany` (RWX) | Read-write from multiple nodes |
| `ReadWriteOncePod` (RWOP) | Single Pod read-write access cluster-wide, with supported CSI setup |

Access-mode support depends on the storage backend. Standard EBS volumes are generally used with RWO, not RWX.

## 8. Reclaim Policies
- **Retain:** underlying storage remains after PVC deletion; released PV needs manual handling before reuse.
- **Delete:** backing storage is deleted during reclaim; **potential permanent data loss**.

Deleting a Pod usually **does not** delete a separately created PVC. Do not delete PVCs/PVs just to troubleshoot mounting errors.

## 9. Important Commands
```bash
kubectl get pv
kubectl get pvc -A
kubectl describe pv test-pv
kubectl describe pvc my-pvc
kubectl get storageclass
kubectl get volumeattachments -o wide
kubectl describe pod ebs-demo-pod
kubectl exec ebs-demo-pod -- df -h /data
kubectl exec ebs-demo-pod -- cat /data/test.txt
```

For EKS (if AWS CLI permissions permit):
```bash
aws ec2 describe-volumes --volume-ids vol-xxxxxxxxxxxxxxxxx
```

## 10. Troubleshooting
### PVC Pending
Start with `kubectl describe pvc <claim>`:
- No matching PV, insufficient capacity, access-mode or class mismatch.
- CSI provisioner or IAM problem during dynamic provisioning.
- `WaitForFirstConsumer` may be waiting for a Pod.

### Pod ContainerCreating / FailedAttachVolume / FailedMount
```bash
kubectl describe pod <pod-name>
kubectl get volumeattachments -o wide
kubectl get pods -n kube-system | grep ebs-csi
```

Check: AZ mismatch, attachment state, CSI controller logs, and AWS IAM permissions. A manually created EBS volume may lack tags required by conditional IAM policies. **Do not delete VolumeAttachments, remove finalizers, or detach volumes blindly.**

### Data persistence test
```bash
kubectl exec ebs-demo-pod -- sh -c 'echo "EBS persistence test" > /data/test.txt'
kubectl exec ebs-demo-pod -- cat /data/test.txt
```

Recreate the Pod **without deleting its PVC or EBS volume**, then verify the file remains.

## 11. Interview Questions
1. **PV vs PVC?** PV offers storage; PVC requests it.
2. **Static vs dynamic provisioning?** Manual volume/PV creation vs provisioner-created storage from a PVC and StorageClass.
3. **Why `WaitForFirstConsumer`?** Align volume topology with the Pod's node placement.
4. **Does RWO mean one Pod?** No, it restricts read-write mounting to one node.
5. **Why does EBS AZ matter?** EBS volumes attach to compatible EC2 instances in the same AZ.
6. **Does deleting a Pod delete persistent data?** Normally not if the separately managed PVC and underlying volume remain.

## 12. Quick Revision
- `emptyDir`: Pod-lifetime temporary storage.
- `hostPath`: node-specific filesystem path.
- PV: storage resource; PVC: storage request.
- StorageClass: dynamic provisioning rules.
- CSI: storage driver integration.
- EBS: AZ-scoped AWS block storage.
- RWO: one node, not necessarily one Pod.
- `Retain`: preserve volume; `Delete`: remove backing volume during reclaim.
- First diagnostic: `kubectl describe pvc` or `kubectl describe pod` and inspect Events.
