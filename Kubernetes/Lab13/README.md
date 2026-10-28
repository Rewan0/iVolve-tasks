# Lab 13: Persistent Storage Setup for Application Logging

> Static provisioning of a **PersistentVolume (hostPath)** + **PersistentVolumeClaim** on Kubernetes, without any StorageClass.

---

## Objective

| Resource | Name | Size | Access Mode | Notes |
|----------|------|------|-------------|-------|
| PV | `ivolvepv` | 1Gi | ReadWriteMany | `hostPath: /mnt/app-logs`, Reclaim = **Retain** |
| PVC | `ivolvepvc` | 1Gi | ReadWriteMany | No StorageClass (`storageClassName: ""`) |

> Kubernetes object names must be lowercase, so `ivolvePV` / `ivolvePVC` became `ivolvepv` / `ivolvepvc`.

---

## Tools and Technologies

| Category | Tool | Purpose |
|----------|------|---------|
| Container Orchestration | Kubernetes | Cluster running the lab (1 master + 1 worker) |
| CLI | kubectl | Create and inspect cluster resources |
| Storage | PersistentVolume / PersistentVolumeClaim | Static provisioning of persistent storage |
| Volume Type | hostPath | Backing storage on the worker node filesystem |
| Networking (CNI) | Calico | Pod networking |
| Container Image | busybox | Lightweight test Pod for writing logs |
| Configuration | YAML | Declarative resource definitions |
| Operating System | Ubuntu Linux | Master and worker nodes |
| Editor | vim | Writing manifests |

---

## Concepts

| Term | Meaning |
|------|---------|
| **PV** | Cluster-level storage resource (the "disk") |
| **PVC** | A request for storage by a user/Pod |
| **hostPath** | Uses a folder on the node's filesystem (labs only, not production) |
| **RWX** | Many Pods can read/write at the same time |
| **Retain** | PV and its data are kept after the PVC is deleted |
| **No StorageClass** | Static provisioning: PVC binds to PV by matching size + access mode |

```
┌──────────────┐   binds   ┌──────────────┐   mounts   ┌──────────────┐
│  Pod         │──────────▶│  PVC         │───────────▶│  PV          │
│  /logs       │           │  ivolvepvc   │            │  ivolvepv    │
└──────────────┘           └──────────────┘            └──────┬───────┘
                                                              │ hostPath
                                                       ┌──────▼───────┐
                                                       │ worker-1     │
                                                       │ /mnt/app-logs│
                                                       └──────────────┘
```

---

## Steps

### Step 1: Create the folder on the worker node (`worker-1`)

```bash
sudo mkdir -p /mnt/app-logs
sudo chmod 777 /mnt/app-logs
```

<img width="1203" height="97" alt="image" src="https://github.com/user-attachments/assets/4fa254df-2191-4945-a1a7-ee0b4fc1123a" />


### Step 2: Create the PV (`master-1`)

`ivolve-pv.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ivolvepv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath:
    path: /mnt/app-logs
```

### Step 3: Create the PVC (`master-1`)

`ivolve-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ivolvepvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ""
```

> `storageClassName: ""` is essential, otherwise the default StorageClass may be assigned and the PVC stays `Pending`.

### Step 4: Apply both files (`master-1`)

```bash
kubectl apply -f ivolve-pv.yaml
kubectl apply -f ivolve-pvc.yaml
```

```
persistentvolume/ivolvepv created
persistentvolumeclaim/ivolvepvc created
```

<img width="1215" height="877" alt="image" src="https://github.com/user-attachments/assets/a23f86dc-cc07-4c63-b931-6bc6ed8fd1d4" />


### Step 5: Verify the binding (`master-1`)

```bash
kubectl get pv
kubectl get pvc
```

```
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   AGE
ivolvepv   1Gi        RWX            Retain           Bound    default/ivolvepvc                  12s

NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ivolvepvc   Bound    ivolvepv   1Gi        RWX                           12s
```

Both are **Bound**

<img width="2094" height="181" alt="image" src="https://github.com/user-attachments/assets/184ff365-6205-40b0-ba8e-7aaf50e52fd3" />

### Step 6: Test with a Pod (`master-1`)

`test-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "echo 'test log' >> /logs/app.log; sleep 3600"]
      volumeMounts:
        - name: logs
          mountPath: /logs
  volumes:
    - name: logs
      persistentVolumeClaim:
        claimName: ivolvepvc
```

```bash
kubectl apply -f test-pod.yaml
kubectl get pod test-pod -o wide
```

```
NAME       READY   STATUS    RESTARTS   AGE   IP              NODE
test-pod   1/1     Running   0          28s   10.244.226.78   worker-1
```

<img width="1705" height="156" alt="image" src="https://github.com/user-attachments/assets/f7a72e3f-e108-437f-af24-b03a8ad5a250" />

### Step 7: Verify the data (`master-1` and `worker-1`)

```bash
# On master-1
kubectl exec test-pod -- cat /logs/app.log

# On worker-1
cat /mnt/app-logs/app.log
```

```
test log
test log
```

The log written inside the container exists on the node's filesystem, so the storage is persistent

<img width="1038" height="100" alt="image" src="https://github.com/user-attachments/assets/e1117e95-54e3-4ee4-9ba7-147a4eadb36b" />
<img width="832" height="99" alt="image" src="https://github.com/user-attachments/assets/bffbfb43-8458-4029-9faf-9f44b3be1313" />

---

## Troubleshooting (issues faced)

| Problem | Cause | Fix |
|---------|-------|-----|
| `pod does not have a host assigned` | Pod `Pending`: worker had taint `workload=ivolveApp` (left from Lab 10) | Remove the taint: `kubectl taint nodes worker-1 workload=ivolveApp:NoSchedule-` |
| `ContainerCreating` + `calico ... Unauthorized` | Calico CNI on the worker had an invalid token/kubeconfig | Restart `calico-node` (namespace `calico-system`), then recreate the Pod |

```bash
kubectl rollout restart daemonset calico-node -n calico-system
kubectl delete pod test-pod && kubectl apply -f test-pod.yaml
```

---

## Conclusion

This lab demonstrated static provisioning of persistent storage in Kubernetes: a hostPath PV (ivolvepv) and a PVC (ivolvepvc) were bound without a StorageClass. Logs written by a Pod to /logs were stored on the worker node at /mnt/app-logs and survived Pod deletion and recreation. Because hostPath is node-local, it is suitable for labs only, and production environments should use network or cloud storage.
