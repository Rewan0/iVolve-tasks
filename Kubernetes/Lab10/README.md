# Lab 10: Node Isolation Using Taints and Labels in Kubernetes

## Objective

Isolate a worker node so that only pods with a matching **toleration** can be scheduled on it, and label the node so pods can target it with a `nodeSelector`.

| # | Task | Result |
|---|------|--------|
| 1 | Taint `worker-1` with `workload=ivolveApp:NoSchedule` | ✅ |
| 2 | Label `worker-1` with `project=ivolve` | ✅ |
| 3 | Verify with `kubectl describe node` and `kubectl get nodes --show-labels` | ✅ |
| 4 | Test isolation with and without toleration | ✅ |

---

## Cluster Setup

| Node | Role | Version |
|------|------|---------|
| `master-1` | control-plane | v1.28.15 |
| `worker-1` | worker | v1.28.15 |

```
┌────────────────────┐        ┌──────────────────────────────┐
│     master-1       │        │          worker-1            │
│   (control-plane)  │        │  Taint : workload=ivolveApp  │
│                    │        │          :NoSchedule         │
│                    │        │  Label : project=ivolve      │
└────────────────────┘        └──────────────────────────────┘
```

**cluster nodes (`kubectl get nodes`)**
<img width="1453" height="126" alt="image" src="https://github.com/user-attachments/assets/93da837a-2b2d-4ee6-b470-dd8be996cc30" />


---

## Concepts

| Concept | Purpose |
|---------|---------|
| **Taint** | Repels pods from a node unless they tolerate it |
| **Toleration** | Set on a pod to allow it onto a tainted node |
| **Label** | Tags a node so pods can select it |
| **nodeSelector** | Makes a pod target nodes with a specific label |

> A taint only *repels*. To make a pod actually land on the node, it needs **both** a toleration and a nodeSelector.

---

## Steps

### Step 1: Apply the taint

```bash
kubectl taint nodes worker-1 workload=ivolveApp:NoSchedule
```

Format is `key=value:effect`. With `NoSchedule`, the scheduler will not place new pods on `worker-1` unless they carry a matching toleration. Pods already running are not affected.

**taint command output**
<img width="1329" height="66" alt="image" src="https://github.com/user-attachments/assets/b5f9aa5c-0ea7-44eb-867d-627beb2c1f08" />


### Step 2: Label the node

```bash
kubectl label nodes worker-1 project=ivolve
```

This identifies the node so pods can target it using `nodeSelector`.

**label command output**
<img width="1065" height="72" alt="image" src="https://github.com/user-attachments/assets/64c92d71-141e-4164-8470-5528fd0ba3a3" />


### Step 3: Verify

```bash
kubectl describe node worker-1 | grep -i taints
kubectl get nodes --show-labels
```

<img width="2538" height="238" alt="image" src="https://github.com/user-attachments/assets/84b958f9-00bb-4ad8-b350-feb5a165de0c" />



---

## Testing the Isolation

### Test 1: Pod without toleration 

```bash
kubectl run no-toleration --image=nginx
kubectl get pods -o wide
kubectl describe pod no-toleration | tail -n 5
```

The pod stays `Pending`. The scheduler event confirms why:

```
0/2 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: },
1 node(s) had untolerated taint {workload: ivolveApp}.
```

**pod Pending + FailedScheduling event**
<img width="2544" height="372" alt="image" src="https://github.com/user-attachments/assets/45da764f-ede1-487e-91db-20a28faf5ad7" />


### Test 2: Pod with toleration + nodeSelector 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ivolve-pod
spec:
  nodeSelector:
    project: ivolve
  tolerations:
  - key: "workload"
    operator: "Equal"
    value: "ivolveApp"
    effect: "NoSchedule"
  containers:
  - name: nginx
    image: nginx
```

```bash
kubectl apply -f ivolve-pod.yaml
kubectl get pods -o wide
```

The pod is scheduled on `worker-1` and reaches `Running`.

**`ivolve-pod` Running on worker-1**
<img width="2448" height="897" alt="image" src="https://github.com/user-attachments/assets/38848439-055e-47cb-942f-6141d4fa6150" />


---

## Result Summary

| Pod | Toleration | nodeSelector | Result |
|-----|-----------|--------------|--------|
| `no-toleration` | ❌ | ❌ | `Pending` (rejected by taints) |
| `ivolve-pod` | ✅ | ✅ | `Running` on `worker-1` |

---

## Cleanup

```bash
kubectl delete pod no-toleration ivolve-pod

# Optional: remove taint and label
kubectl taint nodes worker-1 workload=ivolveApp:NoSchedule-
kubectl label nodes worker-1 project-
```

---

## Conclusion

Combining a **taint** (to repel) with a **label + nodeSelector** (to attract) gives full node isolation: only pods that tolerate the taint and select the label run on `worker-1`.
