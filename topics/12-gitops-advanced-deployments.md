# 12. GitOps & Advanced Deployment Strategies

## Table of Contents
- [GitOps Principles](#gitops-principles)
- [ArgoCD Installation & Setup](#argocd-installation--setup)
- [ArgoCD Application Management](#argocd-application-management)
- [FluxCD Overview](#fluxcd-overview)
- [Blue-Green Deployments](#blue-green-deployments)
- [Canary Deployments](#canary-deployments)
- [Rolling Updates](#rolling-updates)
- [Feature Flags](#feature-flags)
- [Multi-Environment Pipelines](#multi-environment-pipelines)
- [Projects](#projects)

---

## GitOps Principles

GitOps is an operational framework that applies DevOps best practices for infrastructure automation -- using Git as the single source of truth for declarative infrastructure and applications.

### The Four Principles of GitOps

```
1. DECLARATIVE
   - The entire system is described declaratively (YAML, JSON, HCL)
   - You define the DESIRED STATE, not the steps to get there
   - Example: "I want 3 replicas of nginx:1.25" not "scale up by 1"

2. VERSIONED & IMMUTABLE
   - All configuration is stored in Git
   - Every change creates a new commit (audit trail)
   - You can roll back to any previous state with git revert

3. PULLED AUTOMATICALLY
   - Agents (ArgoCD, FluxCD) run INSIDE the cluster
   - They PULL desired state from Git (no CI pushing to cluster)
   - No need to expose cluster credentials to CI systems

4. CONTINUOUSLY RECONCILED
   - Agents constantly compare desired state (Git) vs actual state (cluster)
   - Drift is detected and corrected automatically
   - Self-healing: manual kubectl changes are reverted
```

### GitOps vs Traditional CI/CD

```
Traditional CI/CD (Push Model):
  Developer -> Git -> CI Build -> CI Deploy (push) -> Cluster
  - CI system needs cluster credentials
  - No drift detection after deploy
  - kubectl apply from CI pipeline

GitOps (Pull Model):
  Developer -> Git <- ArgoCD/Flux (pull) -> Cluster
  - Only the agent needs cluster access
  - Continuous reconciliation
  - Git is the source of truth
```

### Recommended Git Repository Structure

```
# Option A: Monorepo (simpler for small teams)
my-project/
  app/                    # Application source code
    src/
    Dockerfile
  manifests/              # Kubernetes manifests
    base/
      deployment.yaml
      service.yaml
      kustomization.yaml
    overlays/
      dev/
        kustomization.yaml
      staging/
        kustomization.yaml
      production/
        kustomization.yaml

# Option B: Separate repos (recommended for larger teams)
my-app-code/              # App source + Dockerfile + CI pipeline
my-app-config/            # K8s manifests only -- ArgoCD watches this repo
  environments/
    dev/
    staging/
    production/
```

---

## ArgoCD Installation & Setup

ArgoCD is the most popular GitOps tool for Kubernetes. It watches Git repositories and ensures your cluster matches the desired state defined in Git.

### Install ArgoCD on Kubernetes

```bash
# Prerequisites: A running Kubernetes cluster (minikube, kind, EKS, etc.)
# Verify your cluster is running
kubectl cluster-info

# Create the argocd namespace
kubectl create namespace argocd

# Install ArgoCD using the official manifests
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Verify all pods are running (wait 1-2 minutes)
kubectl get pods -n argocd -w
# NAME                                  READY   STATUS    RESTARTS   AGE
# argocd-application-controller-0       1/1     Running   0          2m
# argocd-dex-server-xxx                 1/1     Running   0          2m
# argocd-notifications-controller-xxx   1/1     Running   0          2m
# argocd-redis-xxx                      1/1     Running   0          2m
# argocd-repo-server-xxx                1/1     Running   0          2m
# argocd-server-xxx                     1/1     Running   0          2m
```

### Install ArgoCD Using Helm (Alternative)

```bash
# Add the ArgoCD Helm repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install with default values
helm install argocd argo/argo-cd --namespace argocd --create-namespace

# Install with custom values
cat > argocd-values.yaml << 'EOF'
server:
  extraArgs:
    - --insecure     # Disable TLS on ArgoCD server (for dev/testing only)
  service:
    type: NodePort   # Use NodePort for local access
  ingress:
    enabled: true
    hostname: argocd.local
EOF

helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  -f argocd-values.yaml
```

### Access the ArgoCD Dashboard

```bash
# Option 1: Port-forward (simplest for local development)
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
# Access at: https://localhost:8080

# Option 2: Change service type to LoadBalancer
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Option 3: Change service type to NodePort (for minikube)
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
minikube service argocd-server -n argocd --url

# Get the initial admin password
# ArgoCD generates a random password stored in a Secret
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
# Output: something like "xYz123AbC456"

# Login: username = admin, password = (output from above)
```

### Install the ArgoCD CLI

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/

# Login via CLI
argocd login localhost:8080 --username admin --password <your-password> --insecure

# Change the admin password (recommended)
argocd account update-password \
  --account admin \
  --current-password <initial-password> \
  --new-password <new-password>
```

---

## ArgoCD Application Management

### Application CRD (Custom Resource Definition)

An ArgoCD Application is a Kubernetes custom resource that defines the source (Git repo) and destination (cluster/namespace) for your deployment.

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-web-app
  namespace: argocd            # ArgoCD apps always live in argocd namespace
  # Optional: Add finalizer to cascade delete resources when app is deleted
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  # The ArgoCD project this app belongs to
  project: default

  # SOURCE: Where to get the manifests
  source:
    repoURL: https://github.com/your-org/my-app-config.git
    targetRevision: main       # Branch, tag, or commit SHA
    path: environments/dev     # Path within the repo to the manifests

  # DESTINATION: Where to deploy
  destination:
    server: https://kubernetes.default.svc  # In-cluster
    namespace: my-app-dev

  # SYNC POLICY: How ArgoCD manages the application
  syncPolicy:
    automated:
      prune: true              # Delete resources removed from Git
      selfHeal: true           # Revert manual changes made to cluster
      allowEmpty: false        # Don't sync if source is empty
    syncOptions:
      - CreateNamespace=true   # Create namespace if it doesn't exist
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5                # Retry failed syncs up to 5 times
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

```bash
# Apply the Application CRD
kubectl apply -f argocd-app.yaml

# Check application status
argocd app get my-web-app

# Manually sync (if automated sync is not enabled)
argocd app sync my-web-app

# View sync history
argocd app history my-web-app

# Rollback to a previous sync
argocd app rollback my-web-app <history-id>
```

### Application with Helm Source

```yaml
# argocd-helm-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prometheus-stack
  namespace: argocd
spec:
  project: default
  source:
    # Helm chart from a Helm repository
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 55.5.0
    helm:
      releaseName: monitoring
      values: |
        grafana:
          enabled: true
          adminPassword: admin123
        prometheus:
          prometheusSpec:
            retention: 7d
            resources:
              requests:
                memory: 512Mi
              limits:
                memory: 1Gi
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

### Application with Kustomize Source

```yaml
# argocd-kustomize-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/my-app-config.git
    targetRevision: main
    path: overlays/staging
    kustomize:
      namePrefix: staging-
      commonLabels:
        env: staging
      images:
        - my-app=registry.example.com/my-app:v2.1.0
  destination:
    server: https://kubernetes.default.svc
    namespace: staging
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

### Sync Policies Explained

```yaml
# MANUAL SYNC: ArgoCD detects drift but does NOT auto-correct
syncPolicy: {}

# AUTOMATED SYNC: ArgoCD auto-syncs when Git changes are detected
syncPolicy:
  automated: {}

# AUTOMATED + PRUNE: Also removes resources deleted from Git
syncPolicy:
  automated:
    prune: true

# AUTOMATED + SELF-HEAL: Revert manual kubectl changes
syncPolicy:
  automated:
    selfHeal: true

# FULL AUTOMATION (recommended for production GitOps)
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

```bash
# Demonstrate self-healing:
# 1. Deploy an app with selfHeal enabled
# 2. Manually scale it
kubectl scale deployment my-app --replicas=10 -n my-app-dev

# 3. ArgoCD detects drift within seconds and reverts to Git-defined replicas
# Watch the events:
argocd app get my-web-app --refresh
# Status: Synced (ArgoCD reverted manual change)
```

### ArgoCD Projects

Projects provide logical grouping and access control for applications.

```yaml
# argocd-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-frontend
  namespace: argocd
spec:
  description: "Frontend team applications"

  # Only allow sources from these repos
  sourceRepos:
    - https://github.com/your-org/frontend-*

  # Only allow deploying to these clusters/namespaces
  destinations:
    - server: https://kubernetes.default.svc
      namespace: frontend-dev
    - server: https://kubernetes.default.svc
      namespace: frontend-staging
    - server: https://kubernetes.default.svc
      namespace: frontend-prod

  # Restrict which Kubernetes resources can be managed
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceWhitelist:
    - group: 'apps'
      kind: Deployment
    - group: ''
      kind: Service
    - group: 'networking.k8s.io'
      kind: Ingress
```

---

## FluxCD Overview

FluxCD is another popular GitOps tool. It runs as a set of controllers inside your cluster.

### Install FluxCD

```bash
# Install the Flux CLI
# macOS
brew install fluxcd/tap/flux

# Linux
curl -s https://fluxcd.io/install.sh | sudo bash

# Verify prerequisites
flux check --pre

# Bootstrap Flux with GitHub
# This creates the Flux components in your cluster and a Git repository
export GITHUB_TOKEN=<your-github-personal-access-token>

flux bootstrap github \
  --owner=your-github-username \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/my-cluster \
  --personal

# Verify installation
flux check
kubectl get pods -n flux-system
```

### FluxCD GitRepository and Kustomization

```yaml
# flux-git-source.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m               # Check for changes every minute
  url: https://github.com/your-org/my-app-config
  ref:
    branch: main
  secretRef:
    name: github-credentials   # Secret with Git credentials
---
# flux-kustomization.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 5m               # Reconcile every 5 minutes
  path: ./environments/dev
  prune: true                 # Remove resources deleted from Git
  sourceRef:
    kind: GitRepository
    name: my-app
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: my-app
      namespace: my-app-dev
  timeout: 2m
```

### ArgoCD vs FluxCD Comparison

```
Feature          | ArgoCD                    | FluxCD
-----------------|---------------------------|---------------------------
UI Dashboard     | Built-in web UI           | No built-in UI (use Weave)
Architecture     | Single controller         | Multiple micro-controllers
App Definition   | Application CRD           | GitRepository + Kustomization
Multi-cluster    | Built-in                  | Via Kustomization
Helm Support     | Native                    | Via HelmRelease CRD
Sync Strategy    | Pull with webhooks        | Pull with interval polling
RBAC             | Built-in project RBAC     | Kubernetes-native RBAC
Best For         | Teams wanting a UI        | Teams preferring CLI-first
```

---

## Blue-Green Deployments

Blue-green deployment runs two identical environments. "Blue" is the current production. "Green" is the new version. Once green is verified, traffic switches instantly from blue to green.

### How Blue-Green Works

```
Step 1: Blue is live, serving all traffic
  Users --> [Service] --> [Blue v1 Pods]
                          [Green v2 Pods] (idle, being tested)

Step 2: Switch traffic to Green
  Users --> [Service] --> [Green v2 Pods]
                          [Blue v1 Pods] (idle, kept for rollback)

Step 3: If green is healthy, decommission blue
  Users --> [Service] --> [Green v2 Pods]
                          [Blue v1 Pods] (terminated)
```

### Blue-Green with Kubernetes Services

```yaml
# blue-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
  labels:
    app: my-app
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: blue
  template:
    metadata:
      labels:
        app: my-app
        version: blue
    spec:
      containers:
        - name: my-app
          image: my-app:1.0.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
---
# green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-green
  labels:
    app: my-app
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: green
  template:
    metadata:
      labels:
        app: my-app
        version: green
    spec:
      containers:
        - name: my-app
          image: my-app:2.0.0       # New version
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
---
# service.yaml -- This selector controls which version gets traffic
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: ClusterIP
  selector:
    app: my-app
    version: blue            # <-- Change to "green" to switch traffic
  ports:
    - port: 80
      targetPort: 8080
```

### Blue-Green Switch Script

```bash
#!/bin/bash
# blue-green-switch.sh
# Usage: ./blue-green-switch.sh <blue|green>

set -euo pipefail

TARGET_VERSION=${1:?Usage: $0 <blue|green>}
SERVICE_NAME="my-app-service"
NAMESPACE="default"

echo "Current service selector:"
kubectl get svc $SERVICE_NAME -n $NAMESPACE -o jsonpath='{.spec.selector.version}'
echo ""

# Verify the target deployment is ready
echo "Checking if $TARGET_VERSION deployment is ready..."
kubectl rollout status deployment/my-app-$TARGET_VERSION -n $NAMESPACE --timeout=60s

READY_PODS=$(kubectl get pods -n $NAMESPACE \
  -l app=my-app,version=$TARGET_VERSION \
  -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}' | wc -w)

if [ "$READY_PODS" -lt 1 ]; then
  echo "ERROR: No ready pods for version $TARGET_VERSION"
  exit 1
fi

echo "$READY_PODS pods ready for version $TARGET_VERSION"

# Switch traffic
echo "Switching traffic to $TARGET_VERSION..."
kubectl patch svc $SERVICE_NAME -n $NAMESPACE \
  -p "{\"spec\":{\"selector\":{\"version\":\"$TARGET_VERSION\"}}}"

echo "Traffic now pointing to: $TARGET_VERSION"
echo "Verify: kubectl get svc $SERVICE_NAME -n $NAMESPACE -o jsonpath='{.spec.selector}'"
```

---

## Canary Deployments

Canary deployment gradually shifts traffic from the old version to the new version. If problems are detected, traffic is routed back to the old version.

### How Canary Works

```
Step 1: 100% traffic to stable
  Users --> 100% --> [Stable v1 Pods]
                     [Canary v2 Pods] (0%)

Step 2: Send 10% traffic to canary
  Users --> 90%  --> [Stable v1 Pods]
       --> 10%  --> [Canary v2 Pods]

Step 3: Monitor metrics, increase to 50%
  Users --> 50%  --> [Stable v1 Pods]
       --> 50%  --> [Canary v2 Pods]

Step 4: If healthy, promote canary to 100%
  Users --> 100% --> [Canary v2 Pods] (now stable)
                     [Old v1 Pods] (terminated)
```

### Canary with NGINX Ingress Weight-Based Routing

```yaml
# stable-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-stable
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      track: stable
  template:
    metadata:
      labels:
        app: my-app
        track: stable
    spec:
      containers:
        - name: my-app
          image: my-app:1.0.0
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-stable
spec:
  selector:
    app: my-app
    track: stable
  ports:
    - port: 80
      targetPort: 8080
---
# canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-canary
spec:
  replicas: 1               # Fewer replicas for canary
  selector:
    matchLabels:
      app: my-app
      track: canary
  template:
    metadata:
      labels:
        app: my-app
        track: canary
    spec:
      containers:
        - name: my-app
          image: my-app:2.0.0   # New version
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-canary
spec:
  selector:
    app: my-app
    track: canary
  ports:
    - port: 80
      targetPort: 8080
---
# Primary Ingress (stable)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: my-app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-stable
                port:
                  number: 80
---
# Canary Ingress (with weight annotation)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-canary-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"   # 10% traffic to canary
spec:
  ingressClassName: nginx
  rules:
    - host: my-app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-canary
                port:
                  number: 80
```

### Canary Progressive Rollout Script

```bash
#!/bin/bash
# canary-rollout.sh
# Gradually increases canary traffic weight

set -euo pipefail

CANARY_INGRESS="my-app-canary-ingress"
NAMESPACE="default"
WEIGHTS=(10 25 50 75 100)
PAUSE_SECONDS=60

for weight in "${WEIGHTS[@]}"; do
  echo "Setting canary weight to ${weight}%..."
  kubectl annotate ingress $CANARY_INGRESS \
    -n $NAMESPACE \
    nginx.ingress.kubernetes.io/canary-weight="$weight" \
    --overwrite

  echo "Canary weight is now ${weight}%. Monitoring for ${PAUSE_SECONDS}s..."

  # Check error rate (simplified example using curl)
  sleep $PAUSE_SECONDS

  # In production, you would check Prometheus metrics here:
  # ERROR_RATE=$(curl -s 'http://prometheus:9090/api/v1/query?query=...' | jq ...)
  # if [ "$ERROR_RATE" > "0.05" ]; then rollback; fi

  echo "Canary at ${weight}% looks healthy."
done

echo "Canary rollout complete. Promote canary to stable."
echo "Next steps:"
echo "  1. Update stable deployment image to match canary"
echo "  2. Delete canary deployment and ingress"
```

---

## Rolling Updates

Rolling updates are the default Kubernetes deployment strategy. Pods are replaced incrementally -- new pods are created before old ones are terminated.

### Rolling Update Configuration

```yaml
# rolling-update-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2             # Max pods ABOVE desired count during update
      maxUnavailable: 1       # Max pods BELOW desired count during update
      # With 6 replicas:
      #   maxSurge: 2       -> up to 8 pods during update (6 + 2)
      #   maxUnavailable: 1 -> at least 5 pods always available (6 - 1)
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:2.0.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
      # Graceful shutdown: give pods time to finish requests
      terminationGracePeriodSeconds: 30
  # Minimum time a pod must be ready before it is considered available
  minReadySeconds: 10
  # Number of old ReplicaSets to retain for rollback
  revisionHistoryLimit: 5
```

### Rolling Update Strategies Explained

```
maxSurge and maxUnavailable control the update pace:

Example: replicas=10

AGGRESSIVE (fast, uses more resources):
  maxSurge: 50%           # 5 extra pods (up to 15 total)
  maxUnavailable: 50%     # 5 pods can be down (5 minimum available)
  Result: Very fast update, needs ~50% extra resources

CONSERVATIVE (slow, minimal resource spike):
  maxSurge: 1             # 1 extra pod (up to 11 total)
  maxUnavailable: 0       # All 10 must stay available
  Result: Slow but safe, no downtime, needs 1 extra pod of resources

BALANCED (default-like):
  maxSurge: 25%           # 2-3 extra pods
  maxUnavailable: 25%     # 2-3 can be unavailable
  Result: Good balance of speed and safety

ZERO-DOWNTIME:
  maxSurge: 1
  maxUnavailable: 0
  Result: One new pod must become Ready before one old pod is terminated
```

```bash
# Trigger a rolling update by changing the image
kubectl set image deployment/my-app my-app=my-app:2.0.0

# Watch the rollout progress
kubectl rollout status deployment/my-app
# Waiting for deployment "my-app" rollout to finish:
#   3 out of 6 new replicas have been updated...
#   4 out of 6 new replicas have been updated...
#   6 out of 6 new replicas have been updated...
#   deployment "my-app" successfully rolled out

# View rollout history
kubectl rollout history deployment/my-app
# REVISION  CHANGE-CAUSE
# 1         Initial deploy
# 2         kubectl set image deployment/my-app my-app=my-app:2.0.0

# Rollback to previous version
kubectl rollout undo deployment/my-app

# Rollback to a specific revision
kubectl rollout undo deployment/my-app --to-revision=1

# Pause and resume a rollout (for manual verification)
kubectl rollout pause deployment/my-app
# ... verify partial rollout ...
kubectl rollout resume deployment/my-app
```

---

## Feature Flags

Feature flags allow you to enable or disable features at runtime without redeploying code. They decouple deployment from release.

### Feature Flag Concepts

```
Without Feature Flags:
  Deploy v2 with new feature --> ALL users see the feature immediately

With Feature Flags:
  Deploy v2 with new feature (flag OFF) --> No users see it
  Enable flag for 5% of users            --> 5% see it (canary test)
  Enable flag for internal team           --> Team verifies in production
  Enable flag for 100%                    --> Full rollout
  Disable flag if issues found            --> Instant rollback (no redeploy)
```

### Simple Feature Flag with ConfigMap

```yaml
# feature-flags-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
data:
  NEW_CHECKOUT_FLOW: "false"
  DARK_MODE: "true"
  SEARCH_V2: "false"
---
# deployment using feature flags
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:2.0.0
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: feature-flags
```

### Feature Flag in Application Code (Node.js Example)

```javascript
// feature-flags.js
// Simple feature flag implementation reading from environment variables

class FeatureFlags {
  static isEnabled(flagName) {
    const value = process.env[flagName];
    return value === 'true' || value === '1';
  }
}

// Usage in your Express routes:
const express = require('express');
const app = express();

app.get('/checkout', (req, res) => {
  if (FeatureFlags.isEnabled('NEW_CHECKOUT_FLOW')) {
    // New checkout experience
    return res.render('checkout-v2');
  }
  // Original checkout experience
  return res.render('checkout-v1');
});

app.listen(8080);
```

### Popular Feature Flag Tools

```
Tool              | Type         | Notes
------------------|--------------|---------------------------------------
LaunchDarkly      | SaaS         | Enterprise-grade, SDK for many languages
Unleash           | Open Source   | Self-hosted, REST API
Flagsmith         | Open Source   | Self-hosted or cloud, feature segments
ConfigCat         | SaaS         | Simple pricing, good for small teams
K8s ConfigMaps    | DIY          | Simple on/off flags, requires pod restart
```

---

## Multi-Environment Pipelines

A multi-environment pipeline promotes changes through stages: dev, staging, and production.

### Directory Structure for Multi-Environment

```
my-app-config/
  base/
    deployment.yaml
    service.yaml
    kustomization.yaml
  overlays/
    dev/
      kustomization.yaml        # Low resources, debug enabled
      patches/
        deployment-patch.yaml
    staging/
      kustomization.yaml        # Medium resources, mirrors prod config
      patches/
        deployment-patch.yaml
    production/
      kustomization.yaml        # High resources, strict security
      patches/
        deployment-patch.yaml
```

### Base Kustomization

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

commonLabels:
  app: my-web-app

# base/deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-web-app
  template:
    metadata:
      labels:
        app: my-web-app
    spec:
      containers:
        - name: app
          image: my-web-app:latest
          ports:
            - containerPort: 8080
          env:
            - name: LOG_LEVEL
              value: "info"

# base/service.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: my-web-app
spec:
  selector:
    app: my-web-app
  ports:
    - port: 80
      targetPort: 8080
```

### Environment Overlays

```yaml
# overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: dev-
namespace: dev

patches:
  - path: patches/deployment-patch.yaml

images:
  - name: my-web-app
    newTag: dev-latest

# overlays/dev/patches/deployment-patch.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web-app
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: app
          env:
            - name: LOG_LEVEL
              value: "debug"
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 100m
              memory: 128Mi
```

```yaml
# overlays/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: staging-
namespace: staging

patches:
  - path: patches/deployment-patch.yaml

images:
  - name: my-web-app
    newTag: v1.2.3

# overlays/staging/patches/deployment-patch.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web-app
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: app
          env:
            - name: LOG_LEVEL
              value: "info"
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: prod-
namespace: production

patches:
  - path: patches/deployment-patch.yaml

images:
  - name: my-web-app
    newTag: v1.2.3

# overlays/production/patches/deployment-patch.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web-app
spec:
  replicas: 5
  template:
    spec:
      containers:
        - name: app
          env:
            - name: LOG_LEVEL
              value: "warn"
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
      # Production: add pod anti-affinity for high availability
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - my-web-app
                topologyKey: kubernetes.io/hostname
```

### ArgoCD ApplicationSet for Multiple Environments

```yaml
# appset-multi-env.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-environments
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: dev
            namespace: dev
            autoSync: "true"
          - env: staging
            namespace: staging
            autoSync: "true"
          - env: production
            namespace: production
            autoSync: "false"       # Manual sync for production
  template:
    metadata:
      name: 'my-app-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/my-app-config.git
        targetRevision: main
        path: 'overlays/{{env}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
```

### CI/CD Pipeline Promoting Through Environments

```yaml
# .github/workflows/promote.yaml
name: Promote Through Environments

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.tag.outputs.tag }}
    steps:
      - uses: actions/checkout@v4

      - name: Set image tag
        id: tag
        run: echo "tag=v1.0.${{ github.run_number }}" >> "$GITHUB_OUTPUT"

      - name: Build and push Docker image
        run: |
          docker build -t registry.example.com/my-app:${{ steps.tag.outputs.tag }} .
          docker push registry.example.com/my-app:${{ steps.tag.outputs.tag }}

  deploy-dev:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: your-org/my-app-config
          token: ${{ secrets.CONFIG_REPO_TOKEN }}

      - name: Update dev image tag
        run: |
          cd overlays/dev
          kustomize edit set image my-web-app=registry.example.com/my-app:${{ needs.build.outputs.image_tag }}

      - name: Commit and push
        run: |
          git config user.name "CI Bot"
          git config user.email "ci@example.com"
          git add .
          git commit -m "deploy: update dev to ${{ needs.build.outputs.image_tag }}"
          git push

  deploy-staging:
    needs: [build, deploy-dev]
    runs-on: ubuntu-latest
    environment: staging          # Requires GitHub environment approval
    steps:
      - uses: actions/checkout@v4
        with:
          repository: your-org/my-app-config
          token: ${{ secrets.CONFIG_REPO_TOKEN }}

      - name: Update staging image tag
        run: |
          cd overlays/staging
          kustomize edit set image my-web-app=registry.example.com/my-app:${{ needs.build.outputs.image_tag }}

      - name: Commit and push
        run: |
          git config user.name "CI Bot"
          git config user.email "ci@example.com"
          git add .
          git commit -m "deploy: update staging to ${{ needs.build.outputs.image_tag }}"
          git push

  deploy-production:
    needs: [build, deploy-staging]
    runs-on: ubuntu-latest
    environment: production       # Requires manual approval
    steps:
      - uses: actions/checkout@v4
        with:
          repository: your-org/my-app-config
          token: ${{ secrets.CONFIG_REPO_TOKEN }}

      - name: Update production image tag
        run: |
          cd overlays/production
          kustomize edit set image my-web-app=registry.example.com/my-app:${{ needs.build.outputs.image_tag }}

      - name: Commit and push
        run: |
          git config user.name "CI Bot"
          git config user.email "ci@example.com"
          git add .
          git commit -m "deploy: update production to ${{ needs.build.outputs.image_tag }}"
          git push
```

---

## Projects

### Project 1: Install ArgoCD and Set Up GitOps Workflow

```
Goal: Install ArgoCD, connect a Git repository, and deploy an application
      using GitOps principles.

Steps:
  1. Start a local Kubernetes cluster (minikube or kind)
  2. Install ArgoCD using kubectl or Helm
  3. Access the ArgoCD dashboard
  4. Create a public GitHub repo with a simple nginx deployment YAML
  5. Create an ArgoCD Application CRD pointing to your repo
  6. Enable automated sync with self-healing
  7. Make a change in Git (update the image tag) and verify ArgoCD syncs
  8. Make a manual kubectl change and verify ArgoCD reverts it (self-heal)

Deliverables:
  - Screenshot of ArgoCD dashboard showing a synced application
  - Git repository with Kubernetes manifests
  - ArgoCD Application YAML file
  - Written explanation of what happened during self-healing test
```

### Project 2: Implement Blue-Green Deployment

```
Goal: Deploy two versions of an application and switch traffic between them
      with zero downtime.

Steps:
  1. Create a "blue" deployment running nginx:1.24
  2. Create a "green" deployment running nginx:1.25
  3. Create a Service pointing to the blue deployment
  4. Verify traffic goes to blue pods
  5. Write the blue-green switch script (see examples above)
  6. Switch traffic to green using the script
  7. Verify traffic now goes to green pods
  8. Practice rolling back to blue

Verification:
  - Use "kubectl exec" to curl the service from within a test pod
  - Check which pod version responds
  - Measure the switch time (should be <1 second)
```

### Project 3: Create a Canary Deployment with Progressive Rollout

```
Goal: Set up a canary deployment with NGINX Ingress weight-based routing
      and progressively roll out a new version.

Steps:
  1. Install NGINX Ingress controller:
     minikube addons enable ingress
  2. Deploy the stable version (v1) with 3 replicas
  3. Create the stable Ingress pointing to the stable service
  4. Deploy the canary version (v2) with 1 replica
  5. Create the canary Ingress with canary-weight: "10"
  6. Send 100 requests and verify ~10% go to canary
  7. Increase weight to 50%, verify traffic split
  8. Promote canary to 100%
  9. Clean up: update stable to v2, remove canary resources

Verification Script:
  #!/bin/bash
  # Count responses from each version
  V1_COUNT=0
  V2_COUNT=0
  for i in $(seq 1 100); do
    RESPONSE=$(curl -s http://my-app.example.com/version)
    if echo "$RESPONSE" | grep -q "v1"; then
      V1_COUNT=$((V1_COUNT + 1))
    else
      V2_COUNT=$((V2_COUNT + 1))
    fi
  done
  echo "v1: $V1_COUNT requests, v2: $V2_COUNT requests"
```

---

## Summary

| Strategy       | Downtime | Rollback Speed | Resource Cost | Risk Level |
|---------------|----------|----------------|---------------|------------|
| Rolling Update | None     | Slow (rollout) | Low (+25%)    | Medium     |
| Blue-Green     | None     | Instant        | High (2x)     | Low        |
| Canary         | None     | Fast           | Low (+10%)    | Lowest     |
| Recreate       | Yes      | Slow (redeploy)| None          | Highest    |

**Key Takeaways:**
- GitOps makes Git the single source of truth for your infrastructure
- ArgoCD and FluxCD are the leading GitOps tools for Kubernetes
- Choose your deployment strategy based on risk tolerance, resources, and speed requirements
- Feature flags let you separate deployment from release
- Multi-environment pipelines with Kustomize overlays keep configurations DRY
