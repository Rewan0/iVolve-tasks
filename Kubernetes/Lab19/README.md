# Lab 19: Securing Kubernetes with RBAC and Service Accounts

## Objective
Restrict a CI identity (Jenkins) so it can only **create and update Deployments in the `ivolve` namespace**, using a ServiceAccount, a Role, and a RoleBinding.

## Requirements
1. Create the `jenkinsSA` ServiceAccount in the `ivolve` namespace.
2. Create a token for the ServiceAccount.
3. Define a Role named `ivolveDeployRole` that grants create/update permissions on Deployments in the `ivolve` namespace.
4. Create a RoleBinding named `ivolveDeployRoleBinding` that binds `ivolveDeployRole` to the ServiceAccount.
5. Validate that the role can only create/update Deployments in the `ivolve` namespace.

> **Note:** ServiceAccount names must follow DNS-1123 (lowercase only), so `jenkinsSA` is created as `jenkinssa`. Role and RoleBinding names accept uppercase, so they keep the names from the lab.

## Architecture
```
 ServiceAccount            RoleBinding                   Role
 jenkinssa      <------>   ivolveDeployRoleBinding --->  ivolveDeployRole
 (ivolve ns)                                             deployments: create, update
                                                         (ivolve ns only)
```

## Tools & Technologies
| Tool | Purpose |
|---|---|
| Kubernetes (kubeadm, 1 master + 1 worker) | Container orchestration |
| RBAC | Authorization |
| ServiceAccount + Token | Identity for Jenkins |
| kubectl | Cluster administration and validation |

---

## Implementation

### Step 1: Create the ServiceAccount and its token

The ServiceAccount is the identity Jenkins uses to authenticate to the API server. Since Kubernetes 1.24, tokens are no longer auto-generated as Secrets, so a short-lived JWT is requested explicitly (valid for 1 hour by default). The token is stored in a variable instead of being printed.

```bash
kubectl create serviceaccount jenkinssa -n ivolve
kubectl get sa -n ivolve
TOKEN=$(kubectl create token jenkinssa -n ivolve)
echo "Token length: ${#TOKEN}"
```

<img width="2536" height="372" alt="image" src="https://github.com/user-attachments/assets/777a709b-7845-44bd-a125-87cdd9adc295" />

---

### Step 2: Create the Role and the RoleBinding

The Role defines **what** is allowed: `create` and `update` on `deployments` in the `apps` API group. It is namespaced, so the permissions never apply outside `ivolve`.

The RoleBinding defines **who** gets those permissions: it binds the Role to the `jenkinssa` ServiceAccount (`--serviceaccount` uses the `namespace:name` format).

```bash
kubectl create role ivolveDeployRole \
  --verb=create,update \
  --resource=deployments.apps \
  -n ivolve

kubectl create rolebinding ivolveDeployRoleBinding \
  --role=ivolveDeployRole \
  --serviceaccount=ivolve:jenkinssa \
  -n ivolve

kubectl describe role ivolveDeployRole -n ivolve
kubectl describe rolebinding ivolveDeployRoleBinding -n ivolve
```

<img width="1357" height="597" alt="image" src="https://github.com/user-attachments/assets/6c55d139-d3fb-46dc-8f0a-b3d3ae56dd9f" />

---

### Step 3: Validate the permissions with impersonation

`kubectl auth can-i --as` checks what the ServiceAccount is allowed to do without using its token.

```bash
SA=system:serviceaccount:ivolve:jenkinssa

kubectl auth can-i create deployments -n ivolve --as=$SA       # yes
kubectl auth can-i update deployments -n ivolve --as=$SA       # yes
kubectl auth can-i delete deployments -n ivolve --as=$SA       # no
kubectl auth can-i get deployments -n ivolve --as=$SA          # no
kubectl auth can-i create pods -n ivolve --as=$SA              # no
kubectl auth can-i create deployments -n default --as=$SA      # no
kubectl auth can-i create deployments -n kube-system --as=$SA  # no
```

<img width="1305" height="424" alt="image" src="https://github.com/user-attachments/assets/1f226e6c-2d77-48cf-942d-d3dcb219d207" />

---

### Step 4: Validate with a live test using the token

The kubeconfig on the master contains an admin client certificate, which takes precedence over `--token`. To make sure every request is authenticated as `jenkinssa`, the test uses an empty kubeconfig and passes only the token.

```bash
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
TOKEN=$(kubectl create token jenkinssa -n ivolve)
K="kubectl --kubeconfig=/dev/null --server=$APISERVER --insecure-skip-tls-verify=true --token=$TOKEN"

$K create deployment test-nginx --image=nginx -n ivolve    # allowed
$K create deployment test-nginx --image=nginx -n default   # Forbidden
$K get deployments -n ivolve                               # Forbidden
$K get pods -n ivolve                                      # Forbidden
$K delete deployment test-nginx -n ivolve                  # Forbidden
```

<img width="2559" height="451" alt="image" src="https://github.com/user-attachments/assets/9b361b75-5169-43e0-978e-2cfb7033c0bd" />

| Command | Result |
|---|---|
| Create Deployment in `ivolve` | Created |
| Create Deployment in `default` | Forbidden |
| List Deployments in `ivolve` | Forbidden (`get/list` not granted) |
| List Pods in `ivolve` | Forbidden (other resources not granted) |
| Delete Deployment in `ivolve` | Forbidden (`delete` not granted) |

---

### Step 5: Cleanup

The test Deployment is removed using the admin kubeconfig.

```bash
kubectl delete deployment test-nginx -n ivolve
kubectl get deployments -A | grep test-nginx
```

<img width="973" height="120" alt="image" src="https://github.com/user-attachments/assets/182c6ecc-0554-4c3b-a3bb-ac327d710b43" />

---

## Results
| Action | Namespace | Result |
|---|---|---|
| create Deployment | ivolve | Allowed |
| update Deployment | ivolve | Allowed |
| delete Deployment | ivolve | Forbidden |
| get / list Deployments, Pods | ivolve | Forbidden |
| create Deployment | default, kube-system | Forbidden |

## Key Takeaways
- A Role is namespaced, so its permissions never leak into other namespaces.
- RBAC is deny-by-default: only the listed verbs and resources are allowed.
- `kubectl apply` on an existing Deployment uses `patch`, so add that verb if Jenkins needs it.
- Tokens are time-limited, which reduces the impact of a leak.
- Using `--token` with a kubeconfig that already has a client certificate authenticates as the certificate user, so RBAC tests must use an empty kubeconfig or `--as` impersonation.

## Conclusion
The `jenkinssa` ServiceAccount can create and update Deployments in the `ivolve` namespace and nothing else, which follows the principle of least privilege.
