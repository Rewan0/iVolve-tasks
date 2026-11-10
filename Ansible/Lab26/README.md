# Lab 26: Structured Configuration Management with Ansible Roles

## Objective

Refactor the configuration from Lab 25 into separate, reusable Ansible roles for Docker, kubectl, and Jenkins, and write a playbook that applies them to the managed node.

## Environment

| Role | Hostname | IP Address | Operating System | Notes |
|------|----------|------------|------------------|-------|
| Control node | master-1 | 192.168.217.132 | Ubuntu 22.04 LTS | Ansible 2.10.8 |
| Managed node | worker-1 | 192.168.217.133 | Ubuntu 22.04 LTS | Configured by the roles |

The inventory, `ansible.cfg`, and SSH key from Lab 24 are reused.

## Project Structure

```
ansible-lab/
+-- ansible.cfg
+-- inventory.ini
+-- roles-playbook.yml
+-- roles/
    +-- docker/
    |   +-- tasks/main.yml
    +-- kubectl/
    |   +-- defaults/main.yml
    |   +-- tasks/main.yml
    +-- jenkins/
        +-- tasks/main.yml
```

```
+---------------------+                        +--------------------------+
|    Control node     |   ansible-playbook     |       Managed node       |
|      master-1       | ---------------------> |         worker-1         |
|                     |  roles-playbook.yml    |                          |
|  roles/docker       |                        |   Docker CE              |
|  roles/kubectl      |                        |   kubectl v1.28          |
|  roles/jenkins      |                        |   Jenkins (port 8080)    |
+---------------------+                        +--------------------------+
```

## Tools and Technologies

| Tool | Purpose |
|------|---------|
| Ansible roles | Reusable, self-contained units of automation |
| ansible-galaxy | Generates the standard role directory skeleton |
| Role defaults | Overridable variables with the lowest precedence |
| apt, apt_repository, get_url, stat | Repositories, signing keys, and packages |
| service, user, alternatives | Service management, group membership, default Java selection |

## Implementation

All commands are executed on the control node from `~/ansible-lab`.

### Step 1: Create the role skeletons

```bash
mkdir -p roles && cd roles
ansible-galaxy init docker
ansible-galaxy init kubectl
ansible-galaxy init jenkins
cd ..
```

`ansible-galaxy init` generates the standard role layout (`tasks`, `defaults`, `handlers`, `vars`, `meta`, and others). Only the directories needed by each role are populated in this lab: `tasks` for all three roles and `defaults` for kubectl.

<img width="957" height="226" alt="image" src="https://github.com/user-attachments/assets/2382fe1c-0255-4c62-a05c-8659c292b3a6" />

### Step 2: Define the Docker role

`roles/docker/tasks/main.yml`:

```yaml
---
- name: Install prerequisites
  apt:
    name: [ca-certificates, curl, gnupg]
    state: present
    update_cache: true

- name: Create keyrings dir
  file:
    path: /etc/apt/keyrings
    state: directory
    mode: '0755'

- name: Check if Docker key already exists
  stat:
    path: /etc/apt/keyrings/docker.gpg
  register: docker_key

- name: Add Docker apt key
  get_url:
    url: https://download.docker.com/linux/ubuntu/gpg
    dest: /etc/apt/keyrings/docker.asc
    mode: '0644'
  when: not docker_key.stat.exists

- name: Add Docker repo
  apt_repository:
    repo: "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
    filename: docker
  when: not docker_key.stat.exists

- name: Install Docker
  apt:
    name: docker-ce
    state: present
    update_cache: true

- name: Start and enable Docker
  service:
    name: docker
    state: started
    enabled: true

- name: Add user to docker group
  user:
    name: "{{ ansible_user }}"
    groups: docker
    append: true
```

The role installs Docker CE, enables the service, and adds the connection user to the `docker` group. The repository is added only when the Docker signing key is not already present, because the managed node already has the Docker repository configured and apt rejects the same repository declared with two different keys.

<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/f2a6e8bf-becb-4171-ae92-ecd77db3c424" />

### Step 3: Define the kubectl role

`roles/kubectl/defaults/main.yml`:

```yaml
---
k8s_minor: "v1.28"
```

`roles/kubectl/tasks/main.yml`:

```yaml
---
- name: Install prerequisites
  apt:
    name: [ca-certificates, curl, gnupg, apt-transport-https]
    state: present
    update_cache: true

- name: Create keyrings dir
  file:
    path: /etc/apt/keyrings
    state: directory
    mode: '0755'

- name: Check if Kubernetes key already exists
  stat:
    path: /etc/apt/keyrings/kubernetes-apt-keyring.gpg
  register: k8s_key

- name: Add Kubernetes apt key
  get_url:
    url: "https://pkgs.k8s.io/core:/stable:/{{ k8s_minor }}/deb/Release.key"
    dest: /etc/apt/keyrings/kubernetes-apt-keyring.asc
    mode: '0644'
  when: not k8s_key.stat.exists

- name: Add Kubernetes repo
  apt_repository:
    repo: "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.asc] https://pkgs.k8s.io/core:/stable:/{{ k8s_minor }}/deb/ /"
    filename: kubernetes
  when: not k8s_key.stat.exists

- name: Install kubectl
  apt:
    name: kubectl
    state: present
    update_cache: true
```

The Kubernetes minor version is exposed as the `k8s_minor` default variable, so the role can target a different release without editing its tasks. It is set to `v1.28` to match the cluster, since kubectl should stay within one minor version of the API server.

<img width="1960" height="1206" alt="image" src="https://github.com/user-attachments/assets/2cac31c3-d085-443b-a1d8-3ae15c8663bc" />

### Step 4: Define the Jenkins role

`roles/jenkins/tasks/main.yml`:

```yaml
---
- name: Install Java 21 and fontconfig
  apt:
    name: [openjdk-21-jre, fontconfig, curl, gnupg]
    state: present
    update_cache: true

- name: Set Java 21 as default
  alternatives:
    name: java
    path: /usr/lib/jvm/java-21-openjdk-amd64/bin/java

- name: Create keyrings dir
  file:
    path: /etc/apt/keyrings
    state: directory
    mode: '0755'

- name: Add Jenkins apt key
  get_url:
    url: https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
    dest: /etc/apt/keyrings/jenkins-keyring.asc
    mode: '0644'

- name: Add Jenkins repo
  apt_repository:
    repo: "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/"
    filename: jenkins

- name: Install Jenkins
  apt:
    name: jenkins
    state: present
    update_cache: true

- name: Start and enable Jenkins
  service:
    name: jenkins
    state: started
    enabled: true
```

Jenkins requires Java 21 or newer, so the role installs OpenJDK 21 and selects it as the default Java. The Jenkins stable repository is added with the current signing key, then Jenkins is installed and enabled.

<img width="1756" height="1173" alt="image" src="https://github.com/user-attachments/assets/b416514e-54c5-4a07-8ead-e8c881bfbff5" />

### Step 5: Write the playbook that runs the roles

`roles-playbook.yml`:

```yaml
---
- name: Run roles on managed node
  hosts: managed
  become: true
  roles:
    - docker
    - kubectl
    - jenkins
```

The playbook contains no tasks of its own. It selects the target group, enables privilege escalation, and lists the roles in execution order.

<img width="1012" height="282" alt="image" src="https://github.com/user-attachments/assets/6395726e-2309-4829-85cd-f1490f0e54d5" />

### Step 6: Validate and preview

```bash
find roles -maxdepth 3 -name main.yml | sort
ansible-playbook roles-playbook.yml --syntax-check
ansible-playbook roles-playbook.yml --list-tasks
```

The syntax check parses the playbook and roles. `--list-tasks` prints every task that will run, prefixed with the role that owns it, without executing anything.

<img width="1489" height="1291" alt="image" src="https://github.com/user-attachments/assets/21b2aab4-5353-42a3-b869-df0130666bb8" />

### Step 7: Run the playbook

```bash
ansible-playbook roles-playbook.yml
```

Expected play recap:

```
node1 : ok=18  changed=0  unreachable=0  failed=0  skipped=4  rescued=0  ignored=0
```

All tasks report `ok` with no changes because the managed node was already configured by the playbook in Lab 25. This confirms the roles are idempotent: they describe a desired state and make no modifications when the node already matches it. The four skipped tasks are the conditional repository tasks, skipped because the Docker and Kubernetes repositories already exist.

<img width="1558" height="1386" alt="image" src="https://github.com/user-attachments/assets/5f10c92a-0ee3-446c-83a5-759b4753a528" />

## Verification

```bash
ansible managed -m command -a "docker --version"
ansible managed -m command -a "kubectl version --client"
ansible managed -b -m command -a "systemctl is-active jenkins"
```

| Check | Result |
|-------|--------|
| Docker | Docker version 29.9.3 |
| kubectl | Client Version v1.28.15 |
| Jenkins service | active |

<img width="1131" height="238" alt="image" src="https://github.com/user-attachments/assets/1f244004-a969-4ba7-8893-09f573f57a7a" />

## Conclusion

The configuration from Lab 25 was reorganized into three independent roles, each responsible for a single component, and a short playbook applies them in order. Compared with a single long playbook, this structure keeps each component isolated, makes the roles reusable across other playbooks and hosts, and lets variables such as the Kubernetes version be changed in one place. The run completed with no failures and no changes, confirming that the roles are idempotent.
