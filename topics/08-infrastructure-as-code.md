# 8. Infrastructure as Code (IaC)

## Table of Contents
- [IaC Principles](#iac-principles)
- [Ansible](#ansible)
- [Terraform](#terraform)
- [Projects](#projects)

---

## IaC Principles

```
Traditional vs IaC:

Traditional:                         IaC:
┌──────────────────────┐            ┌──────────────────────┐
│ 1. SSH into server    │            │ 1. Write code         │
│ 2. Run commands       │            │ 2. Version control    │
│ 3. Edit config files  │            │ 3. Review changes     │
│ 4. Hope it works      │            │ 4. Apply automatically│
│ 5. No record of       │            │ 5. Repeatable,        │
│    what was done       │            │    documented         │
└──────────────────────┘            └──────────────────────┘

Key Principles:
- Idempotency: Running the same code multiple times = same result
- Declarative: Describe WHAT you want, not HOW to get there
- Version Controlled: Infrastructure changes tracked in Git
- Reproducible: Same code → same infrastructure every time

Ansible vs Terraform:
┌─────────────────────────────────────────────────┐
│ Tool       │ Type           │ Best For           │
├─────────────────────────────────────────────────┤
│ Ansible    │ Configuration  │ Configure servers  │
│            │ Management     │ Install software   │
│            │                │ Deploy apps        │
├─────────────────────────────────────────────────┤
│ Terraform  │ Infrastructure │ Create servers     │
│            │ Provisioning   │ Create networks    │
│            │                │ Cloud resources    │
└─────────────────────────────────────────────────┘

Common workflow: Terraform creates infrastructure → Ansible configures it
```

---

## Ansible

### Installation

```bash
# Install Ansible
sudo apt update
sudo apt install -y ansible

# Or with pip
pip install ansible

# Verify
ansible --version
# Output: ansible [core 2.16.0]
```

### Inventory File

```ini
# File: inventory.ini
[webservers]
web01 ansible_host=192.168.56.10 ansible_user=ubuntu
web02 ansible_host=192.168.56.11 ansible_user=ubuntu

[dbservers]
db01 ansible_host=192.168.56.20 ansible_user=ubuntu

[all:vars]
ansible_ssh_private_key_file=~/.ssh/devops-key.pem
ansible_python_interpreter=/usr/bin/python3
```

```yaml
# File: inventory.yml (YAML format)
all:
  children:
    webservers:
      hosts:
        web01:
          ansible_host: 192.168.56.10
        web02:
          ansible_host: 192.168.56.11
    dbservers:
      hosts:
        db01:
          ansible_host: 192.168.56.20
  vars:
    ansible_user: ubuntu
    ansible_ssh_private_key_file: ~/.ssh/devops-key.pem
```

### Ad-Hoc Commands

```bash
# Ping all hosts
ansible all -i inventory.ini -m ping

# Run a command on all web servers
ansible webservers -i inventory.ini -m shell -a "uptime"

# Check disk space
ansible all -i inventory.ini -m shell -a "df -h"

# Install a package
ansible webservers -i inventory.ini -m apt -a "name=nginx state=present" --become

# Copy a file
ansible webservers -i inventory.ini -m copy -a "src=./index.html dest=/var/www/html/"

# Restart a service
ansible webservers -i inventory.ini -m service -a "name=nginx state=restarted" --become
```

### Playbooks

```yaml
# File: setup-webserver.yml
---
- name: Configure Web Servers
  hosts: webservers
  become: yes

  vars:
    app_name: devops-app
    http_port: 80
    doc_root: /var/www/html

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install required packages
      apt:
        name:
          - nginx
          - curl
          - git
        state: present

    - name: Create document root
      file:
        path: "{{ doc_root }}"
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'

    - name: Deploy website
      template:
        src: templates/index.html.j2
        dest: "{{ doc_root }}/index.html"
        owner: www-data
        group: www-data
      notify: Restart Nginx

    - name: Deploy Nginx config
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/default
      notify: Restart Nginx

    - name: Ensure Nginx is running and enabled
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Open firewall port
      ufw:
        rule: allow
        port: "{{ http_port }}"
        proto: tcp

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

### Jinja2 Templates

```html
<!-- File: templates/index.html.j2 -->
<!DOCTYPE html>
<html>
<head><title>{{ app_name }}</title></head>
<body>
  <h1>{{ app_name }}</h1>
  <p>Server: {{ ansible_hostname }}</p>
  <p>IP: {{ ansible_default_ipv4.address }}</p>
  <p>OS: {{ ansible_distribution }} {{ ansible_distribution_version }}</p>
  <p>Deployed by Ansible at {{ ansible_date_time.iso8601 }}</p>
</body>
</html>
```

```nginx
# File: templates/nginx.conf.j2
server {
    listen {{ http_port }};
    server_name {{ ansible_hostname }};
    root {{ doc_root }};
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Roles

```bash
# Create a role structure
ansible-galaxy init roles/webserver

# Structure:
# roles/webserver/
# ├── defaults/main.yml     ← Default variables
# ├── handlers/main.yml     ← Handlers
# ├── tasks/main.yml        ← Tasks
# ├── templates/             ← Jinja2 templates
# └── vars/main.yml         ← Role variables
```

```yaml
# File: roles/webserver/tasks/main.yml
---
- name: Install Nginx
  apt:
    name: nginx
    state: present
    update_cache: yes

- name: Deploy site config
  template:
    src: nginx-site.conf.j2
    dest: /etc/nginx/sites-available/default
  notify: Restart Nginx

- name: Enable and start Nginx
  service:
    name: nginx
    state: started
    enabled: yes
```

```yaml
# File: roles/webserver/defaults/main.yml
---
http_port: 80
server_name: localhost
doc_root: /var/www/html
```

```yaml
# File: roles/webserver/handlers/main.yml
---
- name: Restart Nginx
  service:
    name: nginx
    state: restarted
```

```yaml
# File: site.yml - Using the role
---
- name: Configure all web servers
  hosts: webservers
  become: yes
  roles:
    - webserver
```

```bash
# Run the playbook
ansible-playbook -i inventory.ini site.yml

# Run with verbose output
ansible-playbook -i inventory.ini site.yml -v

# Dry run (check mode)
ansible-playbook -i inventory.ini site.yml --check

# Limit to specific hosts
ansible-playbook -i inventory.ini site.yml --limit web01
```

### Ansible Vault (Secrets)

```bash
# Create an encrypted file
ansible-vault create secrets.yml
# Enter password, then add:
# db_password: SuperSecret123
# api_key: abc123def456

# Encrypt existing file
ansible-vault encrypt vars/production.yml

# Edit encrypted file
ansible-vault edit secrets.yml

# Use in playbook
ansible-playbook site.yml --ask-vault-pass
# or
ansible-playbook site.yml --vault-password-file vault-pass.txt
```

```yaml
# Using vault variables in playbook
- name: Deploy with secrets
  hosts: webservers
  become: yes
  vars_files:
    - secrets.yml
  tasks:
    - name: Set database password
      lineinfile:
        path: /app/.env
        line: "DB_PASSWORD={{ db_password }}"
```

---

## Terraform

### Installation

```bash
# macOS
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Ubuntu
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# Verify
terraform --version
# Output: Terraform v1.7.0
```

### HCL Basics (HashiCorp Configuration Language)

```hcl
# File: main.tf

# Configure the AWS provider
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  required_version = ">= 1.5.0"
}

provider "aws" {
  region = var.aws_region
}

# Variables
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "project_name" {
  description = "Project name for tagging"
  type        = string
  default     = "devops-lab"
}

# Data source - get latest Ubuntu AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

# Security Group
resource "aws_security_group" "web" {
  name        = "${var.project_name}-web-sg"
  description = "Allow HTTP and SSH"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.project_name}-web-sg"
  }
}

# EC2 Instance
resource "aws_instance" "web" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = var.instance_type
  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = <<-EOF
    #!/bin/bash
    apt-get update -y
    apt-get install -y nginx
    systemctl start nginx
    echo "<h1>Hello from Terraform!</h1>" > /var/www/html/index.html
  EOF

  tags = {
    Name        = "${var.project_name}-web"
    Environment = "dev"
    ManagedBy   = "terraform"
  }
}

# Outputs
output "instance_id" {
  value = aws_instance.web.id
}

output "public_ip" {
  value = aws_instance.web.public_ip
}

output "public_url" {
  value = "http://${aws_instance.web.public_ip}"
}
```

### Terraform Commands

```bash
# Initialize (download providers)
terraform init

# Preview changes
terraform plan
# Shows what will be created/modified/destroyed

# Apply changes
terraform apply
# Type "yes" to confirm

# Apply without confirmation
terraform apply -auto-approve

# Show current state
terraform show

# List resources in state
terraform state list

# Destroy all resources
terraform destroy

# Format code
terraform fmt

# Validate configuration
terraform validate

# Plan with variables
terraform plan -var="instance_type=t2.small"

# Use a .tfvars file
terraform plan -var-file="production.tfvars"
```

### Terraform Variables File

```hcl
# File: variables.tf
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "instance_count" {
  description = "Number of instances"
  type        = number
  default     = 1
}

variable "allowed_ports" {
  description = "Allowed inbound ports"
  type        = list(number)
  default     = [22, 80, 443]
}

variable "tags" {
  description = "Common tags"
  type        = map(string)
  default = {
    ManagedBy = "terraform"
    Project   = "devops-lab"
  }
}
```

```hcl
# File: dev.tfvars
aws_region     = "us-east-1"
environment    = "dev"
instance_count = 1
```

```hcl
# File: prod.tfvars
aws_region     = "us-east-1"
environment    = "production"
instance_count = 3
```

### State Management

```hcl
# Remote state with S3 backend
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "devops-lab/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

```bash
# Create S3 bucket for state
aws s3 mb s3://my-terraform-state-bucket
aws s3api put-bucket-versioning --bucket my-terraform-state-bucket --versioning-configuration Status=Enabled

# Create DynamoDB table for locking
aws dynamodb create-table \
  --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

# Initialize with new backend
terraform init -migrate-state
```

### Modules

```hcl
# File: modules/webserver/main.tf
variable "instance_type" {
  default = "t2.micro"
}

variable "name" {
  type = string
}

variable "vpc_id" {
  type = string
}

variable "subnet_id" {
  type = string
}

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

resource "aws_security_group" "web" {
  name   = "${var.name}-sg"
  vpc_id = var.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = [aws_security_group.web.id]

  tags = {
    Name = var.name
  }
}

output "public_ip" {
  value = aws_instance.web.public_ip
}

output "instance_id" {
  value = aws_instance.web.id
}
```

```hcl
# File: main.tf - Using the module
module "web_server_1" {
  source        = "./modules/webserver"
  name          = "web-01"
  instance_type = "t2.micro"
  vpc_id        = aws_vpc.main.id
  subnet_id     = aws_subnet.public.id
}

module "web_server_2" {
  source        = "./modules/webserver"
  name          = "web-02"
  instance_type = "t2.micro"
  vpc_id        = aws_vpc.main.id
  subnet_id     = aws_subnet.public.id
}

output "web1_ip" {
  value = module.web_server_1.public_ip
}

output "web2_ip" {
  value = module.web_server_2.public_ip
}
```

---

## Projects

### Project 1: Ansible Web Server Configuration

```bash
mkdir -p ~/ansible-project/{templates,roles}
cd ~/ansible-project
```

Full project to configure Nginx on 3 servers with a custom app deployed.

### Project 2: Terraform VPC + EC2

```hcl
# File: main.tf - Complete infrastructure
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" {
  region = "us-east-1"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = { Name = "devops-vpc" }
}

# Public Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true
  tags = { Name = "public-subnet" }
}

# Private Subnet
resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.3.0/24"
  availability_zone = "us-east-1a"
  tags = { Name = "private-subnet" }
}

# Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "devops-igw" }
}

# Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = { Name = "public-rt" }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Security Groups
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = { Name = "web-sg" }
}

# EC2 Instance
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = <<-EOF
    #!/bin/bash
    apt-get update -y
    apt-get install -y nginx
    echo "<h1>Deployed with Terraform!</h1><p>Instance: $(hostname)</p>" > /var/www/html/index.html
    systemctl start nginx
  EOF

  tags = { Name = "devops-web", ManagedBy = "terraform" }
}

output "web_url" {
  value = "http://${aws_instance.web.public_ip}"
}
```

```bash
# Deploy
terraform init
terraform plan
terraform apply

# Access the web server
curl $(terraform output -raw web_url)

# Clean up
terraform destroy
```

### Project 3: Reusable Terraform Module

Create a reusable `webserver` module that can be called multiple times with different configurations to deploy identical servers.
