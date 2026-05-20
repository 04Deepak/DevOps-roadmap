# DevOps Project-Based Roadmap (Beginner to Advanced)

> **Audience:** Students with no prior DevOps experience
> **Format:** Project-based learning path — each section builds on the previous one
> **Estimated Duration:** 8-10 months (self-paced)
> **Difficulty:** Beginner → Intermediate → Advanced (marked per section)

---

## 1. Foundational Skills: Linux & Networking `[Beginner]` `[Weeks 1-3]`

### Topics:
- **Linux CLI:** basic commands (`cd`, `ls`, `mkdir`, `rm`, `cp`, `mv`), file permissions (`chmod`, `chown`), user management, process management (`ps`, `top`, `kill`), package management (`apt`, `yum`)
- **Shell Scripting:** variables, loops, conditionals (`if/else`), functions, input/output redirection, cron jobs
- **Networking Basics:** IP addresses, DNS, ports, HTTP/HTTPS, common network commands (`ping`, `netstat`, `curl`, `ssh`)
- **Operating System Concepts:** processes, threads, memory management, file systems
- **SSH:** key-based authentication, `ssh-keygen`, `scp`, `ssh-copy-id`
- **Firewalls:** `iptables`, `ufw` basics

### Projects:
- Create a shell script to automate daily backups of specific directories to another location, including timestamping
- Configure a basic Apache or Nginx web server on a Linux VM, ensuring it's accessible via HTTP
- Write a script to monitor disk space and send an alert (e.g., print to console) if it falls below a threshold

---

## 2. Version Control with Git `[Beginner]` `[Weeks 4-5]`

### Topics:
- **Git Basics:** `git init`, `add`, `commit`, `push`, `pull`, `clone`, `status`, `log`
- **Branching Strategies:** GitFlow, GitHub Flow, feature branching
- **Merging & Rebasing:** understanding differences and use cases, conflict resolution
- **Remote Repositories:** GitHub, GitLab, Bitbucket
- **Best Practices:** meaningful commit messages, `.gitignore`, tagging

### Projects:
- Initialize a Git repository for a simple "Hello World" application and perform common operations (commits, branches, merges)
- Collaborate on a mock project with another person (or simulate by yourself) using branches for new features and resolving merge conflicts
- Set up a `.gitignore` file for a typical project to exclude unnecessary files

---

## 3. Scripting for Automation (Python) `[Beginner]` `[Weeks 6-8]`

### Topics:
- **Python Fundamentals:** syntax, data types, control flow, functions, modules
- **File I/O and OS Module:** reading/writing files, interacting with the operating system (paths, environment variables)
- **Subprocess Module:** running shell commands from Python
- **Requests Library:** making HTTP requests to interact with APIs
- **Error Handling:** try-except blocks
- **JSON/YAML parsing:** working with configuration files

### Projects:
- Write a Python script to fetch data from a public API (e.g., weather API) and parse the JSON response
- Automate log file analysis: parse a sample log file (e.g., Apache access logs) to extract specific information (e.g., top IP addresses)
- Create a Python script that orchestrates a series of shell commands, checking their exit codes

---

## 4. Containerization with Docker `[Beginner-Intermediate]` `[Weeks 9-12]`

### Topics:
- **What are Containers?:** Docker vs. Virtual Machines, benefits of containerization
- **Dockerfile:** syntax, best practices, multi-stage builds
- **Docker Commands:** `build`, `run`, `ps`, `images`, `exec`, `stop`, `rm`
- **Docker Compose:** defining and running multi-container applications
- **Docker Networking:** bridge, host, overlay networks, container-to-container communication
- **Docker Volumes:** persistent data storage, bind mounts vs named volumes
- **Image Registries:** Docker Hub, private registries (ECR, ACR, GCR)

### Projects:
- Containerize a simple web application (Node.js, Python Flask, or a static HTML site) using a Dockerfile
- Use Docker Compose to set up a web application connected to a database (e.g., Nginx + Node.js + MongoDB)
- Build a production-ready, multi-stage Dockerfile for the web application, minimizing image size

---

## 5. CI/CD Principles `[Intermediate]` `[Weeks 13-14]`

### Topics:
- **CI/CD Concepts:** what is CI, CD (Delivery), CD (Deployment), benefits, common misconceptions
- **Stages of a CI/CD Pipeline:** build, test, package, release, deploy, monitor
- **Artifact Management:** storing and versioning build outputs
- **Code Quality:** static analysis (linters), unit tests, integration tests
- **Testing Strategy:** test pyramid, when to test what
- **Branching & Release Models:** trunk-based development, release branches

### Projects:
- Design a CI/CD pipeline flowchart (on paper or using a diagram tool) for a hypothetical microservice application
- Write a document outlining the testing strategy for the CI/CD pipeline of the containerized web app from the previous section

---

## 6. CI/CD Tools `[Intermediate]` `[Weeks 15-18]`

### Topics:
- **Jenkins:**
  - Installation & configuration
  - Freestyle Jobs vs Declarative Pipelines (Jenkinsfile)
  - Scripted Pipelines
  - Plugins ecosystem
  - Agents (Master-Slave architecture)
  - Build triggers & webhooks
- **GitHub Actions:**
  - Workflow YAML syntax
  - Jobs, Steps, Actions
  - Runners/Workflows
  - Secrets & environment variables
  - Artifacts & Caching
  - Integrations with cloud services
- **GitLab CI** (optional alternative)
- **Overview of other tools:** CircleCI, Azure DevOps Pipelines, Travis CI

### Projects:
- Set up a CI pipeline (using Jenkins, GitLab CI, or GitHub Actions) for your containerized web application: on push, it should build the Docker image, run unit tests, and push the image to a registry
- Extend the CI pipeline to include a CD stage that deploys the latest image to a "staging" environment (e.g., a Docker container running on a remote VM)

---

## 7. Cloud Fundamentals (AWS / Azure / GCP) `[Intermediate]` `[Weeks 19-23]`

> Choose one provider to focus on. AWS is used as the primary example below.

### Topics:
- **Cloud Service Models:** IaaS, PaaS, SaaS, FaaS
- **Core Concepts:** Regions, Availability Zones, Edge Locations
- **Compute Services:** EC2 (AWS), Virtual Machines (Azure), Compute Engine (GCP)
- **Storage Services:** S3 (AWS), Blob Storage (Azure), Cloud Storage (GCP)
- **Networking Services:** VPC (AWS), VNet (Azure), VPC Network (GCP)
- **Identity & Access Management:** IAM (AWS), RBAC (Azure), IAM (GCP)
- **Database Services:** RDS (AWS), SQL Database (Azure), Cloud SQL (GCP)
- **DNS & Load Balancing:** Route 53, ELB (AWS)
- **Monitoring:** CloudWatch (AWS)
- **CLI & SDK:** AWS CLI basics

### Projects:
- Launch a Linux VM in your chosen cloud provider, install a web server (Apache/Nginx), and expose it to the internet securely
- Create a cloud storage bucket and upload/download some files using the cloud provider's CLI
- Configure a custom VPC/VNet with subnets and security groups to allow specific inbound/outbound traffic for your VM

### Bonus — Serverless Introduction:
- AWS Lambda basics, API Gateway, event-driven architecture
- **Project:** Build a simple serverless API with Lambda + API Gateway

---

## 8. Infrastructure as Code (IaC) `[Intermediate]` `[Weeks 24-28]`

### 8.1 Ansible — Configuration Management

#### Topics:
- Ansible architecture (control node, managed nodes, inventory)
- Playbooks, Roles, Tasks, Handlers
- Modules & Ansible Galaxy
- Vault for secrets
- Idempotency concept

#### Projects:
- Use Ansible to configure the web server on your cloud VM (install Nginx, copy configuration files, start service)

### 8.2 Terraform — Infrastructure Provisioning

#### Topics:
- **IaC Principles:** idempotency, declarative vs. imperative, state management
- HCL syntax (HashiCorp Configuration Language)
- Providers, Resources, Variables, Outputs
- State file management (local vs remote — S3 backend)
- Modules & Workspaces
- `terraform init`, `plan`, `apply`, `destroy`

#### Projects:
- Provision a complete application infrastructure using Terraform: a VPC, subnets, security groups, and an EC2 instance/VM
- Create a reusable Terraform module for deploying a common web server setup

---

## 9. Container Orchestration with Kubernetes `[Intermediate-Advanced]` `[Weeks 29-35]`

### 9.1 Kubernetes Core

#### Topics:
- **Kubernetes Architecture:**
  - Master (Control Plane): kube-apiserver, kube-scheduler, kube-controller-manager, etcd
  - Worker Node: kubelet, kube-proxy, container runtime
- **Core Kubernetes Objects:** Pods, Deployments, ReplicaSets, Services (ClusterIP, NodePort, LoadBalancer), Namespaces
- **kubectl Commands:** basic operations (`get`, `describe`, `apply`, `delete`, `logs`, `exec`)
- **Basic Networking:** Pod-to-Pod communication, Service discovery
- **Persistent Storage:** PersistentVolumes, PersistentVolumeClaims
- **ConfigMaps & Secrets:** externalized configuration

#### Projects:
- Deploy a multi-container application (e.g., your web app + database) to a local Kubernetes cluster (Minikube/K3s)
- Expose the application using a Kubernetes Service and verify accessibility
- Scale your web application horizontally by updating its Deployment configuration

### 9.2 Kubernetes Advanced

#### Topics:
- **Ingress Controllers:** Nginx Ingress, routing rules, TLS termination
- **Helm:** package manager for K8s, Helm charts, templating deployments
- **Horizontal Pod Autoscaler (HPA):** auto-scaling based on metrics
- **RBAC:** role-based access control, ClusterRoles, RoleBindings
- **Resource Limits:** CPU/memory requests and limits
- **Liveness & Readiness Probes:** health checking

#### Projects:
- Deploy an application with Helm, configure Ingress for external access
- Set up HPA to auto-scale your app based on CPU utilization
- Configure RBAC to restrict namespace access for different teams

### 9.3 Managed Kubernetes

#### Topics:
- AWS EKS / Azure AKS / GCP GKE — overview & differences
- Provisioning managed clusters with Terraform
- Node groups, auto-scaling groups

#### Projects:
- Deploy a production-grade application on EKS (or AKS/GKE) provisioned via Terraform

---

## 10. Monitoring & Logging `[Intermediate-Advanced]` `[Weeks 36-39]`

### Topics:
- **Observability vs. Monitoring:** metrics, logs, traces
- **Key Metrics:** CPU, Memory, Disk I/O, Network I/O, application-specific metrics
- **Log Management:** centralized logging, log aggregation strategies
- **Alerting Strategies:** defining thresholds, notification channels
- **Distributed Tracing:** overview (Jaeger, Zipkin)

### Tools:
- **Prometheus:** metrics collection, targets, exporters, PromQL, Alertmanager
- **Grafana:** dashboarding, visualization, data sources
- **ELK Stack** (Elasticsearch, Logstash, Kibana) / **Loki** / Splunk (overview)

### Projects:
- Set up Prometheus to scrape metrics from a running application (e.g., using Node Exporter or a custom application exporter) and visualize them in Grafana
- Implement centralized logging for your containerized application (e.g., using a simple ELK stack or Loki) and create a basic dashboard for log analysis
- Configure an alert in Prometheus (via Alertmanager) for a critical application metric (e.g., high CPU usage) and simulate a trigger

---

## 11. DevSecOps — Security in DevOps `[Advanced]` `[Weeks 40-42]`

### Topics:
- **Shift-Left Security:** integrating security early in the pipeline
- **Container Security:** image scanning (Trivy, Snyk), base image best practices, rootless containers
- **Secret Management:** HashiCorp Vault, AWS Secrets Manager — never hardcode secrets
- **SAST/DAST:** static & dynamic application security testing tools
- **Image Signing & Trust:** verifying image integrity
- **Network Policies:** restricting pod-to-pod traffic in Kubernetes
- **Compliance & Auditing:** basic audit logging, policy-as-code (OPA/Gatekeeper)

### Projects:
- Add Trivy container image scanning to your CI/CD pipeline — fail the build if critical vulnerabilities are found
- Set up HashiCorp Vault and integrate it with your Kubernetes cluster to inject secrets into pods
- Write Kubernetes NetworkPolicies to restrict traffic between namespaces

---

## 12. GitOps & Advanced Deployment Strategies `[Advanced]` `[Weeks 43-45]`

### 12.1 GitOps

#### Topics:
- **GitOps Principles:** declarative, versioned, automated, self-healing
- **ArgoCD:** Application CRDs, sync policies, rollback, multi-cluster management
- **FluxCD** (overview)
- **Git as single source of truth** for infrastructure and application state

#### Projects:
- Install ArgoCD on your Kubernetes cluster
- Set up a GitOps workflow: push a change to Git and watch ArgoCD auto-deploy it to the cluster
- Configure auto-sync and self-healing policies

### 12.2 Advanced Deployment Strategies

#### Topics:
- **Blue-Green Deployments:** zero-downtime switching between two identical environments
- **Canary Deployments:** gradual rollout to a subset of users
- **Rolling Updates:** Kubernetes default strategy, `maxSurge`, `maxUnavailable`
- **Feature Flags:** toggle features without redeploying (LaunchDarkly, Unleash overview)
- **Multi-Environment Pipelines:** dev → staging → production promotion

#### Projects:
- Implement a blue-green deployment using Kubernetes Services
- Set up a canary deployment using Ingress weight-based routing
- Create a multi-environment CI/CD pipeline that promotes builds through dev → staging → production

---

## 13. SRE Practices & Chaos Engineering `[Advanced]` `[Weeks 46-48]`

### Topics:
- **Site Reliability Engineering (SRE):** principles and philosophy
- **SLIs, SLOs, SLAs:** defining and measuring service reliability
- **Error Budgets:** balancing reliability with feature velocity
- **Incident Management:** on-call rotations, runbooks, postmortems/blameless retrospectives
- **Chaos Engineering:** testing system resilience under failure
  - Tools: Chaos Monkey, Litmus, Gremlin (overview)
- **Capacity Planning:** resource forecasting, load testing

### Projects:
- Define SLIs and SLOs for your deployed application (e.g., latency < 200ms for 99% of requests)
- Create an incident runbook for common failure scenarios
- Run a chaos experiment using Litmus on your Kubernetes cluster (e.g., kill a pod, simulate network failure) and observe system behavior

---

## 14. Capstone: End-to-End DevOps Project `[Advanced]` `[Weeks 49-52]`

### Objective:
Build a complete DevOps pipeline from scratch that demonstrates all skills learned throughout the roadmap.

### Requirements:
1. **Application:** A multi-service web application (e.g., frontend + backend API + database)
2. **Source Control:** Git repository with branching strategy (GitHub Flow)
3. **Containerization:** All services containerized with Docker, multi-stage builds
4. **CI/CD Pipeline:** Automated build → test → scan → deploy using Jenkins or GitHub Actions
5. **Cloud Infrastructure:** Provisioned on AWS using Terraform (VPC, EKS, RDS, S3)
6. **Kubernetes:** Deployed on EKS with Helm charts, Ingress, HPA, RBAC
7. **GitOps:** ArgoCD managing deployments from Git
8. **Monitoring:** Prometheus + Grafana dashboards with alerting
9. **Logging:** Centralized logging with EFK/Loki
10. **Security:** Container scanning in pipeline, secrets via Vault, network policies
11. **Documentation:** Architecture diagram, deployment guide, runbook

### Evaluation Criteria:
| Area | What to Demonstrate |
|------|-------------------|
| Automation | Everything is automated — no manual steps |
| Reproducibility | Infrastructure can be destroyed and recreated |
| Security | No hardcoded secrets, images scanned, RBAC applied |
| Observability | Metrics, logs, and alerts are in place |
| Resilience | Application recovers from pod failures |
| Documentation | Clear README, architecture diagram, runbook |

---

## Tools Summary by Category

| Category | Tools |
|----------|-------|
| OS | Linux (Ubuntu, CentOS) |
| Scripting | Bash, Python |
| Version Control | Git, GitHub, GitLab, Bitbucket |
| Containers | Docker, Docker Compose |
| CI/CD | Jenkins, GitHub Actions, GitLab CI |
| Config Management | Ansible |
| IaC | Terraform |
| Cloud | AWS (EC2, S3, VPC, IAM, RDS, EKS, Lambda), Azure, GCP |
| Orchestration | Kubernetes, Helm |
| Monitoring | Prometheus, Grafana |
| Logging | ELK/EFK Stack, Loki |
| Security | Trivy, Snyk, HashiCorp Vault, OPA/Gatekeeper |
| GitOps | ArgoCD, FluxCD |
| Chaos Engineering | Litmus, Chaos Monkey |

---

## Learning Resources

| Section | Resources |
|---------|-----------|
| Linux | Linux Journey (linuxjourney.com), OverTheWire Bandit |
| Git | Git official docs, Atlassian Git tutorials |
| Python | Automate the Boring Stuff (free online), Python.org tutorial |
| Docker | Docker official docs, Play with Docker |
| CI/CD | Jenkins docs, GitHub Actions docs |
| AWS | AWS Free Tier + AWS Skill Builder |
| Terraform | HashiCorp Learn (developer.hashicorp.com) |
| Kubernetes | Kubernetes.io tutorials, KillerCoda labs, Play with Kubernetes |
| Monitoring | Prometheus docs, Grafana tutorials |
| Security | OWASP DevSecOps Guideline, Trivy docs |
| General | DevOps Roadmap (roadmap.sh/devops) |
