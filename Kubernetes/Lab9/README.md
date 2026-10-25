# Lab 9: Deploy a Kubernetes Cluster using Kubeadm

> A 2-node Kubernetes cluster (1 control plane + 1 worker) built from scratch with **kubeadm**, **containerd**, and **Calico** on VMware Workstation.

![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.28.15-326CE5?logo=kubernetes&logoColor=white)
![containerd](https://img.shields.io/badge/containerd-2.4.2-575757)
![Calico](https://img.shields.io/badge/CNI-Calico_v3.27-FB8C00)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-E95420?logo=ubuntu&logoColor=white)

---

## Table of Contents

1. [Objectives](#objectives)
2. [Architecture](#architecture)
3. [Environment](#environment)
4. [Step 1: Create the VMs](#step-1-create-the-vms)
5. [Step 2: Node Preparation (both nodes)](#step-2-node-preparation-both-nodes)
6. [Step 3: Install containerd (both nodes)](#step-3-install-containerd-both-nodes)
7. [Step 4: Install kubeadm, kubelet, kubectl (both nodes)](#step-4-install-kubeadm-kubelet-kubectl-both-nodes)
8. [Step 5: Initialize the Control Plane (master-1)](#step-5-initialize-the-control-plane-master-1)
9. [Step 6: Configure kubectl (master-1)](#step-6-configure-kubectl-master-1)
10. [Step 7: Join the Worker Node (worker-1)](#step-7-join-the-worker-node-worker-1)
11. [Step 8: Install Calico CNI (master-1)](#step-8-install-calico-cni-master-1)
12. [Step 9: Verification](#step-9-verification)
13. [Troubleshooting Notes](#troubleshooting-notes)
14. [Key Concepts](#key-concepts)
15. [Conclusion](#conclusion)

---

## Objectives

| # | Task | Status |
|---|------|:------:|
| 1 | Install and configure **containerd** on all nodes | ✅ |
| 2 | Install **kubeadm, kubelet, kubectl** on control plane and worker | ✅ |
| 3 | Configure **kubectl** to communicate with the cluster | ✅ |
| 4 | Install the **Calico CNI** plugin for pod networking | ✅ |
| 5 | Initialize the control plane with **`kubeadm init`** | ✅ |
| 6 | Join the worker node with **`kubeadm join`** | ✅ |
| 7 | Verify all nodes are `Ready` and components are healthy | ✅ |

---

## Architecture

```
            ┌─────────────────────────────────────┐
            │    Windows Host (VMware NAT)        │
            │    Network: 192.168.217.0/24        │
            └──────────────────┬──────────────────┘
                               │
          ┌────────────────────┴────────────────────┐
          │                                         │
┌─────────▼──────────┐                   ┌──────────▼─────────┐
│     master-1       │                   │      worker-1      │
│  192.168.217.132   │◄─── kubeadm ────► │  192.168.217.133   │
│   Control Plane    │       join        │    Worker Node     │
│                    │                   │                    │
│ • kube-apiserver   │                   │ • kubelet          │
│ • etcd             │                   │ • kube-proxy       │
│ • kube-scheduler   │                   │ • containerd       │
│ • controller-mgr   │                   │ • calico-node      │
│ • kubelet          │                   │                    │
│ • containerd       │                   │                    │
└────────────────────┘                   └────────────────────┘

 Pod network (Calico): 10.244.0.0/16     Service network: 10.96.0.0/12
```

---

## Environment

| Node | Role | IP Address | CPU | RAM | Disk | OS |
|------|------|-----------|:---:|:---:|:----:|----|
| `master-1` | Control Plane | `192.168.217.132` | 2 | 4 GB | 20 GB | Ubuntu 22.04.5 LTS |
| `worker-1` | Worker | `192.168.217.133` | 2 | 2 GB | 20 GB | Ubuntu 22.04.5 LTS |

| Component | Version |
|-----------|---------|
| Kubernetes | v1.28.15 |
| Container runtime | containerd 2.4.2 |
| CNI | Calico v3.27.0 |
| Hypervisor | VMware Workstation (NAT network) |

---

## Step 1: Create the VMs

Two Ubuntu Server 22.04 VMs were created in VMware Workstation (NAT network, 20 GB disk each, OpenSSH server enabled). Work was done over SSH from Windows PowerShell.

**IP address of each node (`ip a`)**
<img width="2560" height="1539" alt="image" src="https://github.com/user-attachments/assets/fec3a5ae-eea9-4419-b4e7-23025e4337e7" />
<img width="2560" height="1539" alt="image" src="https://github.com/user-attachments/assets/d8e5f6d3-0623-4fe3-8954-71e474cfdfb4" />

---

## Step 2: Node Preparation (both nodes)

Set the hostname (each node with its own name):

```bash
sudo hostnamectl set-hostname master-1   # on master-1
sudo hostnamectl set-hostname worker-1   # on worker-1
```

Add host entries:

```bash
sudo tee -a /etc/hosts <<EOF
192.168.217.132 master-1
192.168.217.133 worker-1
EOF
```

Disable swap (kubelet refuses to run with swap on):

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
free -h | grep Swap
```

Load kernel modules (`overlay` for container layers, `br_netfilter` for bridge traffic through iptables):

```bash
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
lsmod | grep -E '^overlay|^br_netfilter'
```

Set sysctl parameters:

```bash
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

**swap disabled, modules loaded, sysctl values = 1**
<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/0489cafd-8862-4f69-bb9e-a75a1932e6ae" />
<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/fdefbfe1-a976-446d-a64e-377e186c1f4a" />


---

## Step 3: Install containerd (both nodes)

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y containerd.io
```

Configure the **systemd cgroup driver** (must match kubelet):

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
grep SystemdCgroup /etc/containerd/config.toml
sudo systemctl enable containerd
sudo systemctl restart containerd
systemctl is-active containerd
```

**`SystemdCgroup = true` and containerd `active` (both nodes)**
<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/a69d344d-3f69-4bc3-834f-da89f9e1323e" />
<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/6de4be14-f736-4acc-b07a-1815e63e698f" />


---

## Step 4: Install kubeadm, kubelet, kubectl (both nodes)

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

Verify:

```bash
kubeadm version -o short
kubelet --version
kubectl version --client
```

**versions on both nodes**
<img width="1684" height="364" alt="image" src="https://github.com/user-attachments/assets/b83d650a-8585-4b99-b666-c353c0169f6a" />
<img width="1441" height="363" alt="image" src="https://github.com/user-attachments/assets/b3ad5df3-04fc-48a0-a220-94ea76c1a9c1" />


---

## Step 5: Initialize the Control Plane (master-1)

The pod CIDR is `10.244.0.0/16` to avoid overlapping with the VM network (`192.168.217.0/24`).

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.217.132 \
  --pod-network-cidr=10.244.0.0/16 \
  --kubernetes-version=v1.28.15 | tee ~/kubeadm-init.log
```

**`kubeadm init`**
<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/dd63c720-c49e-4a3a-a238-ccb5041187e0" />

---

## Step 6: Configure kubectl (master-1)

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
```

The node shows `NotReady` at this point because no CNI is installed yet.

**`kubectl get nodes` (NotReady)**
<img width="1227" height="111" alt="image" src="https://github.com/user-attachments/assets/75df75bf-852a-4188-bb12-074eaa14638b" />

---

## Step 7: Join the Worker Node (worker-1)

On master-1, generate the join command:

```bash
kubeadm token create --print-join-command
```

On **worker-1**, run the printed command with `sudo`:

```bash
sudo kubeadm join 192.168.217.132:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

**"This node has joined the cluster" on worker-1**
<img width="2559" height="498" alt="image" src="https://github.com/user-attachments/assets/af63d549-aeca-411e-adf6-9b3a0d93106e" />

**`kubectl get nodes` showing both nodes**
<img width="2559" height="162" alt="image" src="https://github.com/user-attachments/assets/643ce222-c819-470a-9207-fd4f794ed9d6" />


---

## Step 8: Install Calico CNI (master-1)

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml

# Match the pod CIDR used in kubeadm init
sed -i 's#192.168.0.0/16#10.244.0.0/16#' custom-resources.yaml
grep cidr custom-resources.yaml

kubectl create -f custom-resources.yaml
watch kubectl get pods -n calico-system
```

**Calico pods `Running`**
<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/eb1352ee-643a-4a81-94c1-aaa2a3063123" />

---

## Step 9: Verification

### 9.1 Nodes are Ready

```bash
kubectl get nodes -o wide
```
<img width="1179" height="124" alt="image" src="https://github.com/user-attachments/assets/4dec6909-49a6-4b0f-aaed-5bd669ef602d" />


### 9.2 All system pods are Running

```bash
kubectl get pods -A -o wide
```
<img width="2559" height="570" alt="image" src="https://github.com/user-attachments/assets/92b6a76b-b025-45c8-b795-8c8efc862f69" />


### 9.3 Control plane components

```bash
kubectl get pods -n kube-system
```
<img width="1306" height="306" alt="image" src="https://github.com/user-attachments/assets/3ae62c01-ac3b-4bf8-be6f-a5eb8d3718f2" />

### 9.4 API server health

```bash
kubectl get --raw='/readyz?verbose' | tail -n 5
```
<img width="1158" height="181" alt="image" src="https://github.com/user-attachments/assets/2c683fd8-993d-48a2-a85a-77abe203acf6" />

### 9.5 Workload test (nginx)

```bash
kubectl create deployment nginx-test --image=nginx --replicas=2
kubectl wait --for=condition=Ready pod -l app=nginx-test --timeout=180s
kubectl get pods -o wide
```
<img width="2106" height="271" alt="image" src="https://github.com/user-attachments/assets/560583de-bed1-4434-945c-091e6efcb47f" />

### 9.6 DNS test

```bash
kubectl run dns-test --image=busybox:1.28 --restart=Never -- nslookup kubernetes.default
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/dns-test --timeout=120s
kubectl logs dns-test
```

Expected: `kubernetes.default` resolves to `10.96.0.1`.
<img width="2070" height="310" alt="image" src="https://github.com/user-attachments/assets/f69c316f-1981-45fa-a8e7-467942f030d3" />

### 9.7 Cleanup

```bash
kubectl delete deployment nginx-test
kubectl delete pod dns-test
```

---

## Troubleshooting Notes

| Problem | Cause | Fix |
|---------|-------|-----|
| VMware: *"Could not create anonymous paging file"* | Not enough memory for a 4 GB VM | Lowered VM RAM / increased Windows paging file |
| SSH dropped during `kubeadm init` | Host under memory pressure | Reconnected, checked `kubeadm-init.log` and `admin.conf`; use `tmux` for long commands |
| Commands ran on Windows instead of the VM | SSH disconnected, prompt became `PS C:\...` | Check the prompt (`rewan@master-1`) before running commands |
| containerd 2.x instead of 1.7 | Docker repo ships the latest version | Default config still had `SystemdCgroup = false`, so the same `sed` worked |
| Calico default CIDR overlaps VM network | `192.168.0.0/16` contains `192.168.217.x` | Used `10.244.0.0/16` in both `kubeadm init` and `custom-resources.yaml` |

---

## Key Concepts

| Concept | Why it matters |
|---------|----------------|
| **Swap off** | Scheduler assumes real RAM; kubelet refuses to start with swap enabled |
| **overlay / br_netfilter** | Container image layers and iptables visibility on bridged traffic |
| **ip_forward** | Lets nodes route pod traffic between each other |
| **SystemdCgroup = true** | containerd and kubelet must use the same cgroup driver |
| **CNI (Calico)** | Gives pods IPs and connects them across nodes; nodes stay `NotReady` without it |
| **Pod CIDR vs node network** | Must not overlap, otherwise routing breaks |

---

---

## Conclusion

In this lab, a working Kubernetes cluster was built from scratch using **kubeadm**: one control plane node (`master-1`) and one worker node (`worker-1`), both running **containerd** as the container runtime and **Calico** for pod networking.

**What was achieved**

| Item | Result |
|------|--------|
| containerd installed and configured (systemd cgroup driver) | Done on both nodes |
| kubeadm, kubelet, kubectl v1.28.15 installed | Done on both nodes |
| Control plane initialized with `kubeadm init` | Done on `master-1` |
| Worker joined with `kubeadm join` | `worker-1` registered |
| Calico CNI installed (pod CIDR `10.244.0.0/16`) | Both nodes `Ready` |
| Cluster verification | All system pods `Running`, API `readyz` passed |
| Workload and DNS tests | nginx pods scheduled on the worker, `kubernetes.default` resolved to `10.96.0.1` |

**What this lab demonstrated**

- A node stays `NotReady` until a CNI plugin is installed.
- Swap, kernel modules, and sysctl settings are prerequisites, not optional tweaks.
- containerd and kubelet must use the same cgroup driver (`systemd`).
- The pod CIDR must not overlap with the node network, which is why `10.244.0.0/16` was used instead of Calico's default.
- Control plane nodes are tainted by default, so regular workloads run on the worker.

The cluster is ready for deploying applications, and it can be extended with more worker nodes using the same `kubeadm join` process.
