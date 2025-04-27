# End-to-End-Infrastructure-Provisioning-Using-Terraform-Ansible

---

# 📦 End-to-End Infrastructure Provisioning Using Terraform & Ansible

---

## ✅ Project Objective

Provision a complete Kubernetes-ready infrastructure on **AWS** using **Terraform** and configure it using **Ansible**, while managing secrets securely with **Vault**.

---

## 🛠️ Technologies Used

- **Terraform** – Infrastructure provisioning
- **Ansible** – Server configuration & software installation
- **Vault** – Secret Management
- **AWS** – Public Cloud (VPC, EC2, EKS)
- **Linux** – Server administration
- **Git** – Version control

---

## 🧱 Project Structure

```bash
infra-terraform-ansible/
├── terraform/
│   ├── modules/
│   │   ├── vpc/
│   │   ├── ec2/
│   │   └── eks/
│   ├── environments/
│   │   └── dev/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       └── outputs.tf
├── ansible/
│   ├── inventory/
│   │   └── hosts.ini
│   ├── playbooks/
│   │   ├── install-monitoring.yml
│   │   ├── configure-ssh.yml
│   ├── roles/
│   │   ├── prometheus/
│   │   ├── grafana/
│   │   ├── users/
│   │   └── common/
├── vault/
│   └── vault_setup.sh
├── scripts/
│   └── provision.sh
├── README.md
```
---


## create_infra_project.py

```python
import os

# Define base directory
base_dir = "/home/lilia/VIDEOS/infra-terraform-ansible"

# Files structure with content
files_structure = {
    "terraform/modules/vpc/main.tf": '''
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = {
    Name = "k8s-vpc"
  }
}
''',

    "terraform/modules/ec2/main.tf": '''
resource "aws_instance" "bastion" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id
  key_name      = var.key_name

  tags = {
    Name = "BastionHost"
  }
}
''',

    "terraform/modules/eks/main.tf": '''
module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = var.cluster_name
  cluster_version = "1.28"
  subnets         = var.subnets
  vpc_id          = var.vpc_id
}
''',

    "terraform/environments/dev/main.tf": '''
provider "aws" {
  region = "us-east-1"
}

module "vpc" {
  source = "../../modules/vpc"
  vpc_cidr = "10.0.0.0/16"
}

module "ec2" {
  source = "../../modules/ec2"
  ami_id = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
  subnet_id = module.vpc.subnet_id
  key_name = "your-key-name"
}

module "eks" {
  source = "../../modules/eks"
  cluster_name = "dev-cluster"
  subnets = [module.vpc.subnet_id]
  vpc_id = module.vpc.vpc_id
}
''',

    "terraform/environments/dev/variables.tf": '''
variable "vpc_cidr" {}
variable "ami_id" {}
variable "instance_type" {}
variable "subnet_id" {}
variable "key_name" {}
variable "cluster_name" {}
variable "subnets" {}
variable "vpc_id" {}
''',

    "terraform/environments/dev/outputs.tf": '''
output "vpc_id" {
  value = module.vpc.vpc_id
}

output "bastion_public_ip" {
  value = module.ec2.public_ip
}

output "eks_cluster_name" {
  value = module.eks.cluster_name
}
''',

    "ansible/inventory/hosts.ini": '''
[bastion]
ec2-bastion-public-ip

[eks_nodes]
ec2-node1-public-ip
ec2-node2-public-ip
''',

    "ansible/playbooks/install-monitoring.yml": '''
- hosts: eks_nodes
  become: yes
  roles:
    - prometheus
    - grafana
''',

    "ansible/playbooks/configure-ssh.yml": '''
- hosts: all
  become: yes
  roles:
    - users
    - common
''',

    "ansible/roles/prometheus/tasks/main.yml": '''
- name: Install Prometheus
  apt:
    name: prometheus
    state: present
''',

    "ansible/roles/grafana/tasks/main.yml": '''
- name: Install Grafana
  apt:
    name: grafana
    state: present
''',

    "ansible/roles/users/tasks/main.yml": '''
- name: Create system users
  user:
    name: devuser
    state: present
    groups: sudo
    shell: /bin/bash
''',

    "ansible/roles/common/tasks/main.yml": '''
- name: Install basic packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - git
    - curl
    - htop
''',

    "vault/vault_setup.sh": '''#!/bin/bash

vault server -dev &

sleep 5

export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'

vault secrets enable -path=secret kv

vault kv put secret/aws access_key="your_access_key" secret_key="your_secret_key"
''',

    "scripts/provision.sh": '''#!/bin/bash

echo "==> Provisioning Infrastructure..."
cd terraform/environments/dev/
terraform init && terraform apply -auto-approve

echo "==> Configuring Instances with Ansible..."
cd ../../../ansible/
ansible-playbook -i inventory/hosts.ini playbooks/configure-ssh.yml
ansible-playbook -i inventory/hosts.ini playbooks/install-monitoring.yml

echo "==> Infrastructure Setup Complete!"
'''
}

# Create all files and folders
for relative_path, content in files_structure.items():
    full_path = os.path.join(base_dir, relative_path)
    os.makedirs(os.path.dirname(full_path), exist_ok=True)
    
    with open(full_path, 'w') as f:
        f.write(content.strip())
    
    # Make scripts executable
    if full_path.endswith('.sh'):
        os.chmod(full_path, 0o755)

print(f"✅ All folders and files have been created successfully under {base_dir}")

```

---

## 🚀 Step-by-Step Implementation

---

### 📌 Step 1: Provision Infrastructure with Terraform

- Create reusable Terraform **modules** for:
  - VPC
  - EC2 instances (Bastion + Worker Nodes)
  - EKS Cluster
- Create **environment-specific files** (e.g., `dev/main.tf`) to orchestrate the deployment.

**Terraform Commands:**

```bash
cd terraform/environments/dev/

# Initialize providers and modules
terraform init

# Preview changes
terraform plan

# Apply changes (create infra)
terraform apply
```

---

### 📌 Step 2: Configure Instances with Ansible

- Create an **inventory** file listing EC2 nodes.
- Create **playbooks** to:
  - Configure SSH access
  - Install required packages
  - Install Prometheus and Grafana for monitoring
- Use **roles** for better modularity and reusability.

**Ansible Commands:**

```bash
cd ansible/

# Configure SSH, users, and common packages
ansible-playbook -i inventory/hosts.ini playbooks/configure-ssh.yml

# Install Prometheus and Grafana
ansible-playbook -i inventory/hosts.ini playbooks/install-monitoring.yml
```

---

### 📌 Step 3: Secure Secrets with Vault

- Use Vault to securely manage AWS keys, database passwords, etc.
- Run a Vault dev server and create a KV secrets engine.

**Vault Commands:**

```bash
# Install Vault
sudo apt install vault

# Run Vault setup script
bash vault/vault_setup.sh

# Verify secrets
vault kv get secret/aws
```

---

### 📌 Step 4: Full Automation with Bash Script

Run everything from start to finish with a single script:

```bash
bash scripts/provision.sh
```

---

## 🔥 Full Workflow Summary

| Step | Task | Tool |
|:----:|:----:|:----:|
| 1 | Provision VPC, EC2, EKS | Terraform |
| 2 | Configure nodes, install monitoring | Ansible |
| 3 | Manage secrets securely | Vault |
| 4 | Full automated provisioning | Bash Script |

---

## 🧠 What You'll Learn

- Infrastructure-as-Code using Terraform modules
- Configuration Management with Ansible (roles, handlers, playbooks)
- Secrets management using Vault
- Best practices for organizing DevOps projects
- Real-world end-to-end deployment workflows
- Cloud Infrastructure Automation on AWS

---

## 📷 Overview Diagram

```
Terraform (Provisioning)
        ↓
AWS Infrastructure (VPC, EC2, EKS)
        ↓
Ansible (Configuration: SSH, Monitoring)
        ↓
Vault (Secrets Management)
```

---

## 📝 Prerequisites

- AWS Account with permissions (EC2, VPC, EKS, IAM)
- Terraform installed (`>= 1.0.0`)
- Ansible installed
- Vault installed
```bash
# Ubuntu/Debian
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
sudo apt-add-repository "deb [arch=amd64 signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt update && sudo apt install vault
vault --version

```
- Git installed
- SSH Key pair (for EC2 access)

---

## 💡 Notes

- Vault is running in **Dev mode** for simplicity. In production, always run Vault in HA mode with storage backend (e.g., Consul, S3).
- SSH key must be configured before applying Ansible playbooks.
- Always validate and format Terraform configs:  
  ```bash
  terraform validate
  terraform fmt
  ```
- Store your `.tfstate` file securely or use remote backends like S3 + DynamoDB.

---

---

# 🌟 Thank you! Happy Automating! 🚀

---
