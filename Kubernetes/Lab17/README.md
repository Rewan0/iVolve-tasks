# Lab 17: Control Pod-to-Pod Traffic via Network Policy

## Objective

Restrict network access to the MySQL pod using a Kubernetes NetworkPolicy, so that only the application pods can connect to it, and only on the MySQL port (3306).

## Tools and Technologies

- Kubernetes (NetworkPolicy, StatefulSet, Deployment, ResourceQuota)
- Calico CNI
- kubectl
- BusyBox (netcat for connectivity testing)
  
## Requirements

| Item | Value |
|---|---|
| Policy name | ivolveNP (created as `ivolvenp`, since Kubernetes resource names must be lowercase) |
| Target pod | MySQL pod (`app=ivolvedb`) |
| Policy type | Ingress only |
| Allowed source | Application pods (`app=ivolveapp`) |
| Allowed port | 3306/TCP |

## Environment

- Namespace: `ivolve`
- MySQL pod: `ivolvedb-0` (StatefulSet), label `app=ivolvedb`
- Application pod: `ivolveapp` (Deployment), label `app=ivolveapp`
- CNI plugin: Calico (enforces NetworkPolicy)

## Implementation

### Step 1: Identify the pod labels

NetworkPolicy selects pods by labels, so the actual labels must be known first.

```bash
kubectl get pods -n ivolve --show-labels
```

### Step 2: Verify the CNI supports NetworkPolicy

A policy is only enforced if the CNI plugin supports it. Calico was found running in the cluster.

```bash
kubectl get pods -A | grep -Ei "flannel|calico|cilium|weave"
```

### Step 3: Create the NetworkPolicy manifest

```bash
vim ivolveNP.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ivolvenp
  namespace: ivolve
spec:
  podSelector:
    matchLabels:
      app: ivolvedb
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: ivolveapp
      ports:
        - protocol: TCP
          port: 3306
```

Explanation:

- `podSelector`: the pods protected by the policy (MySQL).
- `policyTypes: Ingress`: only incoming traffic is controlled; egress is unaffected.
- `ingress.from.podSelector`: only pods labeled `app=ivolveapp` are allowed.
- `ports`: restricts access to TCP port 3306.

<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/bfa0b311-4814-45b6-ac96-597eddee7785" />

### Step 4: Apply the policy

```bash
kubectl apply -f ivolveNP.yaml
```

### Step 5: Verify the policy

```bash
kubectl get networkpolicy -n ivolve
kubectl describe networkpolicy ivolvenp -n ivolve
```

The output confirms: `PodSelector: app=ivolvedb`, ingress from `app=ivolveapp`, port `3306/TCP`, and policy type `Ingress`.

<img width="1135" height="399" alt="image" src="https://github.com/user-attachments/assets/94a802df-be78-4704-90ab-7e25f97e69c6" />

### Step 6: Test the policy

Get the MySQL pod IP:

```bash
kubectl get pod ivolvedb-0 -n ivolve -o wide
```

**Test A: from the application pod (expected: allowed)**

```bash
kubectl exec -n ivolve ivolveapp-56d4b75775-wj6fs -c ivolveapp -- nc -zv -w 3 10.244.226.95 3306
```

Result: `10.244.226.95 (10.244.226.95:3306) open`

<img width="1695" height="57" alt="image" src="https://github.com/user-attachments/assets/c2e9d3ee-fde9-4d2f-a233-3a9e9acc728b" />

**Test B: from a pod without the application label (expected: blocked)**

A temporary busybox pod was created with a toleration for the worker node taint (`workload=ivolveApp:NoSchedule`):

```bash
kubectl apply -n ivolve -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  tolerations:
    - operator: Exists
  containers:
    - name: busybox
      image: busybox:1.36
      command: ["sleep", "3600"]
EOF

kubectl exec -n ivolve test-pod -- nc -zv -w 3 10.244.226.95 3306
```

Result: `Connection timed out`

<img width="1240" height="97" alt="image" src="https://github.com/user-attachments/assets/262dca1e-2275-47ce-a5ba-2d0c6403f62d" />

**Test C: same pod after adding the application label (expected: allowed)**

```bash
kubectl label pod test-pod app=ivolveapp -n ivolve
kubectl exec -n ivolve test-pod -- nc -zv -w 3 10.244.226.95 3306
```

Result: `10.244.226.95 (10.244.226.95:3306) open`

This confirms the policy filters traffic based on pod labels.

<img width="1312" height="118" alt="image" src="https://github.com/user-attachments/assets/4bcefca7-f434-4d75-99a9-dceb8ea7f617" />

## Results

| Source | Label | Result |
|---|---|---|
| ivolveapp pod | app=ivolveapp | Allowed (open) |
| test-pod | none | Blocked (timed out) |
| test-pod | app=ivolveapp | Allowed (open) |


## Conclusion

The NetworkPolicy `ivolvenp` successfully isolates the MySQL pod. Only pods labeled `app=ivolveapp` can reach it, and only on TCP port 3306; all other ingress traffic is dropped. This applies the principle of least privilege at the network level and reduces the attack surface of the database tier.
