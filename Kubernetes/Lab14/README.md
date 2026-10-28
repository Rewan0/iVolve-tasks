# Lab 14: StatefulSet with Headless Service

Deploying a stateful MySQL database on Kubernetes using a StatefulSet, a headless Service, and persistent storage, with node placement controlled through a node selector and a toleration.

---

## Objectives

- Create a StatefulSet named `ivolvedb` in the `ivolve` namespace with one MySQL replica.
- Inject configuration from a ConfigMap and a Secret as environment variables.
- Schedule the pod only on nodes labeled `project=ivolve`.
- Add a toleration for `workload=ivolveApp` with effect `NoSchedule`.
- Provision storage through a PVC template mounted at `/var/lib/mysql`.
- Expose the pod through a headless Service named `ivolvedbsvc`.
- Verify database connectivity with a MySQL client.
- Verify DNS resolution of the pod inside the cluster.

> Kubernetes resource names must be lowercase (RFC 1123), so `ivolveDB` and `ivolveDBSvc` are implemented as `ivolvedb` and `ivolvedbsvc`.

## Tools and Technologies

| Category | Tool | Purpose |
|---|---|---|
| Container orchestration | Kubernetes v1.28 (kubeadm) | Cluster running the workload |
| CLI | kubectl | Cluster management and verification |
| Workload API | StatefulSet | Stable identity and storage for the database pod |
| Networking | Headless Service, CoreDNS | Per-pod DNS records |
| Configuration | ConfigMap, Secret | Non-sensitive and sensitive environment variables |
| Storage | PersistentVolume (hostPath), PVC template | Persistent database storage |
| Scheduling | nodeSelector, taints, tolerations | Controlled pod placement |
| Container runtime | containerd | Runs the containers |
| Database | MySQL 8.0 | Stateful application |
| Testing | busybox 1.36 (nslookup), mysql:8.0 client | DNS and connectivity checks |
| Infrastructure | VMware Workstation, Ubuntu Server | Virtual machines hosting the cluster |

## Architecture

```
                 Namespace: ivolve
   +----------------------------------------------------+
   |                                                    |
   |   Headless Service: ivolvedbsvc (clusterIP: None)  |
   |                      |                             |
   |                      v                             |
   |   StatefulSet: ivolvedb                            |
   |      Pod: ivolvedb-0  (mysql:8.0)                  |
   |        envFrom: ConfigMap + Secret                 |
   |        nodeSelector: project=ivolve                |
   |        toleration:   workload=ivolveApp:NoSchedule |
   |                      |                             |
   |                      v                             |
   |   PVC: data-ivolvedb-0  --->  PV: mysql-pv         |
   |   mounted at /var/lib/mysql   (hostPath)           |
   +----------------------------------------------------+
                          |
                   Node: worker-1
        label: project=ivolve
        taint: workload=ivolveApp:NoSchedule
```

## Implementation Steps

### Step 1: Prepare the namespace and the node

```bash
kubectl create ns ivolve
kubectl label node worker-1 project=ivolve
kubectl taint node worker-1 workload=ivolveApp:NoSchedule
```

The label allows the `nodeSelector` to match the worker. The taint repels any pod that does not carry a matching toleration.

<img width="1183" height="207" alt="image" src="https://github.com/user-attachments/assets/9f6cdcd7-8d9c-44ac-a041-06a83972531f" />


### Step 2: Create the ConfigMap and the Secret

```bash
kubectl create configmap mysql-config -n ivolve \
  --from-literal=MYSQL_DATABASE=ivolve_db

kubectl create secret generic mysql-secret -n ivolve \
  --from-literal=MYSQL_ROOT_PASSWORD=RootPass123 \
  --from-literal=MYSQL_USER=ivolve \
  --from-literal=MYSQL_PASSWORD=UserPass123
```

The ConfigMap holds non-sensitive settings (database name). The Secret holds credentials.

<img width="1033" height="214" alt="image" src="https://github.com/user-attachments/assets/00bb12a4-6ef1-4d4b-bdca-530e7388c741" />


### Step 3: Provision storage

The cluster has no default StorageClass, so a PersistentVolume backed by `hostPath` is created manually.

`pv.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 1Gi
  accessModes: ["ReadWriteOnce"]
  storageClassName: manual
  hostPath:
    path: /mnt/mysql-data
```

On the worker node:

```bash
sudo mkdir -p /mnt/mysql-data
```

Then on the control-plane node:

```bash
kubectl apply -f pv.yaml
```


### Step 4: Create the headless Service

`svc.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ivolvedbsvc
  namespace: ivolve
spec:
  clusterIP: None
  selector:
    app: ivolvedb
  ports:
    - port: 3306
      name: mysql
```

`clusterIP: None` makes the Service headless. The selector must match the pod labels defined in the StatefulSet.

```bash
kubectl apply -f svc.yaml
```

### Step 5: Create the StatefulSet

`sts.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ivolvedb
  namespace: ivolve
spec:
  serviceName: ivolvedbsvc
  replicas: 1
  selector:
    matchLabels:
      app: ivolvedb
  template:
    metadata:
      labels:
        app: ivolvedb
    spec:
      nodeSelector:
        project: ivolve
      tolerations:
        - key: "workload"
          operator: "Equal"
          value: "ivolveApp"
          effect: "NoSchedule"
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
              name: mysql
          envFrom:
            - configMapRef:
                name: mysql-config
            - secretRef:
                name: mysql-secret
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: manual
        resources:
          requests:
            storage: 1Gi
```

| Field | Purpose |
|---|---|
| `serviceName` | Binds the StatefulSet to the headless Service, which provides per-pod DNS. |
| `nodeSelector` | Restricts scheduling to nodes labeled `project=ivolve`. |
| `tolerations` | Allows scheduling on the node tainted `workload=ivolveApp:NoSchedule`. |
| `envFrom` | Loads all keys of the ConfigMap and the Secret as environment variables. |
| `volumeClaimTemplates` | Creates one PVC per replica (`data-ivolvedb-0`) mounted at `/var/lib/mysql`. |

```bash
kubectl apply -f sts.yaml
kubectl get sts,pods,pvc,svc -n ivolve -o wide
```

The pod reaches `Running`, and the PVC is `Bound` to `mysql-pv`.

<img width="1677" height="390" alt="image" src="https://github.com/user-attachments/assets/8ef0d587-9cc5-407f-b2a5-9112f6e33f59" />


## Verification

### Database connectivity

```bash
kubectl exec -it ivolvedb-0 -n ivolve -- \
  mysql -uroot -pRootPass123 -e "SHOW DATABASES;"
```

The output lists `ivolve_db`, confirming that the ConfigMap and Secret values were applied and that MySQL initialized correctly.


### DNS resolution

A temporary pod is used for the lookups. It carries the same toleration because the worker node is tainted.

```bash
kubectl run dns-test -n ivolve --rm -it --image=busybox:1.36 --restart=Never \
  --overrides='{"spec":{"tolerations":[{"key":"workload","operator":"Equal","value":"ivolveApp","effect":"NoSchedule"}]}}' \
  -- nslookup ivolvedb-0.ivolvedbsvc.ivolve.svc.cluster.local

kubectl run dns-test -n ivolve --rm -it --image=busybox:1.36 --restart=Never \
  --overrides='{"spec":{"tolerations":[{"key":"workload","operator":"Equal","value":"ivolveApp","effect":"NoSchedule"}]}}' \
  -- nslookup ivolvedbsvc.ivolve.svc.cluster.local
```

Both lookups return the pod IP (`10.244.226.81`). The Service name resolves directly to the pod address rather than to a virtual ClusterIP, which confirms the Service is headless.



### Connection through DNS from a separate pod

```bash
kubectl run mysql-client -n ivolve --rm -it --image=mysql:8.0 --restart=Never \
  --overrides='{"spec":{"tolerations":[{"key":"workload","operator":"Equal","value":"ivolveApp","effect":"NoSchedule"}]}}' \
  -- mysql -h ivolvedb-0.ivolvedbsvc -uroot -pRootPass123 -e "SELECT 1;"
```

The query returns `1`, confirming that the database is reachable through its stable DNS name.

<img width="2397" height="1335" alt="image" src="https://github.com/user-attachments/assets/7023794a-48a0-453d-b7ab-332cb8e2772b" />







## Conclusion

This lab demonstrated how to run a database as a stateful workload on Kubernetes. The StatefulSet provided a stable pod identity (`ivolvedb-0`) and a dedicated PVC through `volumeClaimTemplates`, so database files persist independently of the pod lifecycle. The headless Service exposed per-pod DNS records, allowing clients to reach a specific database instance by name instead of through a load-balanced virtual IP, which is the behavior stateful systems such as databases and clustered data stores require.

Configuration was separated from the container image by consuming a ConfigMap and a Secret through `envFrom`, keeping credentials out of the manifest. The node selector and toleration together demonstrated controlled scheduling: the pod is constrained to a labeled node and is permitted to run on a tainted one.

Practical issues encountered along the way, namely a namespace ResourceQuota, first-start database initialization time, taint effects on temporary test pods, and swap-related kubelet failures after a restart, reinforced the value of inspecting events, logs, and `kubectl describe` output when diagnosing cluster behavior.
