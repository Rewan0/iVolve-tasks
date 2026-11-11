# Lab 27: Securing Sensitive Data with Ansible Vault

## Overview

This lab automates a MySQL deployment with an Ansible role and protects the database user password with Ansible Vault. The role installs MySQL, creates the `ivolve` database, and creates the `ivolveuser` user with all privileges on it. The password is never stored in plain text: it lives in an AES256-encrypted vault file and is decrypted only at run time.

## Objectives

- Create a `mysql` role that installs MySQL on the managed node
- Create the `ivolve` database
- Create the `ivolveuser` user with all privileges on the `ivolve` database
- Encrypt the database user password with Ansible Vault
- Validate the setup by connecting as the new user and listing databases

## Tools and Technologies

| Tool                         | Purpose                                              |
|------------------------------|------------------------------------------------------|
| Ansible 2.10.8               | Configuration management and automation              |
| Ansible Roles                | Reusable, structured automation                      |
| Ansible Vault (AES256)       | Encryption of sensitive variables                    |
| community.mysql 1.2.0        | `mysql_db` and `mysql_user` modules                  |
| MySQL Server                 | Database engine on the managed node                  |
| PyMySQL (`python3-pymysql`)  | Python driver required by the MySQL modules          |
| Ubuntu                       | Operating system of control and managed nodes        |

## Environment

| Component     | Details                      |
|---------------|------------------------------|
| Control node  | master-1                     |
| Managed node  | node1 (192.168.217.133)      |
| Project path  | `~/ansible-lab`              |

## Architecture

```
+---------------------+        SSH (ansible_key)        +---------------------+
|   Control node      | ------------------------------> |   Managed node      |
|   master-1          |                                 |   node1             |
|                     |                                 |                     |
|  mysql-playbook.yml |   role: mysql                   |  MySQL server       |
|  vault.yml (AES256) |   - install mysql-server        |  DB:   ivolve       |
|                     |   - create database             |  User: ivolveuser   |
+---------------------+   - create user (vaulted pass)  +---------------------+
```

## Project Structure

```
ansible-lab/
├── mysql-playbook.yml
├── group_vars/
│   └── all/
│       └── vault.yml          # encrypted with Ansible Vault
└── roles/
    └── mysql/
        ├── defaults/main.yml
        └── tasks/main.yml
```

## Implementation

### Step 1: Create the role skeleton

`ansible-galaxy init` generates the standard role directory layout.

```bash
cd ~/ansible-lab
ansible-galaxy init roles/mysql
```

<img width="973" height="96" alt="image" src="https://github.com/user-attachments/assets/8cec3157-90ca-4d81-becd-38bcd47e74d9" />

### Step 2: Define the role defaults

`roles/mysql/defaults/main.yml`

```yaml
---
mysql_db_name: ivolve
mysql_db_user: ivolveuser
# mysql_db_password is defined in the encrypted vault file
```

The database name and user are plain defaults. The password is intentionally absent and comes from the vault.

### Step 3: Write the role tasks

`roles/mysql/tasks/main.yml`

```yaml
---
- name: Install MySQL server and Python MySQL driver
  ansible.builtin.apt:
    name:
      - mysql-server
      - python3-pymysql
    state: present
    update_cache: true

- name: Ensure MySQL is started and enabled
  ansible.builtin.service:
    name: mysql
    state: started
    enabled: true

- name: Create ivolve database
  community.mysql.mysql_db:
    name: "{{ mysql_db_name }}"
    state: present
    login_unix_socket: /var/run/mysqld/mysqld.sock

- name: Create ivolveuser with all privileges on ivolve DB
  community.mysql.mysql_user:
    name: "{{ mysql_db_user }}"
    password: "{{ mysql_db_password }}"
    priv: "{{ mysql_db_name }}.*:ALL"
    host: localhost
    state: present
    login_unix_socket: /var/run/mysqld/mysqld.sock
  no_log: true
```

- The MySQL modules connect through the local unix socket, so no root password is needed.
- `no_log: true` keeps the password out of the playbook output.
  
<img width="1141" height="64" alt="image" src="https://github.com/user-attachments/assets/30b40672-b561-4032-99d2-0b23373d522a" />

### Step 4: Encrypt the password with Ansible Vault

```bash
mkdir -p group_vars/all
EDITOR=nano ansible-vault create group_vars/all/vault.yml
```

Content of the file (stored encrypted):

```yaml
mysql_db_password: "<your-db-password>"
```

Verify that the file is encrypted:

```bash
head -1 group_vars/all/vault.yml
```

```
$ANSIBLE_VAULT;1.1;AES256
```

View the decrypted content when needed:

```bash
ansible-vault view group_vars/all/vault.yml
```

<img width="1399" height="91" alt="image" src="https://github.com/user-attachments/assets/8df9e417-bcdf-42a6-9d01-e16b2bc73bee" />

### Step 5: Create the playbook

`mysql-playbook.yml`

```yaml
---
- name: Deploy MySQL with Vault-protected credentials
  hosts: node1
  become: true
  roles:
    - mysql
```

### Step 6: Run the playbook

```bash
ansible-playbook mysql-playbook.yml --ask-vault-pass
```

Result:

```
PLAY [Deploy MySQL with Vault-protected credentials] ***********************

TASK [Gathering Facts] *****************************************************
ok: [node1]

TASK [mysql : Install MySQL server and Python MySQL driver] ****************
changed: [node1]

TASK [mysql : Ensure MySQL is started and enabled] *************************
ok: [node1]

TASK [mysql : Create ivolve database] **************************************
changed: [node1]

TASK [mysql : Create ivolveuser with all privileges on ivolve DB] **********
changed: [node1]

PLAY RECAP *****************************************************************
node1 : ok=5  changed=3  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

<img width="2455" height="663" alt="image" src="https://github.com/user-attachments/assets/133b26d2-b7f2-43be-b5de-896d4af135cf" />

## Validation

Connect to MySQL on the managed node as `ivolveuser` and list the databases:

```bash
ansible node1 -b -m shell \
  -a "mysql -u ivolveuser -p'<your-db-password>' -e 'SHOW DATABASES;'" \
  --ask-vault-pass
```

Output:

```
node1 | CHANGED | rc=0 >>
Database
information_schema
ivolve
performance_schema
```

The `ivolve` database is visible to `ivolveuser`, which confirms that the database, the user, and its privileges were created correctly.

<img width="1506" height="313" alt="image" src="https://github.com/user-attachments/assets/88f267ed-9bf6-4f6b-a737-29f9f5917f7f" />

## Requirements Checklist

| Requirement                                   | Status |
|-----------------------------------------------|--------|
| MySQL installed through the `mysql` role      | Done   |
| `ivolve` database created                     | Done   |
| `ivolveuser` created with ALL on `ivolve`     | Done   |
| Password encrypted with Ansible Vault         | Done   |
| Connection validated and databases listed     | Done   |

## Conclusion

The lab delivered a reusable `mysql` role that installs MySQL, creates the `ivolve` database, and provisions `ivolveuser` with full privileges on it. Ansible Vault keeps the database password encrypted at rest, and `no_log` keeps it out of the run output, so the repository can hold the automation code without exposing credentials. Logging in as `ivolveuser` and listing the databases confirmed the deployment end to end.

Operational notes:

- Never commit the vault password or a decrypted `vault.yml` to version control. The encrypted file itself is safe to commit.
- For non-interactive runs, keep the vault password in a file outside the repository and use `--vault-password-file`.
