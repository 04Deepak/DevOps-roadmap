# 9. Container Orchestration with Kubernetes

## Table of Contents
- [Kubernetes Setup](#kubernetes-setup)
- [Architecture](#architecture)
- [Core Objects](#core-objects)
- [kubectl Commands](#kubectl-commands)
- [Deployments & Services](#deployments--services)
- [ConfigMaps & Secrets](#configmaps--secrets)
- [Persistent Storage](#persistent-storage)
- [Ingress](#ingress)
- [Helm](#helm)
- [Auto-Scaling (HPA)](#auto-scaling-hpa)
- [RBAC](#rbac)
- [Managed Kubernetes (EKS)](#managed-kubernetes-eks)
- [Projects](#projects)

---

## Kubernetes Setup

### Minikube (Local Development)

```bash
# Install Minikube
# macOS
brew install minikube

# Ubuntu
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Install kubectl
# macOS
brew install kubectl

# Ubuntu
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/kubectl

# Start Minikube
minikube start --cpus=2 --memory=4096 --driver=docker

# Verify
kubectl cluster-info
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   1m    v1.28.3

# Enable addons
minikube addons enable ingress
minikube addons enable metrics-server
minikube addons enable dashboard

# Open dashboard
minikube dashboard
```

### Kind (Kubernetes in Docker)

```bash
# Install Kind
brew install kind        # macOS
go install sigs.k8s.io/kind@latest  # Go

# Create a cluster
kind create cluster --name devops-lab

# Multi-node cluster
cat > kind-config.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF
kind create cluster --config kind-config.yaml --name multi-node

# Delete cluster
kind delete cluster --name devops-lab
```

---

## Architecture

```
Kubernetes Cluster Architecture:

┌──────────────────────────────────────────────────────────────────┐
│                        CONTROL PLANE (Master)                     │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐   │
│  │  API Server   │  │  Scheduler   │  │ Controller Manager   │   │
│  │  (kube-api)   │  │              │  │ - Node Controller    │   │
│  │              │  │ Decides which │  │ - Replica Controller │   │
│  │ All requests  │  │ node runs    │  │ - Endpoint Controller│   │
│  │ go through    │  │ which pod    │  │                      │   │
│  │ here          │  │              │  │ Watches desired vs   │   │
│  └──────────────┘  └──────────────┘  │ actual state         │   │
│                                       └─────────────────────┘   │
│  ┌──────────────┐                                                │
│  │    etcd       │  Key-value store, holds ALL cluster data      │
│  └──────────────┘                                                │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                         WORKER NODE 1                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   kubelet     │  │  kube-proxy  │  │  Container   │          │
│  │              │  │              │  │  Runtime      │          │
│  │ Agent on each│  │ Network proxy│  │  (containerd) │          │
│  │ node, talks  │  │ handles      │  │              │          │
│  │ to API server│  │ service IPs  │  │ Runs          │          │
│  └──────────────┘  └──────────────┘  │ containers   │          │
│                                       └──────────────┘          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                         │
│  │  Pod A  │  │  Pod B  │  │  Pod C  │                         │
│  │┌───────┐│  │┌───────┐│  │┌───────┐│                         │
│  ││ nginx ││  ││ api   ││  ││ worker││                         │
│  │└───────┘│  │└───────┘│  │└───────┘│                         │
│  └─────────┘  └─────────┘  └─────────┘                         │
└──────────────────────────────────────────────────────────────────┘
```

---

## Core Objects

### Pods

```yaml
# File: pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```

```bash
# Create a pod
kubectl apply -f pod.yaml

# Quick run (imperative)
kubectl run nginx --image=nginx:1.25 --port=80

# Get pods
kubectl get pods
kubectl get pods -o wide               # More details
kubectl get pods --show-labels          # Show labels
kubectl get pods -l app=nginx           # Filter by label

# Describe pod (detailed info)
kubectl describe pod nginx-pod

# Pod logs
kubectl logs nginx-pod
kubectl logs nginx-pod -f               # Follow logs
kubectl logs nginx-pod -c nginx         # Specific container

# Execute command inside pod
kubectl exec -it nginx-pod -- bash
kubectl exec nginx-pod -- cat /etc/nginx/nginx.conf

# Port forward (access from localhost)
kubectl port-forward nginx-pod 8080:80
# Now visit http://localhost:8080

# Delete pod
kubectl delete pod nginx-pod
kubectl delete -f pod.yaml
```

### Deployments

```yaml
# File: deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "250m"
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
```

```bash
# Apply deployment
kubectl apply -f deployment.yaml

# Get deployments
kubectl get deployments
kubectl get deploy         # Short form

# Scale deployment
kubectl scale deployment web-app --replicas=5

# Update image (rolling update)
kubectl set image deployment/web-app web=nginx:1.26

# Check rollout status
kubectl rollout status deployment/web-app

# View rollout history
kubectl rollout history deployment/web-app

# Rollback to previous version
kubectl rollout undo deployment/web-app

# Rollback to specific revision
kubectl rollout undo deployment/web-app --to-revision=2
```

### Services

```yaml
# File: service.yaml

# ClusterIP - Internal only (default)
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80

---
# NodePort - Expose on each node's IP
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080    # Range: 30000-32767

---
# LoadBalancer - Cloud provider load balancer
apiVersion: v1
kind: Service
metadata:
  name: web-lb
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
```

```bash
# Create service
kubectl apply -f service.yaml

# Get services
kubectl get services
kubectl get svc

# Access NodePort (Minikube)
minikube service web-nodeport --url

# Test ClusterIP from inside cluster
kubectl run curl-test --image=curlimages/curl --rm -it -- curl http://web-service
```

### Namespaces

```bash
# List namespaces
kubectl get namespaces

# Create namespace
kubectl create namespace staging
kubectl create namespace production

# Deploy to a specific namespace
kubectl apply -f deployment.yaml -n staging

# Get resources in a namespace
kubectl get all -n staging

# Set default namespace
kubectl config set-context --current --namespace=staging
```

---

## ConfigMaps & Secrets

### ConfigMaps

```yaml
# File: configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  APP_PORT: "3000"
  LOG_LEVEL: "info"
  config.json: |
    {
      "database": {
        "host": "db-service",
        "port": 5432,
        "name": "myapp"
      }
    }
```

```yaml
# Using ConfigMap in a Pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: myapp:1.0
          # Environment variables from ConfigMap
          envFrom:
            - configMapRef:
                name: app-config
          # Or individual keys
          env:
            - name: DATABASE_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: APP_ENV
          # Mount as file
          volumeMounts:
            - name: config-volume
              mountPath: /app/config
      volumes:
        - name: config-volume
          configMap:
            name: app-config
```

### Secrets

```bash
# Create secret (imperative)
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=SuperSecret123

# Create from file
kubectl create secret generic tls-cert \
  --from-file=cert.pem \
  --from-file=key.pem
```

```yaml
# File: secret.yaml (base64 encoded)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=          # echo -n "admin" | base64
  password: U3VwZXJTZWNyZXQ=  # echo -n "SuperSecret" | base64
```

```yaml
# Using Secrets in a Pod
spec:
  containers:
    - name: app
      image: myapp:1.0
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

---

## Persistent Storage

```yaml
# File: pv-pvc.yaml

# PersistentVolume (admin creates this)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: db-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/db

---
# PersistentVolumeClaim (developer requests storage)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi

---
# Using PVC in a Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
          volumeMounts:
            - name: db-storage
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: db-storage
          persistentVolumeClaim:
            claimName: db-pvc
```

---

## Ingress

```yaml
# File: ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 3000

    - host: admin.myapp.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 8080
```

```bash
# Enable Ingress controller (Minikube)
minikube addons enable ingress

# Apply ingress
kubectl apply -f ingress.yaml

# Add to /etc/hosts
echo "$(minikube ip) myapp.local admin.myapp.local" | sudo tee -a /etc/hosts

# Test
curl http://myapp.local
curl http://myapp.local/api
curl http://admin.myapp.local
```

---

## Helm

```bash
# Install Helm
brew install helm          # macOS
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash  # Linux

# Add popular repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Search for charts
helm search repo nginx
helm search repo postgresql

# Install a chart
helm install my-nginx bitnami/nginx
helm install my-pg bitnami/postgresql --set auth.postgresPassword=secret123

# List releases
helm list

# Get release info
helm status my-nginx

# Uninstall
helm uninstall my-nginx
```

### Create Your Own Helm Chart

```bash
# Create chart
helm create myapp

# Structure:
# myapp/
# ├── Chart.yaml           # Chart metadata
# ├── values.yaml           # Default values
# ├── templates/
# │   ├── deployment.yaml   # Templated deployment
# │   ├── service.yaml      # Templated service
# │   ├── ingress.yaml      # Templated ingress
# │   └── _helpers.tpl      # Template helpers
# └── charts/               # Dependencies
```

```yaml
# File: myapp/values.yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  host: myapp.local

resources:
  requests:
    memory: "64Mi"
    cpu: "100m"
  limits:
    memory: "128Mi"
    cpu: "250m"
```

```yaml
# File: myapp/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "myapp.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "myapp.name" . }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

```bash
# Install your chart
helm install myapp ./myapp

# Install with custom values
helm install myapp ./myapp --set replicaCount=5 --set image.tag=1.26

# Install with values file
helm install myapp ./myapp -f production-values.yaml

# Upgrade
helm upgrade myapp ./myapp --set replicaCount=3

# Dry run (preview)
helm install myapp ./myapp --dry-run

# Package chart
helm package myapp
```

---

## Auto-Scaling (HPA)

```yaml
# File: hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

```bash
# Apply HPA
kubectl apply -f hpa.yaml

# Check HPA status
kubectl get hpa

# Generate load to test scaling
kubectl run load-gen --image=busybox --rm -it -- /bin/sh -c "while true; do wget -q -O- http://web-service; done"

# Watch pods scale
kubectl get pods -w
kubectl get hpa -w
```

---

## RBAC

```yaml
# File: rbac.yaml

# Role - namespace-scoped permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: staging
  name: developer-role
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]

---
# RoleBinding - binds role to a user
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: staging
subjects:
  - kind: User
    name: dev-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io

---
# ClusterRole - cluster-wide permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: readonly-role
rules:
  - apiGroups: [""]
    resources: ["*"]
    verbs: ["get", "list", "watch"]
```

---

## Projects

### Project 1: Deploy Multi-Container App

```yaml
# File: full-app.yaml
# Complete app: Frontend + API + Database

# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: devops-app

---
# Database Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: devops-app
type: Opaque
data:
  POSTGRES_PASSWORD: c2VjcmV0MTIz  # secret123

---
# Database ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
  namespace: devops-app
data:
  POSTGRES_DB: myapp
  POSTGRES_USER: admin

---
# Database PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: devops-app
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi

---
# Database Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: devops-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          ports:
            - containerPort: 5432
          envFrom:
            - configMapRef:
                name: db-config
            - secretRef:
                name: db-secret
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: postgres-pvc

---
# Database Service
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: devops-app
spec:
  selector:
    app: postgres
  ports:
    - port: 5432

---
# API Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: devops-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: nginx:alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 64Mi

---
# API Service
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: devops-app
spec:
  selector:
    app: api
  ports:
    - port: 80

---
# Frontend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: devops-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: nginx:alpine
          ports:
            - containerPort: 80

---
# Frontend Service (NodePort for external access)
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: devops-app
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - port: 80
      nodePort: 30080
```

```bash
# Deploy everything
kubectl apply -f full-app.yaml

# Check all resources
kubectl get all -n devops-app

# Access frontend
minikube service frontend -n devops-app --url

# Scale API
kubectl scale deployment api --replicas=5 -n devops-app

# Watch pods
kubectl get pods -n devops-app -w

# Clean up
kubectl delete namespace devops-app
```

---

## kubectl Cheat Sheet

| Command | Description |
|---------|-------------|
| `kubectl get pods` | List pods |
| `kubectl get all` | List all resources |
| `kubectl describe pod <name>` | Pod details |
| `kubectl logs <pod>` | View pod logs |
| `kubectl exec -it <pod> -- sh` | Shell into pod |
| `kubectl apply -f file.yaml` | Apply configuration |
| `kubectl delete -f file.yaml` | Delete resources |
| `kubectl scale deploy <name> --replicas=N` | Scale deployment |
| `kubectl rollout status deploy/<name>` | Check rollout |
| `kubectl rollout undo deploy/<name>` | Rollback |
| `kubectl port-forward <pod> 8080:80` | Forward port |
| `kubectl get events --sort-by=.lastTimestamp` | View events |
| `kubectl top pods` | Resource usage |
