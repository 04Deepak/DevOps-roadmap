# 11. DevSecOps - Security in DevOps

## Table of Contents
- [Shift-Left Security](#shift-left-security)
- [Container Image Scanning with Trivy](#container-image-scanning-with-trivy)
- [Secret Management with HashiCorp Vault](#secret-management-with-hashicorp-vault)
- [SAST and DAST Tools](#sast-and-dast-tools)
- [Kubernetes Network Policies](#kubernetes-network-policies)
- [Container Image Signing](#container-image-signing)
- [Policy-as-Code with OPA and Gatekeeper](#policy-as-code-with-opa-and-gatekeeper)
- [Projects](#projects)

---

## Shift-Left Security

"Shift left" means moving security practices earlier in the software development lifecycle. Instead of testing for security only before deployment, you build it into every stage.

```
┌──────────────────────────────────────────────────────────────────────┐
│                Traditional vs Shift-Left Security                     │
│                                                                       │
│  Traditional (late):                                                  │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────────┐  ┌────────┐ │
│  │ Code  │─►│Build │─►│ Test │─►│Deploy│─►│ Security │─►│  Prod  │ │
│  └──────┘  └──────┘  └──────┘  └──────┘  │  Review   │  └────────┘ │
│                                            └──────────┘              │
│  Shift-Left (early and continuous):                                   │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌────────┐               │
│  │ Code  │─►│Build │─►│ Test │─►│Deploy│─►│  Prod  │               │
│  │ +SAST │  │+Image│  │+DAST │  │+Policy│  │+Monitor│               │
│  │ +Lint │  │ Scan │  │+Pen  │  │ Check│  │+Audit  │               │
│  └──────┘  └──────┘  └──────┘  └──────┘  └────────┘               │
│     ▲          ▲          ▲         ▲          ▲                     │
│     └──────────┴──────────┴─────────┴──────────┘                     │
│              Security integrated at EVERY stage                       │
└──────────────────────────────────────────────────────────────────────┘
```

### Key DevSecOps Practices

| Practice | Stage | Tools |
|----------|-------|-------|
| Secret scanning | Code | GitLeaks, TruffleHog |
| Static analysis (SAST) | Code | SonarQube, Semgrep, Bandit |
| Dependency scanning | Build | Snyk, Dependabot, Trivy |
| Container image scanning | Build | Trivy, Grype, Clair |
| Dynamic analysis (DAST) | Test | OWASP ZAP, Burp Suite |
| Infrastructure scanning | Deploy | Checkov, tfsec, Trivy |
| Network policies | Runtime | Kubernetes NetworkPolicy, Calico |
| Secret management | Runtime | HashiCorp Vault, AWS Secrets Manager |
| Policy enforcement | Runtime | OPA/Gatekeeper, Kyverno |
| Runtime monitoring | Production | Falco, Sysdig |

---

## Container Image Scanning with Trivy

Trivy is a comprehensive security scanner by Aqua Security. It scans container images, filesystems, Git repositories, and Kubernetes clusters for vulnerabilities, misconfigurations, and exposed secrets.

### Install Trivy

```bash
# macOS
brew install trivy

# Ubuntu/Debian
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

# Run via Docker (no installation required)
docker run --rm aquasec/trivy:latest image python:3.11

# Verify installation
trivy --version
# Version: 0.50.0
```

### Scanning Container Images

```bash
# Scan a public image for vulnerabilities
trivy image python:3.11
# Output shows vulnerabilities by severity: CRITICAL, HIGH, MEDIUM, LOW

# Scan and only show CRITICAL and HIGH severity
trivy image --severity CRITICAL,HIGH python:3.11

# Scan and fail if CRITICAL vulnerabilities are found (useful in CI/CD)
trivy image --exit-code 1 --severity CRITICAL python:3.11
# Exit code 1 means vulnerabilities found -> pipeline fails

# Scan a locally built image
docker build -t myapp:latest .
trivy image myapp:latest

# Output results as JSON for programmatic processing
trivy image --format json --output results.json python:3.11

# Output as a table (default) with full details
trivy image --format table python:3.11

# Scan with a specific vulnerability database
trivy image --skip-db-update python:3.11

# Ignore unfixed vulnerabilities (show only what can be patched)
trivy image --ignore-unfixed python:3.11
```

**Example output:**

```
python:3.11 (debian 12.4)
Total: 287 (CRITICAL: 3, HIGH: 25, MEDIUM: 102, LOW: 157)

┌──────────────────┬──────────────────┬──────────┬──────────┬───────────────────┐
│     Library      │  Vulnerability   │ Severity │  Status  │  Installed Ver    │
├──────────────────┼──────────────────┼──────────┼──────────┼───────────────────┤
│ libssl3          │ CVE-2024-0727    │ CRITICAL │ fixed    │ 3.0.11-1~deb12u1  │
│ zlib1g           │ CVE-2023-45853   │ HIGH     │ fixed    │ 1:1.2.13.dfsg-1   │
│ curl             │ CVE-2024-2004    │ MEDIUM   │ fixed    │ 7.88.1-10+deb12u4 │
└──────────────────┴──────────────────┴──────────┴──────────┴───────────────────┘
```

### Scanning Beyond Images

```bash
# Scan a filesystem / project directory for vulnerabilities in dependencies
trivy fs --scanners vuln,secret,misconfig ./my-project

# Scan a Git repository (remote)
trivy repo https://github.com/example/my-app

# Scan Kubernetes cluster for misconfigurations
trivy k8s --report summary cluster

# Scan a Dockerfile for misconfigurations
trivy config ./Dockerfile

# Scan Terraform files for misconfigurations
trivy config ./terraform/

# Scan for exposed secrets in a directory
trivy fs --scanners secret ./my-project
```

### Trivy in CI/CD Pipelines

#### GitHub Actions

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "myapp:${{ github.sha }}"
          format: "table"
          exit-code: "1"                  # Fail the build on findings
          severity: "CRITICAL,HIGH"       # Only fail on critical/high
          ignore-unfixed: true            # Skip unfixed vulns

      - name: Run Trivy for IaC scanning
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: "config"
          scan-ref: "."
          format: "table"
          exit-code: "1"
          severity: "CRITICAL,HIGH"
```

#### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - build
  - security

build:
  stage: build
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

trivy-scan:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image
        --exit-code 1
        --severity CRITICAL,HIGH
        --ignore-unfixed
        --format table
        $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  allow_failure: false
```

#### Jenkins Pipeline

```groovy
// Jenkinsfile
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'docker build -t myapp:${BUILD_NUMBER} .'
            }
        }
        stage('Security Scan') {
            steps {
                sh '''
                    trivy image \
                      --exit-code 1 \
                      --severity CRITICAL,HIGH \
                      --format json \
                      --output trivy-results.json \
                      myapp:${BUILD_NUMBER}
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-results.json', allowEmptyArchive: true
                }
            }
        }
    }
}
```

### Trivy Ignore File

When a vulnerability is a known false positive or cannot be fixed, you can ignore it:

```yaml
# .trivyignore.yaml
vulnerabilities:
  - id: CVE-2023-44487       # Known HTTP/2 rapid reset, mitigated at LB
    statement: "Mitigated at load balancer level"
  - id: CVE-2024-0727
    statement: "Accepted risk - no exposure path"

misconfigurations:
  - id: DS002                 # Root user in Dockerfile
    statement: "Build stage only, runtime uses non-root"
```

```bash
# Use the ignore file during scanning
trivy image --ignorefile .trivyignore.yaml myapp:latest
```

---

## Secret Management with HashiCorp Vault

HashiCorp Vault is a tool for securely storing and accessing secrets (API keys, passwords, certificates, tokens). It provides encryption, access control, and audit logging.

```
┌─────────────────────────────────────────────────────────────┐
│                     Vault Architecture                       │
│                                                              │
│  ┌──────────┐    ┌──────────────────────────────────────┐  │
│  │  App 1    │───►│                                      │  │
│  └──────────┘    │                                      │  │
│  ┌──────────┐    │           HashiCorp Vault             │  │
│  │  App 2    │───►│                                      │  │
│  └──────────┘    │  ┌─────────────┐  ┌──────────────┐  │  │
│  ┌──────────┐    │  │ Secret      │  │ Auth Methods │  │  │
│  │  CI/CD    │───►│  │ Engines     │  │              │  │  │
│  └──────────┘    │  │ - KV        │  │ - Token      │  │  │
│  ┌──────────┐    │  │ - Database  │  │ - AppRole    │  │  │
│  │  K8s Pods │───►│  │ - PKI      │  │ - Kubernetes │  │  │
│  └──────────┘    │  │ - Transit   │  │ - LDAP       │  │  │
│                   │  └─────────────┘  └──────────────┘  │  │
│                   └──────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Install Vault with Docker

```yaml
# ~/vault/docker-compose.yml
version: "3.8"

services:
  vault:
    image: hashicorp/vault:latest
    container_name: vault
    ports:
      - "8200:8200"
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: "my-root-token"
      VAULT_DEV_LISTEN_ADDRESS: "0.0.0.0:8200"
    cap_add:
      - IPC_LOCK
    volumes:
      - vault-data:/vault/data
    command: server -dev

volumes:
  vault-data:
```

```bash
# Start Vault in development mode
cd ~/vault
docker compose up -d

# Set environment variables for the Vault CLI
export VAULT_ADDR="http://127.0.0.1:8200"
export VAULT_TOKEN="my-root-token"

# Install Vault CLI
# macOS
brew install vault

# Ubuntu
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update
sudo apt-get install vault

# Verify Vault is running
vault status
# Output:
# Sealed: false
# Key Shares: 1
# Key Threshold: 1
# ...
```

### Vault Basics - Key/Value Secrets

```bash
# ---- Enable KV secrets engine (v2, enabled by default in dev mode) ----
vault secrets enable -path=secret kv-v2

# ---- Write secrets ----
vault kv put secret/myapp/database \
  username="dbadmin" \
  password="s3cureP@ss!" \
  host="db.example.com" \
  port="5432"

vault kv put secret/myapp/api \
  api_key="ak_live_abc123def456" \
  api_secret="sk_live_xyz789"

# ---- Read secrets ----
vault kv get secret/myapp/database
# Output:
# ====== Data ======
# Key         Value
# ---         -----
# host        db.example.com
# password    s3cureP@ss!
# port        5432
# username    dbadmin

# Get a specific field
vault kv get -field=password secret/myapp/database
# Output: s3cureP@ss!

# Get as JSON
vault kv get -format=json secret/myapp/database

# ---- List secrets ----
vault kv list secret/myapp
# Output:
# Keys
# ----
# api
# database

# ---- Delete a secret ----
vault kv delete secret/myapp/api

# ---- View secret versions (KV v2 supports versioning) ----
vault kv metadata get secret/myapp/database
```

### Vault Policies - Access Control

Policies define what a token can and cannot do.

```hcl
# myapp-policy.hcl - Policy for the application
# Read-only access to myapp secrets
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}

# No access to other secrets
path "secret/data/*" {
  capabilities = ["deny"]
}
```

```bash
# Write the policy to Vault
vault policy write myapp-read myapp-policy.hcl

# Create a token with this policy
vault token create -policy="myapp-read" -ttl="1h"
# Output:
# Key                Value
# ---                -----
# token              hvs.CAESIAbcdef...
# token_accessor     abc123...
# token_duration     1h
# token_policies     ["default" "myapp-read"]

# Test with the new token
export VAULT_TOKEN="hvs.CAESIAbcdef..."
vault kv get secret/myapp/database    # Works
vault kv put secret/myapp/database password="new"  # Permission denied
```

### Vault AppRole Authentication (for CI/CD and Applications)

AppRole is designed for machine-to-machine authentication.

```bash
# Switch back to root token
export VAULT_TOKEN="my-root-token"

# Enable AppRole auth method
vault auth enable approle

# Create a role for the application
vault write auth/approle/role/myapp-role \
  token_policies="myapp-read" \
  token_ttl=1h \
  token_max_ttl=4h \
  secret_id_ttl=720h

# Get the Role ID (like a username - stable, not secret)
vault read auth/approle/role/myapp-role/role-id
# role_id    db02de05-fa39-4855-059b-67221c5c2f63

# Generate a Secret ID (like a password - rotatable)
vault write -f auth/approle/role/myapp-role/secret-id
# secret_id    6a174c20-f6de-a53c-74d2-6018fcceff64

# Login with AppRole (returns a token)
vault write auth/approle/login \
  role_id="db02de05-fa39-4855-059b-67221c5c2f63" \
  secret_id="6a174c20-f6de-a53c-74d2-6018fcceff64"
# auth.client_token    hvs.CAESIJ...
```

### Using Vault in Applications

```python
# app.py - Python example using hvac library
# pip install hvac
import hvac
import os

# Initialize client
client = hvac.Client(
    url=os.environ.get("VAULT_ADDR", "http://127.0.0.1:8200"),
)

# Login with AppRole
client.auth.approle.login(
    role_id=os.environ["VAULT_ROLE_ID"],
    secret_id=os.environ["VAULT_SECRET_ID"],
)

# Read secrets
secret = client.secrets.kv.v2.read_secret_version(
    path="myapp/database",
    mount_point="secret",
)

db_config = secret["data"]["data"]
print(f"Connecting to {db_config['host']}:{db_config['port']}")
print(f"User: {db_config['username']}")
# Never print passwords in production!
```

### Vault Kubernetes Integration

```bash
# Enable Kubernetes auth method in Vault
vault auth enable kubernetes

# Configure Vault to talk to the Kubernetes API
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443"

# Create a role that binds a Kubernetes service account to a Vault policy
vault write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=default \
  policies=myapp-read \
  ttl=1h
```

```yaml
# kubernetes/vault-injector-example.yaml
# Using Vault Agent Sidecar Injector to automatically inject secrets into pods

apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: default

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        # These annotations tell the Vault Agent Injector to inject secrets
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp"
        vault.hashicorp.com/agent-inject-secret-db-creds: "secret/data/myapp/database"
        vault.hashicorp.com/agent-inject-template-db-creds: |
          {{- with secret "secret/data/myapp/database" -}}
          export DB_HOST={{ .Data.data.host }}
          export DB_PORT={{ .Data.data.port }}
          export DB_USER={{ .Data.data.username }}
          export DB_PASS={{ .Data.data.password }}
          {{- end }}
    spec:
      serviceAccountName: myapp-sa
      containers:
        - name: myapp
          image: myapp:latest
          command: ["/bin/sh", "-c", "source /vault/secrets/db-creds && python app.py"]
```

---

## SAST and DAST Tools

### SAST (Static Application Security Testing)

SAST analyzes source code without executing it. It finds vulnerabilities early during development.

#### SonarQube Setup

```yaml
# ~/sonarqube/docker-compose.yml
version: "3.8"

services:
  sonarqube:
    image: sonarqube:community
    container_name: sonarqube
    ports:
      - "9000:9000"
    environment:
      - SONAR_JDBC_URL=jdbc:postgresql://sonar-db:5432/sonar
      - SONAR_JDBC_USERNAME=sonar
      - SONAR_JDBC_PASSWORD=sonar
    volumes:
      - sonar-data:/opt/sonarqube/data
      - sonar-extensions:/opt/sonarqube/extensions
    depends_on:
      - sonar-db
    networks:
      - sonar

  sonar-db:
    image: postgres:15
    container_name: sonar-db
    environment:
      - POSTGRES_USER=sonar
      - POSTGRES_PASSWORD=sonar
      - POSTGRES_DB=sonar
    volumes:
      - sonar-db-data:/var/lib/postgresql/data
    networks:
      - sonar

volumes:
  sonar-data:
  sonar-extensions:
  sonar-db-data:

networks:
  sonar:
    driver: bridge
```

```bash
# Start SonarQube
cd ~/sonarqube
docker compose up -d

# Wait for startup (takes 1-2 minutes)
# Access at http://localhost:9000 (admin / admin)

# Install SonarScanner CLI
# macOS
brew install sonar-scanner

# Scan a project
cd /path/to/your/project
sonar-scanner \
  -Dsonar.projectKey=my-project \
  -Dsonar.sources=. \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.token=YOUR_SONAR_TOKEN
```

#### Semgrep - Lightweight SAST

```bash
# Install Semgrep
pip install semgrep
# or
brew install semgrep

# Scan a project with default rules
semgrep scan --config auto .

# Scan with specific rulesets
semgrep scan --config p/python .
semgrep scan --config p/javascript .
semgrep scan --config p/docker .
semgrep scan --config p/owasp-top-ten .

# Output as JSON
semgrep scan --config auto --json --output results.json .
```

#### Semgrep in GitHub Actions

```yaml
# .github/workflows/sast.yml
name: SAST Scan

on:
  pull_request:
    branches: [main]

jobs:
  semgrep:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/python
            p/owasp-top-ten
            p/secrets
```

### DAST (Dynamic Application Security Testing)

DAST tests a running application from the outside, simulating real attacks.

#### OWASP ZAP (Zed Attack Proxy)

```bash
# Run ZAP as a Docker container for automated scanning
# Baseline scan - passive scan only (fast, safe)
docker run --rm -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
  -t http://host.docker.internal:8080

# Full scan - active scan (slower, tests for more vulnerabilities)
docker run --rm -t ghcr.io/zaproxy/zaproxy:stable zap-full-scan.py \
  -t http://host.docker.internal:8080

# API scan - scan an API definition
docker run --rm -t ghcr.io/zaproxy/zaproxy:stable zap-api-scan.py \
  -t http://host.docker.internal:8080/openapi.json \
  -f openapi
```

#### OWASP ZAP in GitHub Actions

```yaml
# .github/workflows/dast.yml
name: DAST Scan

on:
  push:
    branches: [main]

jobs:
  zap-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Start application
        run: |
          docker compose up -d
          sleep 10

      - name: ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.10.0
        with:
          target: "http://localhost:8080"
          rules_file_name: ".zap/rules.tsv"
          allow_issue_writing: false
```

---

## Kubernetes Network Policies

Network Policies control traffic flow between pods in a Kubernetes cluster. By default, all pods can communicate with all other pods. Network Policies let you restrict this.

**Important:** Network Policies require a CNI plugin that supports them (Calico, Cilium, Weave Net). The default kubenet does not enforce them.

```
┌──────────────────────────────────────────────────────────────┐
│              Network Policy Concepts                          │
│                                                               │
│  Without policies:       With policies:                       │
│                                                               │
│  ┌─────┐  ◄──►  ┌─────┐   ┌─────┐  ───►  ┌─────┐          │
│  │Pod A │        │Pod B │   │Pod A │        │Pod B │          │
│  └─────┘  ◄──►  └─────┘   └─────┘  ✗◄──  └─────┘          │
│     ▲                         ▲                               │
│     │     ◄──►  ┌─────┐      │     ✗◄──  ┌─────┐          │
│     └───────────│Pod C │      └──────✗───│Pod C │          │
│                 └─────┘                   └─────┘          │
│  All traffic allowed     Only explicit traffic allowed       │
└──────────────────────────────────────────────────────────────┘
```

### Default Deny All Traffic

The most secure starting point is to deny all traffic and then selectively allow what is needed.

```yaml
# network-policies/default-deny-all.yaml
# Deny all ingress AND egress traffic in the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}          # Applies to ALL pods in the namespace
  policyTypes:
    - Ingress
    - Egress
```

```bash
# Apply the default deny policy
kubectl apply -f network-policies/default-deny-all.yaml

# Now no pods in 'production' can send or receive traffic
# You must create allow policies for required communication
```

### Allow Frontend to Backend Communication

```yaml
# network-policies/allow-frontend-to-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend            # This policy applies to backend pods
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend   # Only allow traffic FROM frontend pods
      ports:
        - protocol: TCP
          port: 8080          # Only on port 8080
```

### Allow Backend to Database

```yaml
# network-policies/allow-backend-to-database.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 5432
```

### Allow DNS Resolution (Required When Egress Is Denied)

```yaml
# network-policies/allow-dns.yaml
# When you deny all egress, pods cannot resolve DNS names.
# This policy allows DNS traffic to kube-dns.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}             # All pods
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### Allow Ingress from External Load Balancer

```yaml
# network-policies/allow-external-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-to-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Ingress
  ingress:
    - from:
        # Allow from the ingress controller namespace
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 80
```

### Cross-Namespace Communication

```yaml
# network-policies/cross-namespace.yaml
# Allow monitoring namespace to scrape metrics from production pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-scrape
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              purpose: monitoring
      ports:
        - protocol: TCP
          port: 9090       # Prometheus metrics port
```

### Restrict Egress to Specific External IPs

```yaml
# network-policies/restrict-egress.yaml
# Only allow backend to access a specific external API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress-restricted
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
    # Allow traffic to database pods
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
    # Allow traffic to specific external IP
    - to:
        - ipBlock:
            cidr: 203.0.113.0/24   # External API IP range
      ports:
        - protocol: TCP
          port: 443
```

### Testing Network Policies

```bash
# Apply all policies
kubectl apply -f network-policies/

# Test connectivity from frontend to backend (should succeed)
kubectl exec -n production deploy/frontend -- curl -s -o /dev/null -w "%{http_code}" http://backend:8080/health
# 200

# Test connectivity from frontend to database (should fail/timeout)
kubectl exec -n production deploy/frontend -- curl -s --connect-timeout 3 http://database:5432
# curl: (28) Connection timed out

# Test DNS resolution (should work with the DNS allow policy)
kubectl exec -n production deploy/frontend -- nslookup backend
# Server: 10.96.0.10
# Name: backend.production.svc.cluster.local

# Describe a network policy to verify its rules
kubectl describe networkpolicy allow-frontend-to-backend -n production
```

---

## Container Image Signing

Image signing ensures that the images running in your cluster are authentic and have not been tampered with. Cosign (from the Sigstore project) is the standard tool.

### Install and Use Cosign

```bash
# Install cosign
# macOS
brew install cosign

# Ubuntu
wget https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
chmod +x cosign-linux-amd64
sudo mv cosign-linux-amd64 /usr/local/bin/cosign

# Generate a key pair
cosign generate-key-pair
# Creates cosign.key (private) and cosign.pub (public)

# Sign an image (image must be in a registry, not local)
cosign sign --key cosign.key docker.io/myuser/myapp:v1.0
# You will be prompted for the private key password

# Verify an image signature
cosign verify --key cosign.pub docker.io/myuser/myapp:v1.0
# Output shows the signature payload if valid

# Keyless signing with Sigstore (uses OIDC identity, no key management)
cosign sign docker.io/myuser/myapp:v1.0
# Opens browser for OIDC login, signs with ephemeral key

# Verify keyless signature
cosign verify \
  --certificate-identity=user@example.com \
  --certificate-oidc-issuer=https://accounts.google.com \
  docker.io/myuser/myapp:v1.0
```

### Image Signing in CI/CD

```yaml
# .github/workflows/build-sign.yml
name: Build and Sign

on:
  push:
    tags: ["v*"]

jobs:
  build-sign:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write       # Required for keyless signing

    steps:
      - uses: actions/checkout@v4

      - name: Install cosign
        uses: sigstore/cosign-installer@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push
        run: |
          docker build -t myuser/myapp:${{ github.ref_name }} .
          docker push myuser/myapp:${{ github.ref_name }}

      - name: Sign the image (keyless)
        run: |
          cosign sign myuser/myapp:${{ github.ref_name }}
```

---

## Policy-as-Code with OPA and Gatekeeper

Open Policy Agent (OPA) is a general-purpose policy engine. OPA Gatekeeper integrates OPA with Kubernetes to enforce policies on what resources can be created.

```
┌──────────────────────────────────────────────────────────────┐
│                  Gatekeeper Flow                              │
│                                                               │
│  kubectl apply ──► K8s API Server ──► Gatekeeper Webhook     │
│                                           │                   │
│                                    ┌──────┴───────┐          │
│                                    │  Evaluate     │          │
│                                    │  Constraints  │          │
│                                    │  against Rego │          │
│                                    │  Policies     │          │
│                                    └──────┬───────┘          │
│                                           │                   │
│                                    Allow / Deny               │
└──────────────────────────────────────────────────────────────┘
```

### Install Gatekeeper

```bash
# Install Gatekeeper in your Kubernetes cluster
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.15.0/deploy/gatekeeper.yaml

# Verify installation
kubectl get pods -n gatekeeper-system
# NAME                                           READY   STATUS
# gatekeeper-audit-...                           1/1     Running
# gatekeeper-controller-manager-...              1/1     Running
```

### Constraint Templates and Constraints

Gatekeeper uses two resources:
1. **ConstraintTemplate** -- defines the policy logic in Rego
2. **Constraint** -- applies the template with specific parameters

#### Policy: Require Labels on All Resources

```yaml
# gatekeeper/templates/required-labels-template.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Missing required labels: %v", [missing])
        }
```

```yaml
# gatekeeper/constraints/require-team-label.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
  parameters:
    labels:
      - "team"
      - "environment"
```

```bash
# Apply template and constraint
kubectl apply -f gatekeeper/templates/required-labels-template.yaml
kubectl apply -f gatekeeper/constraints/require-team-label.yaml

# Test - this should be DENIED (missing required labels)
kubectl create deployment nginx-test --image=nginx
# Error: Missing required labels: {"environment", "team"}

# This should succeed
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
  labels:
    team: backend
    environment: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
EOF
# deployment.apps/nginx-test created
```

#### Policy: Block Containers Running as Root

```yaml
# gatekeeper/templates/block-root-template.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sblockrootcontainer
spec:
  crd:
    spec:
      names:
        kind: K8sBlockRootContainer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sblockrootcontainer

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := sprintf("Container '%v' must set securityContext.runAsNonRoot to true", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.runAsUser == 0
          msg := sprintf("Container '%v' must not run as root (UID 0)", [container.name])
        }
```

```yaml
# gatekeeper/constraints/block-root.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sBlockRootContainer
metadata:
  name: block-root-containers
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet", "DaemonSet"]
    namespaces:
      - "production"
```

#### Policy: Only Allow Images from Trusted Registries

```yaml
# gatekeeper/templates/allowed-repos-template.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        openAPIV3Schema:
          type: object
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedrepos

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not image_from_allowed_repo(container.image)
          msg := sprintf("Container '%v' uses image '%v' which is not from an allowed repository. Allowed: %v", [container.name, container.image, input.parameters.repos])
        }

        image_from_allowed_repo(image) {
          repo := input.parameters.repos[_]
          startswith(image, repo)
        }
```

```yaml
# gatekeeper/constraints/allowed-repos.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: only-trusted-registries
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet", "DaemonSet"]
  parameters:
    repos:
      - "docker.io/mycompany/"
      - "gcr.io/my-project/"
      - "123456789.dkr.ecr.us-east-1.amazonaws.com/"
```

```bash
# Apply and test
kubectl apply -f gatekeeper/templates/allowed-repos-template.yaml
kubectl apply -f gatekeeper/constraints/allowed-repos.yaml

# This should be DENIED (untrusted registry)
kubectl run test --image=random-registry.io/suspicious:latest
# Error: Container 'test' uses image 'random-registry.io/suspicious:latest' which is not from an allowed repository

# Check audit results (violations in existing resources)
kubectl get k8sallowedrepos only-trusted-registries -o yaml
```

#### Policy: Require Resource Limits

```yaml
# gatekeeper/templates/require-limits-template.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequirelimits
spec:
  crd:
    spec:
      names:
        kind: K8sRequireLimits
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequirelimits

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("Container '%v' must specify cpu limits", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container '%v' must specify memory limits", [container.name])
        }
```

```yaml
# gatekeeper/constraints/require-limits.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequireLimits
metadata:
  name: require-resource-limits
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
    namespaces:
      - "production"
```

---

## Projects

### Project 1: Add Trivy Scanning to a CI/CD Pipeline

**Goal:** Build a CI/CD pipeline that automatically scans Docker images and code for vulnerabilities before deployment.

**Steps:**

```bash
# 1. Create a sample project
mkdir -p ~/project-trivy-cicd
cd ~/project-trivy-cicd

# 2. Create a deliberately vulnerable Dockerfile for testing
cat > Dockerfile << 'EOF'
FROM python:3.9-slim
RUN pip install flask==2.2.0 requests==2.28.0
COPY app.py /app.py
USER root
CMD ["python", "/app.py"]
EOF

# 3. Create a simple app
cat > app.py << 'PYEOF'
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello, DevSecOps!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
PYEOF

# 4. Build the image
docker build -t myapp:test .

# 5. Scan with Trivy - check for vulnerabilities
trivy image --severity CRITICAL,HIGH myapp:test

# 6. Scan Dockerfile for misconfigurations
trivy config ./Dockerfile
# Should warn about: running as root, no HEALTHCHECK, etc.

# 7. Fix the Dockerfile based on Trivy findings
cat > Dockerfile.secure << 'EOF'
FROM python:3.11-slim
RUN pip install --no-cache-dir flask==3.0.0 requests==2.31.0
RUN groupadd -r appuser && useradd -r -g appuser appuser
COPY app.py /app.py
USER appuser
HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:5000/ || exit 1
EXPOSE 5000
CMD ["python", "/app.py"]
EOF

# 8. Rebuild and rescan
docker build -t myapp:secure -f Dockerfile.secure .
trivy image --severity CRITICAL,HIGH myapp:secure
# Should have fewer or no critical vulnerabilities

# 9. Create a GitHub Actions workflow that runs Trivy on every PR
#    (see the GitHub Actions example in the Trivy CI/CD section above)
```

**Verification checklist:**
- [ ] Trivy scans the image and reports vulnerabilities
- [ ] Trivy config scan identifies Dockerfile misconfigurations
- [ ] Fixed Dockerfile passes Trivy config scan
- [ ] Fixed image has fewer critical vulnerabilities
- [ ] CI/CD pipeline fails on critical vulnerabilities

---

### Project 2: Set Up HashiCorp Vault for Secret Management

**Goal:** Run Vault, store application secrets, and access them from an application using AppRole authentication.

**Steps:**

```bash
# 1. Create project directory
mkdir -p ~/project-vault
cd ~/project-vault

# 2. Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: "3.8"
services:
  vault:
    image: hashicorp/vault:latest
    container_name: vault
    ports:
      - "8200:8200"
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: "dev-root-token"
      VAULT_DEV_LISTEN_ADDRESS: "0.0.0.0:8200"
    cap_add:
      - IPC_LOCK
    command: server -dev

  app:
    build: .
    container_name: secret-app
    environment:
      VAULT_ADDR: "http://vault:8200"
      VAULT_ROLE_ID: "${ROLE_ID}"
      VAULT_SECRET_ID: "${SECRET_ID}"
    depends_on:
      - vault
    networks:
      - default
EOF

# 3. Create the Python app that reads secrets from Vault
cat > app.py << 'PYEOF'
import hvac
import os
import time

def main():
    # Wait for Vault to be ready
    time.sleep(3)

    client = hvac.Client(url=os.environ["VAULT_ADDR"])

    # Login with AppRole
    role_id = os.environ.get("VAULT_ROLE_ID")
    secret_id = os.environ.get("VAULT_SECRET_ID")

    if role_id and secret_id:
        client.auth.approle.login(role_id=role_id, secret_id=secret_id)
        print("Authenticated with AppRole")
    else:
        client.token = "dev-root-token"
        print("Using dev root token")

    # Read the database secret
    secret = client.secrets.kv.v2.read_secret_version(
        path="myapp/database",
        mount_point="secret",
    )

    db = secret["data"]["data"]
    print(f"DB Host: {db['host']}")
    print(f"DB Port: {db['port']}")
    print(f"DB User: {db['username']}")
    print("DB Password: ******* (hidden)")
    print("Secrets retrieved successfully!")

if __name__ == "__main__":
    main()
PYEOF

# 4. Create Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim
RUN pip install hvac
COPY app.py /app.py
CMD ["python", "/app.py"]
EOF

# 5. Start Vault
docker compose up -d vault
sleep 3

# 6. Configure Vault
export VAULT_ADDR="http://127.0.0.1:8200"
export VAULT_TOKEN="dev-root-token"

# Store secrets
vault kv put secret/myapp/database \
  username="appuser" \
  password="SuperSecret123!" \
  host="db.internal" \
  port="5432"

# Create policy
vault policy write myapp-read - <<POLICY
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}
POLICY

# Enable AppRole and create a role
vault auth enable approle
vault write auth/approle/role/myapp-role \
  token_policies="myapp-read" \
  token_ttl=1h

# Get credentials
ROLE_ID=$(vault read -field=role_id auth/approle/role/myapp-role/role-id)
SECRET_ID=$(vault write -f -field=secret_id auth/approle/role/myapp-role/secret-id)

echo "Role ID: $ROLE_ID"
echo "Secret ID: $SECRET_ID"

# 7. Run the app with the AppRole credentials
export ROLE_ID SECRET_ID
docker compose up app
# Output should show:
# Authenticated with AppRole
# DB Host: db.internal
# DB Port: 5432
# DB User: appuser
# DB Password: ******* (hidden)
# Secrets retrieved successfully!
```

**Verification checklist:**
- [ ] Vault starts and is accessible at http://localhost:8200
- [ ] Secrets are stored in Vault KV store
- [ ] Policy restricts access to only myapp secrets
- [ ] AppRole authentication works
- [ ] Application retrieves secrets from Vault
- [ ] Application fails gracefully with wrong credentials

---

### Project 3: Write Kubernetes Network Policies for a Three-Tier Application

**Goal:** Deploy a frontend, backend, and database in Kubernetes with strict network policies that only allow necessary communication.

**Steps:**

```bash
# 1. Create the namespace
kubectl create namespace secure-app

# 2. Deploy the three-tier application
cat > three-tier-app.yaml << 'EOF'
# Frontend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: secure-app
  labels:
    app: frontend
    tier: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: secure-app
spec:
  selector:
    app: frontend
  ports:
    - port: 80
---
# Backend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: secure-app
  labels:
    app: backend
    tier: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        tier: backend
    spec:
      containers:
        - name: api
          image: hashicorp/http-echo:latest
          args:
            - "-text=Hello from Backend"
            - "-listen=:8080"
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: secure-app
spec:
  selector:
    app: backend
  ports:
    - port: 8080
---
# Database Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: secure-app
  labels:
    app: database
    tier: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
        tier: database
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              value: "testpassword"
---
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: secure-app
spec:
  selector:
    app: database
  ports:
    - port: 5432
EOF

kubectl apply -f three-tier-app.yaml

# 3. Apply network policies
cat > network-policies.yaml << 'EOF'
# Default deny all traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: secure-app
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# Allow DNS for all pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: secure-app
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
---
# Frontend: allow ingress from external, allow egress to backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: secure-app
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - ports:
        - protocol: TCP
          port: 80
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - protocol: TCP
          port: 8080
---
# Backend: allow ingress from frontend, allow egress to database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: secure-app
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: database
      ports:
        - protocol: TCP
          port: 5432
---
# Database: allow ingress ONLY from backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: secure-app
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - protocol: TCP
          port: 5432
EOF

kubectl apply -f network-policies.yaml

# 4. Test the policies
echo "--- Testing frontend -> backend (should SUCCEED) ---"
kubectl exec -n secure-app deploy/frontend -- \
  curl -s --connect-timeout 3 http://backend:8080
# Expected: Hello from Backend

echo "--- Testing frontend -> database (should FAIL) ---"
kubectl exec -n secure-app deploy/frontend -- \
  curl -s --connect-timeout 3 http://database:5432 2>&1 || echo "BLOCKED (expected)"
# Expected: Connection timed out

echo "--- Testing backend -> database (should SUCCEED) ---"
kubectl exec -n secure-app deploy/backend -- \
  curl -s --connect-timeout 3 http://database:5432 2>&1 || echo "Connection refused (expected - not HTTP, but network reachable)"

echo "--- Testing database -> backend (should FAIL) ---"
kubectl exec -n secure-app deploy/database -- \
  curl -s --connect-timeout 3 http://backend:8080 2>&1 || echo "BLOCKED (expected)"

# 5. Verify all policies
kubectl get networkpolicy -n secure-app
# NAME                   POD-SELECTOR     AGE
# default-deny-all       <none>           1m
# allow-dns              <none>           1m
# frontend-policy        tier=frontend    1m
# backend-policy         tier=backend     1m
# database-policy        tier=database    1m

# 6. Clean up
kubectl delete namespace secure-app
```

**Verification checklist:**
- [ ] All pods are running in the secure-app namespace
- [ ] Frontend can reach backend on port 8080
- [ ] Frontend cannot reach database on port 5432
- [ ] Backend can reach database on port 5432
- [ ] Database cannot reach backend or frontend
- [ ] DNS resolution works for all pods

---

## DevSecOps Cheat Sheet

| Tool | Purpose | Command |
|------|---------|---------|
| Trivy | Image scan | `trivy image myapp:latest` |
| Trivy | Filesystem scan | `trivy fs --scanners vuln,secret .` |
| Trivy | Config scan | `trivy config .` |
| Semgrep | SAST scan | `semgrep scan --config auto .` |
| OWASP ZAP | DAST scan | `docker run zaproxy/zaproxy zap-baseline.py -t URL` |
| Vault | Write secret | `vault kv put secret/path key=value` |
| Vault | Read secret | `vault kv get secret/path` |
| Cosign | Sign image | `cosign sign --key cosign.key IMAGE` |
| Cosign | Verify image | `cosign verify --key cosign.pub IMAGE` |
| Gatekeeper | Check violations | `kubectl get constraints` |
| NetworkPolicy | List policies | `kubectl get networkpolicy -n NAMESPACE` |
