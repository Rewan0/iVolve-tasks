# Lab 11: Namespace Management and Resource Quota Enforcement

![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.28.15-326CE5?logo=kubernetes&logoColor=white)
![Status](https://img.shields.io/badge/Lab-Completed-success)
![Track](https://img.shields.io/badge/Track-iVolve%20Cloud%20DevOps-orange)

## Objective

Isolate application resources in a dedicated namespace and enforce a limit of **3 pods** using a ResourceQuota.

| # | Task | Result |
|---|------|--------|
| 1 | Create namespace `ivolve` | ✅ |
| 2 | Create ResourceQuota `ivolve-quota` (pods = 3) | ✅ |
| 3 | Verify the quota is applied | ✅ |
| 4 | Confirm the 4th pod is rejected | ✅ |

> **Naming note:** Kubernetes resource names must be lowercase (RFC 1123), so `ivolveQuota` is created as `ivolve-quota`.

---

## Concepts

| Concept | Purpose |
|---------|---------|
| **Namespace** | Logical isolation of resources inside a cluster |
| **ResourceQuota** | Caps the total resources (pods, CPU, memory, ...) a namespace can use |

```
┌──────────────── Cluster ────────────────┐
│  ┌──────── namespace: ivolve ────────┐  │
│  │  ResourceQuota: ivolve-quota      │  │
│  │  hard: pods = 3                   │  │
│  │  [pod1] [pod2] [pod3]  ❌ pod4    │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

---

## Steps

### Step 1: Create the namespace

```bash
kubectl create namespace ivolve
kubectl get ns
```

Creates an isolated space for the application resources.

**namespace created and listed**
<img width="1174" height="370" alt="image" src="https://github.com/user-attachments/assets/4b86e36a-948e-46e6-adb8-22b30b47bb33" />


### Step 2: Create the ResourceQuota

```bash
kubectl create quota ivolve-quota --hard=pods=3 -n ivolve
```

`--hard=pods=3` sets the maximum number of pods in the `ivolve` namespace to 3.

Equivalent YAML:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ivolve-quota
  namespace: ivolve
spec:
  hard:
    pods: "3"
```


### Step 3: Verify the quota

```bash
kubectl get resourcequota -n ivolve
kubectl describe resourcequota ivolve-quota -n ivolve
```

Expected: `pods  0  3` (0 used out of 3).

**`get` and `describe` resourcequota**
<img width="1432" height="337" alt="image" src="https://github.com/user-attachments/assets/11355025-bb6c-4212-88f6-d9aee33832f6" />


---

## Testing the Quota

### Test 1: Create 3 pods 

```bash
kubectl run pod1 --image=nginx -n ivolve
kubectl run pod2 --image=nginx -n ivolve
kubectl run pod3 --image=nginx -n ivolve
kubectl get resourcequota -n ivolve
```

The quota now shows `pods: 3/3`.

**3 pods created + quota at 3/3**
<img width="1089" height="274" alt="image" src="https://github.com/user-attachments/assets/a816456d-50c1-4cf5-a535-3ea4df5327d7" />


### Test 2: Create a 4th pod 

```bash
kubectl run pod4 --image=nginx -n ivolve
```

Kubernetes rejects it:

```
Error from server (Forbidden): pods "pod4" is forbidden: exceeded quota: ivolve-quota, requested: pods=1, used: pods=3, limited: pods=3
```

**Forbidden error for pod4**
<img width="2139" height="57" alt="image" src="https://github.com/user-attachments/assets/9cb664ca-06ee-4054-bcfc-c5c07b2d6484" />


---

## Result Summary

| Pod | Result |
|-----|--------|
| `pod1`, `pod2`, `pod3` | ✅ Created |
| `pod4` | ❌ Forbidden: quota exceeded |

---

## Conclusion

The ResourceQuota `ivolve-quota` successfully limits the `ivolve` namespace to 3 pods, and Kubernetes blocks any additional pod once the limit is reached.
