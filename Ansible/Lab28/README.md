# Lab 28: Automated Host Discovery with Ansible Dynamic Inventory

## Overview

This lab replaces a static inventory with an Ansible dynamic inventory that discovers EC2 instances automatically. Instances are tagged `server: ivolve`, and the `aws_ec2` inventory plugin queries the EC2 API, filters the running instances by that tag, and exposes them as Ansible hosts. A playbook is then run against the discovered host and its result is verified.

The EC2 API used in this lab is provided by LocalStack, a local AWS emulator running in Docker. The inventory plugin, the filters, and the playbook are identical to what is used with real AWS. The only differences are the endpoint the plugin talks to and the connection method, as described in the Notes section.

## Objectives

- Create an EC2 instance with the tag `server: ivolve`
- Set up an Ansible dynamic inventory that discovers running instances with that tag
- List the target hosts using the `ansible-inventory` command
- Run a playbook on the discovered instance
- Verify the configuration on the instance

## Tools and Technologies

| Tool                          | Purpose                                                   |
|-------------------------------|-----------------------------------------------------------|
| Ansible 2.10.8                | Automation engine                                         |
| `amazon.aws.aws_ec2` plugin   | Dynamic inventory based on EC2 instances and tags         |
| LocalStack 3.8.1              | Local emulation of the AWS EC2 API                        |
| Docker                        | Runs the LocalStack container                             |
| Python 3 and boto3 1.44.0     | AWS SDK used by the inventory plugin and the setup script |

## Environment

| Component         | Details                                    |
|-------------------|--------------------------------------------|
| Control node      | master-1                                   |
| EC2 API endpoint  | `http://localhost:4566` (LocalStack)       |
| Region            | us-east-1                                  |
| Project path      | `~/ansible-lab28`                          |

## Architecture

```
+-----------------------------------------------+
|                master-1                       |
|                                               |
|  ansible-inventory / ansible-playbook         |
|        |                                      |
|        |  aws_ec2 plugin (boto3)              |
|        v                                      |
|  http://localhost:4566  --->  LocalStack      |
|                               (Docker)        |
|                               EC2 API         |
|                               - ivolve-server (server=ivolve)  <-- discovered
|                               - other-server  (server=other)   <-- filtered out
+-----------------------------------------------+
```

## Project Structure

```
ansible-lab28/
├── env.sh                # environment variables for the EC2 API endpoint
├── create_instances.py   # creates the tagged EC2 instances
├── aws_ec2.yml           # dynamic inventory configuration
└── ec2-playbook.yml      # playbook run on the discovered host
```

## Implementation

### Step 1: Start the EC2 API

LocalStack provides the EC2 API locally in a Docker container.

```bash
docker run -d --name localstack -p 4566:4566 localstack/localstack:3.8.1
curl -s localhost:4566/_localstack/health | head -c 100
```

The health output lists `"ec2": "available"`.

<img width="2559" height="154" alt="image" src="https://github.com/user-attachments/assets/801615cd-e228-4dc5-81a6-e403932f0751" />

### Step 2: Set the environment variables

`env.sh`

```bash
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
export AWS_ENDPOINT_URL=http://localhost:4566
```

```bash
pip3 install --user --upgrade boto3 botocore
source env.sh
```

LocalStack accepts any credentials. `AWS_ENDPOINT_URL` points boto3, and therefore the inventory plugin, to the local EC2 API instead of AWS.

<img width="467" height="109" alt="Screenshot 2026-11-12 014908" src="https://github.com/user-attachments/assets/f185634e-4bc7-4112-99ab-efe8495715f7" />

### Step 3: Create the EC2 instances

`create_instances.py` creates the target instance tagged `server: ivolve`, plus a second instance tagged `server: other` to prove that the inventory filter works.

```python
import boto3

ec2 = boto3.client("ec2")

for name, value in [("ivolve-server", "ivolve"), ("other-server", "other")]:
    ec2.run_instances(
        ImageId="ami-ff0fea8310f3", InstanceType="t2.micro",
        MinCount=1, MaxCount=1,
        TagSpecifications=[{
            "ResourceType": "instance",
            "Tags": [{"Key": "Name", "Value": name},
                     {"Key": "server", "Value": value}],
        }],
    )

for r in ec2.describe_instances()["Reservations"]:
    for i in r["Instances"]:
        tags = {t["Key"]: t["Value"] for t in i.get("Tags", [])}
        print(i["InstanceId"], i["State"]["Name"], "server=" + str(tags.get("server")))
```

```bash
python3 create_instances.py
```

```
i-2006186891a359d46 running server=ivolve
i-f3144722bf2d44915 running server=other
```

<img width="922" height="94" alt="image" src="https://github.com/user-attachments/assets/82c16463-6964-4f6a-835b-c3ddc6f792e8" />

### Step 4: Configure the dynamic inventory

`aws_ec2.yml`

```yaml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
filters:
  tag:server: ivolve
  instance-state-name: running
hostnames:
  - tag:Name
```

| Setting                          | Meaning                                              |
|----------------------------------|------------------------------------------------------|
| `plugin`                         | Use the EC2 dynamic inventory plugin                 |
| `regions`                        | Region to query                                      |
| `filters: tag:server: ivolve`    | Keep only instances tagged `server: ivolve`          |
| `filters: instance-state-name`   | Keep only running instances                          |
| `hostnames: tag:Name`            | Name each host after its `Name` tag                  |

The inventory file name must end with `aws_ec2.yml` for the plugin to accept it.

<img width="946" height="307" alt="image" src="https://github.com/user-attachments/assets/cc89971d-ae92-472d-b141-a4d530082ab7" />

### Step 5: List the target hosts

```bash
ansible-inventory -i aws_ec2.yml --graph
```

```
@all:
  |--@aws_ec2:
  |  |--ivolve-server
  |--@ungrouped:
```

Only `ivolve-server` is discovered. `other-server` is excluded by the tag filter.

```bash
ansible-inventory -i aws_ec2.yml --list
```

The full listing shows the host variables collected from the EC2 API, for example:

```
"instance_id": "i-2006186891a359d46",
"instance_type": "t2.micro",
"state": { "name": "running" },
"tags": { "Name": "ivolve-server", "server": "ivolve" }
```

<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/8da3eb8a-54d8-4f1b-a129-1147ca144bca" />

### Step 6: Run a playbook on the discovered host

`ec2-playbook.yml`

```yaml
---
- name: Configure the discovered ivolve EC2 instance
  hosts: aws_ec2
  connection: local
  gather_facts: false
  tasks:
    - name: Show discovered instance details
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} | {{ instance_id }} | {{ public_ip_address }} | server={{ tags.server }}"

    - name: Write configuration file for the instance
      ansible.builtin.copy:
        dest: /tmp/ivolve-ec2.txt
        content: |
          host: {{ inventory_hostname }}
          instance_id: {{ instance_id }}
          public_ip: {{ public_ip_address }}
          server_tag: {{ tags.server }}
```

```bash
ansible-playbook -i aws_ec2.yml ec2-playbook.yml
```

```
PLAY [Configure the discovered ivolve EC2 instance] ************************

TASK [Show discovered instance details] ************************************
ok: [ivolve-server] => {
    "msg": "ivolve-server | i-2006186891a359d46 | 54.214.1.54 | server=ivolve"
}

TASK [Write configuration file for the instance] ***************************
changed: [ivolve-server]

PLAY RECAP *****************************************************************
ivolve-server : ok=2  changed=1  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

The playbook targets the `aws_ec2` group, so it runs only on hosts discovered dynamically. No host was listed manually anywhere.

<img width="2242" height="925" alt="image" src="https://github.com/user-attachments/assets/b5d8150e-e6b4-4755-97cb-753ae4288160" />

## Verification

```bash
ansible -i aws_ec2.yml aws_ec2 -c local -m command -a "cat /tmp/ivolve-ec2.txt"
```

```
ivolve-server | CHANGED | rc=0 >>
host: ivolve-server
instance_id: i-2006186891a359d46
public_ip: 54.214.1.54
server_tag: ivolve
```

The configuration file contains the instance details taken from the dynamic inventory, which confirms that the playbook ran on the discovered host.

<img width="1486" height="162" alt="image" src="https://github.com/user-attachments/assets/172b6d1a-8276-4026-b24c-40d8d601acc5" />

## Requirements Checklist

| Requirement                                               | Status |
|-----------------------------------------------------------|--------|
| EC2 instance created with tag `server: ivolve`            | Done   |
| Dynamic inventory discovers running instances by the tag  | Done   |
| Target hosts listed with `ansible-inventory`              | Done   |
| Playbook run on the discovered instance                   | Done   |
| Configuration verified                                    | Done   |

## Notes

- The EC2 API in this lab is emulated by LocalStack, so the instance is an API record and not a virtual machine. For that reason the playbook uses `connection: local` and runs on the control node, using the discovered instance data.
- On a real AWS account the setup is the same, with two changes: remove `AWS_ENDPOINT_URL` and use real credentials, and replace `connection: local` with SSH access to the instance (`-u <user> --private-key <key>.pem`).
- The credentials in `env.sh` are dummy values required by LocalStack. Real AWS keys must never be committed to a repository.
- LocalStack 3.8.1 is used because newer `latest` images require an authentication token.

## Conclusion

The `aws_ec2` dynamic inventory removed the need to maintain a host list by hand. Ansible discovered the instance tagged `server: ivolve`, ignored the instance with a different tag, and exposed the instance attributes as host variables. The playbook ran on the discovered host and wrote a configuration file built from those variables, and the verification confirmed its content. The same inventory file and playbook structure carry over to a real AWS account with minimal changes.
