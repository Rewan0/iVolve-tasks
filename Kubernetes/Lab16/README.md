# Lab 16: Pod Resource Management with CPU and Memory Requests and Limits

## Overview

This lab updates the existing `ivolveapp` Deployment in the `ivolve` namespace to define CPU and memory **requests** and **limits** for its application container. The applied values are then verified with `kubectl describe pod`, and real-time consumption is monitored with `kubectl top pod` using the Kubernetes Metrics Server.

## Objectives

- Define resource requests and limits on an existing Deployment.
- Verify the applied values on the running Pod.
- Deploy and configure the Metrics Server to monitor real-time resource usage.
- Troubleshoot scheduling and quota issues caused by resource reservations.

## Concepts

| Term | Description | Enforced by |
|------|-------------|-------------|
| Request | Minimum amount of a resource guaranteed to the container. Used for scheduling decisions. | kube-scheduler |
| Limit | Maximum amount of a resource the container may use. | kubelet / container runtime |

Behavior when a limit is exceeded:

| Resource | Behavior |
|----------|----------|
| CPU | The container is throttled. It is not terminated. |
| Memory | The container is terminated with `OOMKilled` and restarted. |

## Requirements

| Resource | Request | Limit |
|----------|---------|-------|
| CPU | `1` (1 vCPU) | `2` (2 vCPUs) |
| Memory | `1Gi` | `2Gi` |

## Tools and Technologies

- Kubernetes (kubeadm cluster: 1 master node, 1 worker node)
- kubectl
- Metrics Server

## Architecture

```
                 +-------------------+
                 |     master-1      |
                 |   (control-plane) |
                 +---------+---------+
                           |
                 +---------v---------+
                 |     worker-1      |
                 |  label: project=ivolve
                 |  taint: workload=ivolveApp:NoSchedule
                 |                   |
                 |  +-------------+  |
                 |  | ivolveapp   |  |   requests: 1 CPU / 1Gi
                 |  | (Deployment)|  |   limits:   2 CPU / 2Gi
                 |  +-------------+  |
                 |  +-------------+  |
                 |  | ivolvedb-0  |  |
                 |  | (StatefulSet)| |
                 |  +-------------+  |
                 +-------------------+
```

## Implementation Steps

### Step 1: Inspect the existing Deployment

```bash
kubectl get deployment -n ivolve
kubectl get deployment ivolveapp -n ivolve -o yaml
```

This confirms the Deployment name, namespace, and container name (`ivolveapp`). At this point the container shows `resources: {}`.

<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/bcf67f0e-e7fd-4847-a378-d8854001ea52" />

### Step 2: Add resource requests and limits

```bash
kubectl edit deployment ivolveapp -n ivolve
```

Under the `ivolveapp` container (not the `init-db` init container), add:

```yaml
        resources:
          requests:
            cpu: "1"
            memory: "1Gi"
          limits:
            cpu: "2"
            memory: "2Gi"
```

Because the Pod template changed, the Deployment triggers a rolling update automatically.

<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/41b01f69-793e-44d3-a452-1a2d783654ea" />

### Step 3: Adjust the rolling update strategy

The namespace has a ResourceQuota of 3 Pods, so the default strategy (which creates a new Pod before deleting an old one) cannot proceed. The strategy is changed to replace Pods one at a time without exceeding the quota:

```bash
kubectl patch deployment ivolveapp -n ivolve -p \
'{"spec":{"strategy":{"rollingUpdate":{"maxSurge":0,"maxUnavailable":1}}}}'
kubectl rollout status deployment ivolveapp -n ivolve
kubectl get pods -n ivolve
```
<img width="1174" height="115" alt="image" src="https://github.com/user-attachments/assets/437c5678-3097-4334-a857-d2bc566ffa43" />

### Step 4: Verify with `kubectl describe pod`

```bash
kubectl get pods -n ivolve
kubectl describe pod <ivolveapp-pod-name> -n ivolve | grep -A6 -E "Limits:|Requests:|QoS"
```

Output:

```
    Limits:
      cpu:     2
      memory:  2Gi
    Requests:
      cpu:     1
      memory:  1Gi
...
QoS Class:                   Burstable
```

The QoS class is `Burstable` because the requests are lower than the limits. If they were equal, it would be `Guaranteed`.

<img width="1725" height="543" alt="image" src="https://github.com/user-attachments/assets/4e3c0da4-7777-4a82-bfcd-e4c2bad14f35" />

### Step 5: Deploy the Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
Two adjustments are required for this cluster:

A toleration for the workload=ivolveApp:NoSchedule taint, since worker-1 is the only schedulable node.
The --kubelet-insecure-tls flag, since kubelet certificates in a kubeadm cluster are not signed by the cluster CA.
```bash
kubectl patch deployment metrics-server -n kube-system --type=json -p='[
  {"op":"add","path":"/spec/template/spec/tolerations","value":[{"key":"workload","operator":"Equal","value":"ivolveApp","effect":"NoSchedule"}]},
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}
]'
kubectl get pods -n kube-system | grep metrics-server
```
The Pod reaches 1/1 Running.

<img width="2118" height="367" alt="image" src="https://github.com/user-attachments/assets/ffc2afdc-711d-4676-ad2d-1c4ee7261487" />

### Step 7: Monitor real-time usage

```bash
kubectl top pod -n ivolve
kubectl top node
```

Output:

```
NAME                         CPU(cores)   MEMORY(bytes)
ivolveapp-56d4b75775-wj6fs   1m           21Mi
ivolvedb-0                   4m           381Mi

NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
master-1   88m          4%     1549Mi          84%
worker-1   32m          1%     1436Mi          78%
```

Actual usage is far below the requests. This is expected: a request is a reservation used for scheduling, not a measure of consumption.

<img width="916" height="231" alt="image" src="https://github.com/user-attachments/assets/7b3f2301-683d-4fa1-802d-8a7e060d29ee" />

## Troubleshooting Summary

| Symptom | Root cause | Resolution |
|---------|------------|------------|
| Rollout stuck at `0 out of 2 new replicas have been updated` | `ResourceQuota` allows only 3 Pods and the rolling update needs a 4th | Set `maxSurge: 0`, `maxUnavailable: 1` |
| Second `ivolveapp` Pod `Pending` with `Insufficient memory` | `worker-1` allocatable memory is about 1.78Gi, and each Pod reserves 1Gi | Increase worker RAM, or scale to 1 replica |
| Metrics Server `Pending` | Untolerated taint `workload=ivolveApp` on `worker-1` | Add a matching toleration |
| `Metrics API not available` | Metrics Server not installed or not ready | Install it and wait for `1/1 Running` |


## Conclusion

The ivolveapp Deployment now defines CPU and memory requests (1 CPU, 1Gi) and limits (2 CPU, 2Gi). The values were verified with kubectl describe pod, and real-time usage was monitored with kubectl top after deploying the Metrics Server.
