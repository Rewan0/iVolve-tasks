# Lab 18: Node-Wide Pod Management with DaemonSet

## Objectives

- Create a `monitoring` namespace.
- Deploy Prometheus, Alertmanager and Grafana in the `monitoring` namespace.
- Deploy a DaemonSet for Prometheus `node-exporter`.
- Validate that a `node-exporter` pod is scheduled on each node.
- Confirm metrics exposure at `<node_IP>:9100/metrics`.
- Configure Prometheus to scrape `node-exporter`.
- Configure a Grafana dashboard for Kubernetes.

## Tools and Technologies

| Tool | Purpose |
|---|---|
| Kubernetes | Container orchestration platform |
| DaemonSet | Runs one pod per node |
| Prometheus | Metrics collection and storage |
| Alertmanager | Alert routing and handling |
| Grafana | Metrics visualization |
| node-exporter | Exposes node-level hardware and OS metrics |
| kubectl | Cluster management CLI |
| containerd / crictl | Container runtime and image management |
| Calico | Cluster networking |

## Architecture

```
                 +-------------------------------------------+
                 |            namespace: monitoring          |
                 |                                           |
  master-1       |   node-exporter (DaemonSet pod, :9100)    |
  worker-1       |   node-exporter (DaemonSet pod, :9100)    |
                 |          |                                |
                 |          | scrape (every 15s)             |
                 |          v                                |
                 |      Prometheus (:9090) --alerts--> Alertmanager (:9093)
                 |          |                                |
                 |          | datasource                     |
                 |          v                                |
                 |       Grafana (:3000)                     |
                 +-------------------------------------------+
```

| Component | Kind | Port | Exposed as |
|---|---|---|---|
| Prometheus | Deployment | 9090 | NodePort 30090 |
| Alertmanager | Deployment | 9093 | NodePort 30093 |
| Grafana | Deployment | 3000 | NodePort 30300 |
| node-exporter | DaemonSet | 9100 | hostNetwork / hostPort 9100 |

## Environment

| Item | Value |
|---|---|
| Kubernetes version | v1.28.15 |
| Nodes | master-1 (control-plane), worker-1 |
| Container runtime | containerd 2.4.2 |
| CNI | Calico |
| OS | Ubuntu 22.04.5 LTS |

## Concept: What is a DaemonSet?

A DaemonSet ensures that one copy of a pod runs on every node (or on a selected subset of nodes). When a new node joins the cluster, the pod is created on it automatically, and when a node is removed, the pod is garbage collected. This makes it the right choice for node-level agents such as log collectors and metrics exporters, because these must run on every machine to observe it.


## Implementation

### Step 1: Create the namespace

```bash
kubectl create namespace monitoring
```

A dedicated namespace isolates all monitoring components from application workloads.

<img width="739" height="60" alt="image" src="https://github.com/user-attachments/assets/b0dff7ae-eba4-4b60-aded-cb9264263525" />


### Step 2: Deploy Prometheus

`prometheus.yaml` contains a ServiceAccount, ClusterRole and ClusterRoleBinding (so Prometheus can discover nodes through the API server), a ConfigMap holding `prometheus.yml`, a Deployment and a NodePort Service.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
  - apiGroups: [""]
    resources: ["nodes", "nodes/metrics", "services", "endpoints", "pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitoring
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    alerting:
      alertmanagers:
        - static_configs:
            - targets: ["alertmanager.monitoring.svc:9093"]
    scrape_configs:
      - job_name: prometheus
        static_configs:
          - targets: ["localhost:9090"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
        - name: prometheus
          image: prom/prometheus:latest
          args:
            - --config.file=/etc/prometheus/prometheus.yml
            - --storage.tsdb.path=/prometheus
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: config
              mountPath: /etc/prometheus
            - name: data
              mountPath: /prometheus
      volumes:
        - name: config
          configMap:
            name: prometheus-config
        - name: data
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app: prometheus
  ports:
    - port: 9090
      targetPort: 9090
      nodePort: 30090
```

```bash
kubectl apply -f prometheus.yaml
```

### Step 3: Deploy Alertmanager

`alertmanager.yaml` contains a ConfigMap, a Deployment and a NodePort Service.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    route:
      receiver: default
    receivers:
      - name: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alertmanager
  template:
    metadata:
      labels:
        app: alertmanager
    spec:
      containers:
        - name: alertmanager
          image: prom/alertmanager:latest
          args:
            - --config.file=/etc/alertmanager/alertmanager.yml
          ports:
            - containerPort: 9093
          volumeMounts:
            - name: config
              mountPath: /etc/alertmanager
      volumes:
        - name: config
          configMap:
            name: alertmanager-config
---
apiVersion: v1
kind: Service
metadata:
  name: alertmanager
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app: alertmanager
  ports:
    - port: 9093
      targetPort: 9093
      nodePort: 30093
```

```bash
kubectl apply -f alertmanager.yaml
```

### Step 4: Deploy Grafana

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:latest
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app: grafana
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 30300
```

```bash
kubectl apply -f grafana.yaml
```

### Step 5: Deploy the node-exporter DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      tolerations:
        - operator: Exists
      containers:
        - name: node-exporter
          image: prom/node-exporter:latest
          args:
            - --path.procfs=/host/proc
            - --path.sysfs=/host/sys
            - --path.rootfs=/host/root
          ports:
            - containerPort: 9100
              hostPort: 9100
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
            - name: sys
              mountPath: /host/sys
              readOnly: true
            - name: root
              mountPath: /host/root
              readOnly: true
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
        - name: root
          hostPath:
            path: /
```

```bash
kubectl apply -f node-exporter.yaml
```

Key fields:

| Field | Purpose |
|---|---|
| `kind: DaemonSet` | Runs one pod per node, no `replicas` field |
| `hostNetwork: true` | The pod uses the node network, so metrics are reachable at `<node_IP>:9100` |
| `hostPID: true` and hostPath volumes | Lets the exporter read real node statistics from `/proc`, `/sys` and `/` |
| `tolerations: operator: Exists` | Tolerates every taint, so the pod runs on the control-plane node as well |

<img width="1351" height="646" alt="image" src="https://github.com/user-attachments/assets/d2579cfa-1a37-45a2-bf9c-78181d3e33a9" />

### Step 6: Validate one pod per node

```bash
kubectl get ds -n monitoring
kubectl get pods -n monitoring -o wide -l app=node-exporter
```

Result:

```
NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
node-exporter   2         2         2       2            2

NAME                  READY   STATUS    NODE
node-exporter-pdkzd   1/1     Running   master-1
node-exporter-pkkrf   1/1     Running   worker-1
```

`DESIRED` equals the number of nodes, and each node hosts exactly one pod.

<img width="1585" height="334" alt="image" src="https://github.com/user-attachments/assets/4df2111c-1e28-4961-9c40-fa94b1e38930" />


### Step 7: Confirm metrics exposure

```bash
kubectl get nodes -o wide
curl -s http://192.168.217.132:9100/metrics | grep -E "^node_(cpu|memory_MemTotal)" | head -5
curl -s http://192.168.217.133:9100/metrics | grep -E "^node_(cpu|memory_MemTotal)" | head -5
```

Both nodes return metrics such as `node_cpu_seconds_total`, confirming that the exporter works on each node.

<img width="1585" height="334" alt="image" src="https://github.com/user-attachments/assets/9e630d23-986f-467d-bb18-ea144a4232cd" />

### Step 8: Configure Prometheus to scrape node-exporter

```bash
kubectl edit configmap prometheus-config -n monitoring
```

The following job was added under `scrape_configs`:

```yaml
      - job_name: node-exporter
        kubernetes_sd_configs:
          - role: node
        relabel_configs:
          - source_labels: [__address__]
            regex: "(.*):10250"
            replacement: "${1}:9100"
            target_label: __address__
```

`role: node` makes Prometheus discover every node automatically, so any node added later is scraped without further changes. The relabel rule replaces the kubelet port `10250` with the exporter port `9100`.

Prometheus was restarted to load the new configuration:

```bash
kubectl rollout restart deployment prometheus -n monitoring
kubectl get pods -n monitoring -l app=prometheus
```

Verification at `http://<node_IP>:30090/targets`: the `node-exporter` job shows both targets as **UP**.

<img width="1834" height="568" alt="image" src="https://github.com/user-attachments/assets/3b2d68d0-9271-4b98-a469-7469395af2c5" />

### Step 9: Configure the Grafana dashboard

1. Open `http://<node_IP>:30300` and log in with the default credentials `admin` / `admin`.
2. Go to **Connections > Data sources > Add data source > Prometheus**.
3. Set the URL to `http://prometheus.monitoring.svc:9090` and click **Save & test**.
4. Go to **Dashboards > New > Import**, enter dashboard ID `1860` (Node Exporter Full), click **Load**, select the Prometheus data source and click **Import**.

The dashboard shows CPU, memory, disk and network metrics, with a selector to switch between nodes.

#### Prometheus data source test
<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/ce161ae3-ba58-4351-92bb-b2a32825d9c0" />

<img width="1280" height="764" alt="Screenshot 2026-11-02 220203" src="https://github.com/user-attachments/assets/69217889-c8a9-468e-8136-b8680862ce81" />

<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/39cb3e51-d05b-4e5e-bee4-50dd40609b26" />

#### Grafana dashboard for master-1
<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/1cd27247-ea2c-43cc-9aed-e0ca650950fd" />

#### Grafana dashboard for worker-1
<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/471f728d-072d-424a-9ad3-eaf844cf8a58" />


---

## Conclusion

This lab demonstrated how a DaemonSet guarantees one pod per node, which is the correct pattern for node-level agents such as `node-exporter`. A complete monitoring pipeline was built in the `monitoring` namespace: `node-exporter` exposes node metrics on port 9100, Prometheus discovers the nodes automatically and scrapes them, Alertmanager is connected for alert handling, and Grafana visualizes the data through the Node Exporter dashboard. The lab also covered practical troubleshooting of taints and tolerations, image pull policies and node DNS issues.
