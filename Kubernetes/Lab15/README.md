<img width="2553" height="834" alt="image" src="https://github.com/user-attachments/assets/b52f8da6-9c55-44ac-8d44-04faef051895" /># Lab 15: Node.js Application Deployment with ClusterIP Service

## Objective

Deploy the custom Node.js application image built in Lab 8 to a Kubernetes cluster as a Deployment with two replicas, connect it to the MySQL StatefulSet from Lab 14, and expose it internally through a ClusterIP Service.

The lab combines concepts from the previous labs:

| Concept | Source Lab | Usage in This Lab |
|---|---|---|
| Taints and labels | Lab 10 | Pods are scheduled only on the dedicated worker node |
| Namespace and ResourceQuota | Lab 11 | All resources live in the `ivolve` namespace |
| ConfigMap and Secret | Lab 12 | Provide database settings and credentials to the pods |
| PersistentVolume and PVC | Lab 13 | Persist application logs |
| StatefulSet and headless Service | Lab 14 | MySQL backend that the application connects to |

---

## Architecture

```
                        Namespace: ivolve
+------------------------------------------------------------------+
|                                                                  |
|   ClusterIP Service: ivolveappsvc (port 3000)                    |
|                  |                                               |
|          +-------+--------+                                      |
|          v                v                                      |
|   +--------------+  +--------------+        Node: worker-1       |
|   | ivolveapp    |  | ivolveapp    |        (label project=ivolve|
|   | pod 1        |  | pod 2        |         taint workload=      |
|   |              |  |              |         ivolveApp:NoSchedule)|
|   | init-db      |  | init-db      |                              |
|   | Node.js app  |  | Node.js app  |                              |
|   +------+-------+  +------+-------+                              |
|          |                 |                                      |
|          |  /app/logs      |                                      |
|          +--------+--------+                                      |
|                   v                                               |
|        PVC ivolvepvc -> PV ivolvepv (hostPath /mnt/app-logs)      |
|                                                                  |
|          |  DB_HOST (ConfigMap) + credentials (Secret)            |
|          v                                                        |
|   Headless Service: ivolvedbsvc (port 3306)                       |
|          |                                                        |
|   StatefulSet pod: ivolvedb-0 (MySQL 8.0)                         |
+------------------------------------------------------------------+
```

---

## Tools and Technologies

| Tool | Purpose |
|---|---|
| Kubernetes (kubeadm cluster, v1.28) | Container orchestration platform |
| kubectl | Cluster management and verification |
| Docker Hub | Registry hosting the custom application image from Lab 8 |
| Node.js | Application runtime |
| MySQL 8.0 | Database backend and MySQL client used by the init container |
| Calico | Pod networking |
| YAML | Declarative resource definitions |

---

## Prerequisites

The following resources must exist before starting. They were created in earlier labs.

| Resource | Name | Namespace |
|---|---|---|
| Namespace | `ivolve` | cluster-wide |
| ConfigMap | `ivolve-cm` (keys: `DB_HOST`, `DB_USER`) | `ivolve` |
| Secret | `ivolve-secret` (keys: `MYSQL_ROOT_PASSWORD`, `DB_PASSWORD`) | `ivolve` |
| PersistentVolume | `ivolvepv` (1Gi, RWX, hostPath `/mnt/app-logs`, Retain) | cluster-wide |
| StatefulSet | `ivolvedb` (pod `ivolvedb-0`) | `ivolve` |
| Headless Service | `ivolvedbsvc` | `ivolve` |
| Worker node | `worker-1` with label `project=ivolve` and taint `workload=ivolveApp:NoSchedule` | cluster-wide |

Note: Kubernetes object names must be lowercase (RFC 1123), so the names above are written in lowercase even though the lab sheet uses mixed case.

### Step 0: Verify the Existing Resources

```bash
kubectl get ns ivolve
kubectl get cm,secret,pvc,pv -n ivolve
kubectl get pods -n ivolve -o wide
kubectl get nodes --show-labels
```

Explanation: the Deployment depends on every resource listed above. A missing ConfigMap, Secret, or PVC leaves the pod in `Pending` or `CreateContainerConfigError`, and the MySQL pod must be running because the init container connects to it.

<img width="2553" height="834" alt="image" src="https://github.com/user-attachments/assets/fe30659b-7dc1-4e88-9ca5-cd45d4211c05" />

---

## Implementation Steps

### Step 1: Create the PersistentVolumeClaim in the `ivolve` Namespace

A PVC is a namespaced resource, so the claim must live in the same namespace as the pods that mount it. The existing PV was in `Released` state because it was bound to a claim in another namespace that had been deleted. With the `Retain` reclaim policy, the old `claimRef` stays on the PV and blocks new claims, so it is cleared first.

```bash
kubectl patch pv ivolvepv -p '{"spec":{"claimRef":null}}'
kubectl get pv ivolvepv
```

`ivolvepvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ivolvepvc
  namespace: ivolve
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ""
  volumeName: ivolvepv
```

```bash
kubectl apply -f ivolvepvc.yaml
kubectl get pvc -n ivolve
```

Explanation:

- `storageClassName: ""` disables dynamic provisioning and prevents a default StorageClass from being assigned, as the lab requires no StorageClass.
- `volumeName: ivolvepv` binds the claim to the specific PV from Lab 13.
- The access mode matches the PV (`ReadWriteMany`).

The hostPath directory must also exist on the worker node:

```bash
ssh worker-1
sudo mkdir -p /mnt/app-logs && sudo chmod 777 /mnt/app-logs
```

<img width="1447" height="238" alt="image" src="https://github.com/user-attachments/assets/9befeec0-f614-4f96-8dae-cdb02e816c70" />

### Step 2: Create the Deployment

`ivolveapp-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ivolveapp
  namespace: ivolve
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ivolveapp
  template:
    metadata:
      labels:
        app: ivolveapp
    spec:
      nodeSelector:
        project: ivolve
      tolerations:
        - key: workload
          operator: Equal
          value: ivolveApp
          effect: NoSchedule
      initContainers:
        - name: init-db
          image: mysql:8.0
          envFrom:
            - configMapRef:
                name: ivolve-cm
            - secretRef:
                name: ivolve-secret
          command: ["/bin/sh", "-c"]
          args:
            - |
              until mysql -h "$DB_HOST" -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT 1"; do
                echo "Waiting for MySQL..."; sleep 3
              done
              mysql -h "$DB_HOST" -uroot -p"$MYSQL_ROOT_PASSWORD" <<EOF
              CREATE DATABASE IF NOT EXISTS ivolve;
              CREATE USER IF NOT EXISTS '${DB_USER}'@'%' IDENTIFIED BY '${DB_PASSWORD}';
              GRANT ALL PRIVILEGES ON ivolve.* TO '${DB_USER}'@'%';
              FLUSH PRIVILEGES;
              EOF
      containers:
        - name: ivolveapp
          image: <DOCKERHUB_USERNAME>/<IMAGE_NAME>:<TAG>
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: ivolve-cm
            - secretRef:
                name: ivolve-secret
          volumeMounts:
            - name: app-logs
              mountPath: /app/logs
      volumes:
        - name: app-logs
          persistentVolumeClaim:
            claimName: ivolvepvc
```

Explanation of each section:

| Section | Purpose |
|---|---|
| `replicas: 2` | Two identical application pods for availability and load distribution |
| `selector` and `labels` | The Deployment manages pods labeled `app=ivolveapp`; the Service uses the same label |
| `nodeSelector` | Restricts scheduling to nodes labeled `project=ivolve` |
| `tolerations` | Allows the pods onto the node tainted `workload=ivolveApp:NoSchedule` |
| `initContainers` | Runs to completion before the application starts. It waits until MySQL accepts connections, creates the `ivolve` database, creates the application user, and grants it privileges on that database |
| `envFrom` | Injects every key from the ConfigMap and Secret as environment variables, for both the init container and the application |
| `volumeMounts` and `volumes` | Mounts the PVC at `/app/logs` so application logs persist on the node |

The statements use `IF NOT EXISTS`, which makes the init container idempotent, so a pod restart does not cause it to fail.

```bash
kubectl apply -f ivolveapp-deployment.yaml
```

### Step 3: Create the ClusterIP Service

`ivolveappsvc.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ivolveappsvc
  namespace: ivolve
spec:
  type: ClusterIP
  selector:
    app: ivolveapp
  ports:
    - port: 3000
      targetPort: 3000
```

```bash
kubectl apply -f ivolveappsvc.yaml
```

Explanation: pod IP addresses change whenever a pod is recreated. The Service provides a stable virtual IP and DNS name (`ivolveappsvc.ivolve.svc.cluster.local`) and load balances traffic across all pods matching the `app=ivolveapp` label. The `ClusterIP` type makes it reachable only from inside the cluster.

### Step 4: Verify the Rollout

```bash
kubectl rollout status deployment/ivolveapp -n ivolve
kubectl get pods -n ivolve -o wide
kubectl get svc,endpoints -n ivolve
```

Expected result:

- The rollout completes successfully.
- Both `ivolveapp` pods are `Running 1/1` on `worker-1`, which confirms that the nodeSelector and toleration work.
- The `ivolveappsvc` endpoints list the IP addresses of both pods on port 3000.

<img width="2070" height="300" alt="image" src="https://github.com/user-attachments/assets/bbadbffb-9768-4b26-b8c9-18b1623174be" />

### Step 5: Verify the Init Container

```bash
kubectl logs -n ivolve -l app=ivolveapp
```

Explanation: the init container logs show that it connected to MySQL as root and executed the database and user setup without errors.

<img width="1153" height="294" alt="image" src="https://github.com/user-attachments/assets/fe5490ad-6b40-4613-ac26-83154ec7a4bc" />

### Step 6: Verify the Database and Privileges Manually

```bash
kubectl exec -it ivolvedb-0 -n ivolve -- mysql -uroot -p -e "SHOW DATABASES; SHOW GRANTS FOR 'ivolve_user'@'%';"
```

Expected result:

- The `ivolve` database appears in the database list.
- The application user holds `ALL PRIVILEGES` on `ivolve`.*

```
+---------------------------------------------------------+
| Grants for ivolve_user@%                                |
+---------------------------------------------------------+
| GRANT USAGE ON *.* TO `ivolve_user`@`%`                 |
| GRANT ALL PRIVILEGES ON `ivolve`.* TO `ivolve_user`@`%` |
+---------------------------------------------------------+
```

<img width="2394" height="589" alt="image" src="https://github.com/user-attachments/assets/48d217b0-5554-47cd-a4cb-721488b04ebf" />

### Step 7: Verify the Application Logs

```bash
kubectl logs deployment/ivolveapp -n ivolve
```

Expected output:

```
Attempting to connect to MySQL...
Connected to MySQL and 'ivolve' DB found.
Server started on http://0.0.0.0:3000
```

Explanation: this confirms the whole chain works end to end: ConfigMap and Secret injection, the init container, the database, and the network path to the headless service.

<img width="1399" height="202" alt="image" src="https://github.com/user-attachments/assets/84aa1234-19df-479f-a065-a5192a93cbba" />

### Step 8: Test the Service

The `ivolve` namespace is at its ResourceQuota limit (3 pods), so a temporary test pod cannot be created there. The Service is tested from the control plane node instead.

```bash
kubectl get svc ivolveappsvc -n ivolve
curl http://<CLUSTER-IP>:3000
```
<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/1a985362-0fbc-4a1e-9f1f-7454aba3397f" />
<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/e2ddbb5f-239a-47f3-8edd-4ed792b0067c" />

### Step 9: Verify Log Persistence on the Node

On `worker-1`:

```bash
ls -la /mnt/app-logs
```

Explanation: the application log files (`app.log` and `access.log`) appear on the node filesystem, which proves that the PVC and hostPath volume are mounted and that logs survive pod restarts.

<img width="1213" height="255" alt="image" src="https://github.com/user-attachments/assets/9b88b3a1-191c-4ccc-b29d-f2163980063d" />

---

## Verification Summary

| Check | Command | Result |
|---|---|---|
| Deployment rollout | `kubectl rollout status` | Successfully rolled out |
| Pods running on dedicated node | `kubectl get pods -o wide` | 2/2 Running on `worker-1` |
| Service endpoints | `kubectl get endpoints ivolveappsvc` | Two pod IPs on port 3000 |
| Database created | `SHOW DATABASES` | `ivolve` present |
| User privileges | `SHOW GRANTS` | `ALL PRIVILEGES` on `ivolve`.* |
| Application connectivity | `kubectl logs` | Connected to MySQL and `ivolve` DB found |
| Log persistence | `ls /mnt/app-logs` on `worker-1` | `app.log` and `access.log` present |

---

## Conclusion

In this lab, the Node.js application from Lab 8 was deployed as a two-replica Deployment in the `ivolve` namespace and exposed through the `ivolveappsvc` ClusterIP Service. The pods are scheduled on the dedicated worker node using a nodeSelector and a toleration, receive their configuration from a ConfigMap and a Secret, and store logs on a hostPath-backed PersistentVolume.

An init container prepares the database by creating the `ivolve` database and granting the application user the required privileges before the application starts, which removes manual database setup from the deployment process. Verification confirmed that the database and privileges exist, the application connects successfully, the Service load balances across both pods, and logs persist on the node.

Troubleshooting the namespace mismatches, ResourceQuota limits, DNS naming, and credential issues reinforced how the Kubernetes components from the previous labs depend on one another.
