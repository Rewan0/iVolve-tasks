# Lab 12: Managing Configuration and Sensitive Data with ConfigMaps and Secrets

## Objective

Separate MySQL configuration from the application: non-sensitive values go in a **ConfigMap**, credentials go in a **Secret** (Base64-encoded in a YAML manifest).

| # | Task | Result |
|---|------|--------|
| 1 | Create ConfigMap `ivolve-cm` in `ivolve` | ✅ |
| 2 | Encode credentials with Base64 | ✅ |
| 3 | Create Secret `ivolve-secret` from a YAML manifest | ✅ |
| 4 | Verify both resources | ✅ |

> **Naming note:** resource names must be lowercase, so `ivolveCM` → `ivolve-cm` and `ivolveSecret` → `ivolve-secret`.

---

## Concepts

| Resource | Stores | Example keys |
|----------|--------|--------------|
| **ConfigMap** | Non-sensitive configuration | `DB_HOST`, `DB_USER` |
| **Secret** | Sensitive data | `MYSQL_ROOT_PASSWORD`, `DB_PASSWORD` |

```
            namespace: ivolve
   ┌────────────────┐    ┌──────────────────┐
   │  ivolve-cm     │    │  ivolve-secret   │
   │  DB_HOST       │    │  MYSQL_ROOT_...  │
   │  DB_USER       │    │  DB_PASSWORD     │
   └───────┬────────┘    └────────┬─────────┘
           └──────► MySQL / App Pods ◄──────┘
```

> Base64 is **encoding, not encryption**. Anyone can decode it, so never commit real credentials to a public repository.

---

## Steps

### Step 1: Create the ConfigMap

```bash
kubectl create configmap ivolve-cm -n ivolve \
  --from-literal=DB_HOST=mysql-headless \
  --from-literal=DB_USER=ivolve_user
```

- `DB_HOST`: hostname of the MySQL StatefulSet's headless Service
- `DB_USER`: application user allowed to connect to the `ivolve` database


```bash
kubectl get configmap ivolve-cm -n ivolve -o yaml
```

**ConfigMap creation + YAML output**
<img width="1203" height="514" alt="image" src="https://github.com/user-attachments/assets/7c3150a2-8036-4021-ad5d-e393f2b60890" />


### Step 2: Encode the credentials with Base64

```bash
echo -n 'rootpass123' | base64
echo -n 'dbpass123' | base64
```

Output:

```
cm9vdHBhc3MxMjM=
ZGJwYXNzMTIz
```

`-n` prevents a trailing newline from being encoded into the value.

**Base64 encoding**
<img width="1033" height="123" alt="image" src="https://github.com/user-attachments/assets/dfddc9a4-6020-42d5-9c87-55d8a267d621" />


### Step 3: Create the Secret manifest

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ivolve-secret
  namespace: ivolve
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: cm9vdHBhc3MxMjM=
  DB_PASSWORD: ZGJwYXNzMTIz
```

Saved as `ivolve-secret.yaml`.


### Step 4: Apply the Secret

```bash
kubectl apply -f ivolve-secret.yaml
```

**Secret created**
<img width="1045" height="393" alt="image" src="https://github.com/user-attachments/assets/4e2ca5c5-cf24-4e58-8b9f-4c81107b3b63" />


### Step 5: Verify

```bash
kubectl get configmap,secret -n ivolve
kubectl describe secret ivolve-secret -n ivolve
```

`describe` hides the values and shows only their size in bytes.


Decode a value to confirm it was stored correctly:

```bash
kubectl get secret ivolve-secret -n ivolve -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

Returns `dbpass123`.

**`get configmap,secret` and `describe secret`+ decoded password**
<img width="1699" height="652" alt="image" src="https://github.com/user-attachments/assets/64f2f73d-3326-4463-b490-f3950de15f8d" />


---

## Result Summary

| Resource | Name | Keys |
|----------|------|------|
| ConfigMap | `ivolve-cm` | `DB_HOST`, `DB_USER` |
| Secret (Opaque) | `ivolve-secret` | `MYSQL_ROOT_PASSWORD`, `DB_PASSWORD` |

---

## Conclusion

Configuration and credentials are now stored separately in the `ivolve` namespace: plain settings in a ConfigMap and Base64-encoded passwords in a Secret, ready to be injected into the MySQL StatefulSet.
