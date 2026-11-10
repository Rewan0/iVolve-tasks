# Lab 24: Initial Ansible Configuration and Ad-Hoc Execution

## Objective

Install and configure Ansible on a control node, establish key-based SSH access to a managed node, define an inventory, and use an ad-hoc command to install Docker on the managed node.

## Environment

| Role | Hostname | IP Address | Operating System | Notes |
|------|----------|------------|------------------|-------|
| Control node | master-1 | 192.168.217.132 | Ubuntu 22.04 LTS | Ansible 2.10.8 |
| Managed node | worker-1 | 192.168.217.133 | Ubuntu 22.04 LTS | Receives Docker |

```
+-----------------------+      SSH (key-based)      +-----------------------+
|     Control node      | ------------------------> |     Managed node      |
|       master-1        |                           |       worker-1        |
|    192.168.217.132    |  ansible ad-hoc commands  |    192.168.217.133    |
|    Ansible 2.10.8     |                           |       Docker CE       |
+-----------------------+                           +-----------------------+
```

## Tools and Technologies

| Tool | Purpose |
|------|---------|
| Ansible | Agentless configuration management over SSH |
| OpenSSH (ssh-keygen, ssh-copy-id) | Key generation and public key distribution |
| INI inventory | Definition of managed hosts and connection variables |
| apt module | Package installation on Debian-based systems |
| Docker CE | Container runtime installed on the managed node |

## Implementation

All commands in this lab are executed on the control node (master-1).

### Step 1: Install Ansible on the control node

```bash
sudo apt update
sudo apt install -y ansible
ansible --version
```

Ansible is agentless, so it is installed only on the control node. The version output confirms the installation and shows the Python interpreter and module paths in use.

<img width="1644" height="198" alt="image" src="https://github.com/user-attachments/assets/4253aaa6-2f21-4a64-9a5c-6a67d6cf9fe2" />

### Step 2: Generate a new SSH key pair

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ansible_key -N ""
```

An Ed25519 key pair is created at `~/.ssh/ansible_key` with an empty passphrase so Ansible can authenticate non-interactively. A dedicated key is used so it can be managed independently from the user's default key.

<img width="1027" height="505" alt="image" src="https://github.com/user-attachments/assets/3b08e4a9-0d26-49be-b3c8-aa9b0064638f" />

### Step 3: Transfer the public key to the managed node

```bash
ssh-copy-id -i ~/.ssh/ansible_key.pub rewan@192.168.217.133
```

`ssh-copy-id` appends the public key to `~/.ssh/authorized_keys` on the managed node. The account password is requested once for this step; afterwards the control node authenticates with the key only.

<img width="1674" height="408" alt="image" src="https://github.com/user-attachments/assets/45b605ff-6135-46ae-8f37-a884936c8730" />

### Step 4: Create the project directory, inventory, and configuration

```bash
mkdir -p ~/ansible-lab && cd ~/ansible-lab

cat > inventory.ini <<'EOF'
[managed]
node1 ansible_host=192.168.217.133 ansible_user=rewan ansible_ssh_private_key_file=~/.ssh/ansible_key
EOF

cat > ansible.cfg <<'EOF'
[defaults]
inventory = ./inventory.ini
host_key_checking = False
EOF
```

The inventory defines a group named `managed` containing one host, with the address, remote user, and private key used for the connection. `ansible.cfg` points Ansible to this inventory automatically, so it does not need to be passed with `-i`. Ansible commands must be run from this directory for the configuration to be picked up.

<img width="1501" height="292" alt="image" src="https://github.com/user-attachments/assets/5abd5343-7579-4973-88a1-c1ea19f4de9f" />

### Step 5: Test connectivity

```bash
ansible managed -m ping
```

The `ping` module verifies SSH connectivity and that Python is available on the managed node. A `pong` response confirms the control node can manage the host.

<img width="942" height="229" alt="image" src="https://github.com/user-attachments/assets/edd02f84-5046-4ffd-8c81-3eb540b9049f" />

### Step 6: Enable passwordless privilege escalation on the managed node

```bash
ssh -t -i ~/.ssh/ansible_key rewan@192.168.217.133 \
  'echo "rewan ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/rewan && sudo chmod 440 /etc/sudoers.d/rewan'
```

Installing packages requires root privileges (`-b`, become). Since Ansible cannot answer a sudo password prompt, a sudoers drop-in file grants the `rewan` user passwordless sudo on the managed node.

<img width="1677" height="139" alt="image" src="https://github.com/user-attachments/assets/c0661c47-27b2-41f8-9322-050c89200864" />

### Step 7: Inspect the container runtime and prepare apt

```bash
ansible managed -m shell -a "dpkg -l | grep -E 'containerd|docker'"
ansible managed -b -m command -a "apt-get -s --fix-broken install"
ansible managed -b -m command -a "apt-get -y --fix-broken install"
```

The managed node is a Kubernetes worker and already runs `containerd.io` from the Docker repository. For that reason the `docker-ce` package is used rather than the Ubuntu `docker.io` package, which depends on a different containerd package and would conflict.

The node also had an inconsistent dependency state in the kernel meta-packages, which blocks any new installation. A dry run (`-s`) is used first to confirm that only kernel header packages are affected, then the fix is applied.

<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/9eab0647-a08b-4029-9529-57451137696d" />

### Step 8: Install Docker with an ad-hoc command

```bash
ansible managed -b -m apt -a "name=docker-ce state=present update_cache=yes"
```

An ad-hoc command runs a single module without writing a playbook. Here the `apt` module refreshes the package index and ensures `docker-ce` is present. `-b` enables privilege escalation.

<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/9ad6b1b3-cb84-40fc-90cc-1436a9579925" />

## Verification

```bash
ansible managed -m command -a "docker --version"
ansible managed -b -m command -a "systemctl is-active docker"
```

Expected output:

```
node1 | CHANGED | rc=0 >>
Docker version 29.9.3, build 5457199

node1 | CHANGED | rc=0 >>
active
```

<img width="1336" height="169" alt="image" src="https://github.com/user-attachments/assets/92c1b51f-64dc-42ea-9a5a-36100702c6da" />

## Conclusion

Ansible was installed on the control node, key-based SSH access to the managed node was established, an inventory was defined, and connectivity was confirmed with the `ping` module. Docker was then installed on the managed node with a single ad-hoc command, demonstrating how Ansible performs one-off administrative tasks across hosts without a playbook.
