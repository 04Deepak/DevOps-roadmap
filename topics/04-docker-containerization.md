# 4. Containerization with Docker

## Table of Contents
- [Docker Installation](#docker-installation)
- [Docker Concepts](#docker-concepts)
- [Docker Commands](#docker-commands)
- [Dockerfile](#dockerfile)
- [Multi-Stage Builds](#multi-stage-builds)
- [Docker Compose](#docker-compose)
- [Docker Networking](#docker-networking)
- [Docker Volumes](#docker-volumes)
- [Image Registries](#image-registries)
- [Projects](#projects)

---

## Docker Installation

### Ubuntu

```bash
# Remove old versions
sudo apt remove docker docker-engine docker.io containerd runc

# Install prerequisites
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up the repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add current user to docker group (avoid sudo)
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
# Output: Docker version 24.0.7

docker run hello-world
# Output: Hello from Docker! This message shows that your installation appears to be working correctly.
```

### macOS / Windows

```bash
# Download Docker Desktop from docker.com
# Install and start Docker Desktop

# Verify
docker --version
docker compose version
```

---

## Docker Concepts

```
┌──────────────────────────────────────────────────────────┐
│                    Docker Architecture                     │
│                                                            │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐   │
│  │  Container 1 │    │  Container 2 │    │  Container 3 │   │
│  │  (Nginx)     │    │  (Node.js)   │    │  (MongoDB)   │   │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘   │
│         │                  │                  │           │
│  ┌──────┴──────────────────┴──────────────────┴──────┐   │
│  │               Docker Engine (Daemon)                │   │
│  └────────────────────────┬───────────────────────────┘   │
│                           │                               │
│  ┌────────────────────────┴───────────────────────────┐   │
│  │                  Host OS (Linux)                     │   │
│  └────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘

Containers vs VMs:
┌──────────────────────┐     ┌──────────────────────┐
│     Containers        │     │   Virtual Machines    │
│  ┌────┐ ┌────┐ ┌────┐│     │  ┌────┐ ┌────┐ ┌────┐│
│  │App1│ │App2│ │App3││     │  │App1│ │App2│ │App3││
│  ├────┤ ├────┤ ├────┤│     │  ├────┤ ├────┤ ├────┤│
│  │Bins│ │Bins│ │Bins││     │  │Bins│ │Bins│ │Bins││
│  └──┬─┘ └──┬─┘ └──┬─┘│     │  ├────┤ ├────┤ ├────┤│
│  ┌──┴──────┴──────┴──┐│     │  │ OS │ │ OS │ │ OS ││
│  │   Docker Engine    ││     │  └──┬─┘ └──┬─┘ └──┬─┘│
│  ├────────────────────┤│     │  ┌──┴──────┴──────┴──┐│
│  │     Host OS        ││     │  │    Hypervisor      ││
│  └────────────────────┘│     │  ├────────────────────┤│
│  Lightweight, fast,    │     │  │     Host OS        ││
│  shares OS kernel      │     │  └────────────────────┘│
└──────────────────────┘     │  Heavy, each has full OS│
                              └──────────────────────┘
```

### Key Terms

| Term | Description |
|------|-------------|
| **Image** | Blueprint/template for a container (read-only) |
| **Container** | Running instance of an image |
| **Dockerfile** | Instructions to build an image |
| **Registry** | Storage for Docker images (Docker Hub, ECR) |
| **Volume** | Persistent data storage for containers |
| **Network** | Communication channel between containers |

---

## Docker Commands

### Images

```bash
# Pull an image from Docker Hub
docker pull nginx
docker pull nginx:1.25          # Specific version/tag
docker pull ubuntu:22.04

# List local images
docker images
# REPOSITORY   TAG       IMAGE ID       SIZE
# nginx        latest    a8758716bb6a   187MB
# ubuntu       22.04     3db8720ecbf5   77.8MB

# Remove an image
docker rmi nginx
docker rmi -f nginx             # Force remove

# Remove all unused images
docker image prune -a

# Inspect image details
docker inspect nginx
docker history nginx            # Show image layers
```

### Containers

```bash
# Run a container
docker run nginx                        # Foreground (Ctrl+C to stop)
docker run -d nginx                     # Detached (background)
docker run -d --name my-nginx nginx     # With a name
docker run -d -p 8080:80 nginx          # Map host:container port
docker run -d -p 8080:80 --name web nginx

# List containers
docker ps                               # Running containers
docker ps -a                            # All containers (including stopped)

# Container operations
docker stop my-nginx                    # Stop gracefully
docker start my-nginx                   # Start a stopped container
docker restart my-nginx                 # Restart
docker rm my-nginx                      # Remove (must be stopped)
docker rm -f my-nginx                   # Force remove (even if running)

# Interact with a running container
docker exec -it my-nginx bash           # Open shell inside container
docker exec my-nginx cat /etc/nginx/nginx.conf  # Run a command

# View container logs
docker logs my-nginx                    # All logs
docker logs -f my-nginx                 # Follow logs (real-time)
docker logs --tail 50 my-nginx          # Last 50 lines

# View container resource usage
docker stats                            # Live resource usage
docker top my-nginx                     # Running processes

# Copy files to/from container
docker cp index.html my-nginx:/usr/share/nginx/html/
docker cp my-nginx:/etc/nginx/nginx.conf ./nginx.conf

# Remove all stopped containers
docker container prune
```

### Full Example: Run Nginx

```bash
# Run Nginx with custom HTML
mkdir -p ~/docker-demo && cd ~/docker-demo

# Create a custom HTML page
cat > index.html << 'HTML'
<!DOCTYPE html>
<html>
<head><title>Docker Demo</title></head>
<body>
  <h1>Hello from Docker!</h1>
  <p>This page is served by Nginx running in a Docker container.</p>
</body>
</html>
HTML

# Run with volume mount (local file → container)
docker run -d \
  --name web-demo \
  -p 8080:80 \
  -v $(pwd)/index.html:/usr/share/nginx/html/index.html:ro \
  nginx

# Test
curl http://localhost:8080
# Output: <h1>Hello from Docker!</h1>

# Clean up
docker stop web-demo && docker rm web-demo
```

---

## Dockerfile

### Syntax & Best Practices

```dockerfile
# File: Dockerfile

# Base image
FROM ubuntu:22.04

# Metadata
LABEL maintainer="student@devops.com"
LABEL description="Sample DevOps application"

# Set environment variables
ENV APP_HOME=/app
ENV NODE_ENV=production

# Set working directory
WORKDIR $APP_HOME

# Install system dependencies
# Combine RUN commands to reduce layers
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        python3 \
        python3-pip && \
    rm -rf /var/lib/apt/lists/*

# Copy dependency files first (leverages cache)
COPY requirements.txt .

# Install application dependencies
RUN pip3 install --no-cache-dir -r requirements.txt

# Copy application code (this changes most often, so it's last)
COPY . .

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:5000/health || exit 1

# Default command
CMD ["python3", "app.py"]
```

### Python Flask Application Example

```bash
mkdir -p ~/flask-docker && cd ~/flask-docker
```

```python
# File: app.py
from flask import Flask, jsonify
import os
import socket

app = Flask(__name__)

@app.route("/")
def home():
    return jsonify({
        "message": "Hello from Docker!",
        "hostname": socket.gethostname(),
        "environment": os.environ.get("APP_ENV", "development")
    })

@app.route("/health")
def health():
    return jsonify({"status": "healthy"}), 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

```
# File: requirements.txt
flask==3.0.0
```

```dockerfile
# File: Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

ENV APP_ENV=production

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:5000/health || exit 1

CMD ["python", "app.py"]
```

```bash
# Build the image
docker build -t flask-app:1.0 .

# Run the container
docker run -d --name flask-demo -p 5000:5000 flask-app:1.0

# Test
curl http://localhost:5000
# {"environment":"production","hostname":"a1b2c3d4e5f6","message":"Hello from Docker!"}

curl http://localhost:5000/health
# {"status":"healthy"}

# View logs
docker logs flask-demo
```

### Node.js Application Example

```bash
mkdir -p ~/node-docker && cd ~/node-docker
```

```json
// File: package.json
{
  "name": "node-docker-app",
  "version": "1.0.0",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

```javascript
// File: server.js
const express = require('express');
const os = require('os');
const app = express();
const PORT = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Docker!',
    hostname: os.hostname(),
    platform: os.platform(),
    uptime: process.uptime()
  });
});

app.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

```dockerfile
# File: Dockerfile
FROM node:20-alpine

WORKDIR /app

# Copy dependency files first
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY server.js .

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "server.js"]
```

```bash
# Build and run
docker build -t node-app:1.0 .
docker run -d --name node-demo -p 3000:3000 node-app:1.0
curl http://localhost:3000
```

---

## Multi-Stage Builds

```dockerfile
# File: Dockerfile.multistage
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build    # e.g., compile TypeScript, bundle assets

# Stage 2: Production (only runtime, no build tools)
FROM node:20-alpine AS production
WORKDIR /app

# Copy only what's needed from builder
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

EXPOSE 3000
USER node
CMD ["node", "dist/server.js"]

# Result: Much smaller image! Build tools are NOT included
```

```bash
# Compare image sizes
docker build -t app:regular -f Dockerfile .
docker build -t app:multistage -f Dockerfile.multistage .

docker images | grep app
# app    regular      abc123    450MB
# app    multistage   def456    150MB  ← Much smaller!
```

---

## Docker Compose

### docker-compose.yml Basics

```yaml
# File: docker-compose.yml
# Full stack: Nginx + Node.js API + MongoDB

services:
  # Frontend / Reverse Proxy
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - api
    restart: unless-stopped

  # Backend API
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - MONGO_URI=mongodb://mongo:27017/myapp
      - PORT=3000
    depends_on:
      - mongo
    restart: unless-stopped

  # Database
  mongo:
    image: mongo:7
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=secret123
    restart: unless-stopped

volumes:
  mongo-data:
    driver: local
```

### Nginx Reverse Proxy Config

```nginx
# File: nginx/nginx.conf
upstream api {
    server api:3000;
}

server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    location /api/ {
        proxy_pass http://api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Docker Compose Commands

```bash
# Start all services (detached)
docker compose up -d

# Start and rebuild images
docker compose up -d --build

# View running services
docker compose ps

# View logs
docker compose logs           # All services
docker compose logs api       # Specific service
docker compose logs -f api    # Follow logs

# Stop all services
docker compose down

# Stop and remove volumes (deletes data!)
docker compose down -v

# Scale a service
docker compose up -d --scale api=3

# Execute command in service
docker compose exec api sh
docker compose exec mongo mongosh

# Restart a specific service
docker compose restart api
```

---

## Docker Networking

```bash
# List networks
docker network ls
# NETWORK ID   NAME      DRIVER    SCOPE
# abc123       bridge    bridge    local   ← default
# def456       host      host      local
# ghi789       none      null      local

# Create a custom network
docker network create my-app-network

# Run containers on the same network
docker run -d --name web --network my-app-network nginx
docker run -d --name api --network my-app-network node-app:1.0

# Containers can reach each other by name
docker exec web ping api    # Works! Docker DNS resolves "api"

# Inspect network
docker network inspect my-app-network

# Connect existing container to network
docker network connect my-app-network existing-container

# Disconnect
docker network disconnect my-app-network existing-container
```

### Network Types

```
bridge (default):  Containers on same host can communicate
                   Need port mapping (-p) for external access

host:              Container uses host's network directly
                   No port mapping needed, but no isolation

none:              No networking (fully isolated)

overlay:           Multi-host networking (Docker Swarm)
```

---

## Docker Volumes

```bash
# Create a named volume
docker volume create app-data

# List volumes
docker volume ls

# Use volume in container
docker run -d \
  --name db \
  -v app-data:/var/lib/mysql \
  mysql:8

# Bind mount (host directory → container)
docker run -d \
  --name web \
  -v $(pwd)/html:/usr/share/nginx/html \
  nginx

# Read-only mount
docker run -d \
  -v $(pwd)/config:/app/config:ro \
  my-app

# Inspect volume
docker volume inspect app-data

# Remove unused volumes
docker volume prune

# Backup a volume
docker run --rm \
  -v app-data:/data \
  -v $(pwd):/backup \
  ubuntu tar czf /backup/data-backup.tar.gz /data
```

---

## Image Registries

### Docker Hub

```bash
# Login to Docker Hub
docker login
# Enter username and password

# Tag image for Docker Hub
docker tag flask-app:1.0 yourusername/flask-app:1.0
docker tag flask-app:1.0 yourusername/flask-app:latest

# Push to Docker Hub
docker push yourusername/flask-app:1.0
docker push yourusername/flask-app:latest

# Pull from Docker Hub
docker pull yourusername/flask-app:1.0
```

### AWS ECR (Elastic Container Registry)

```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Create repository
aws ecr create-repository --repository-name flask-app

# Tag and push
docker tag flask-app:1.0 123456789012.dkr.ecr.us-east-1.amazonaws.com/flask-app:1.0
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/flask-app:1.0
```

---

## Projects

### Project 1: Containerize a Flask App

```bash
mkdir -p ~/project-flask-docker && cd ~/project-flask-docker
```

Full project structure:

```
project-flask-docker/
├── Dockerfile
├── .dockerignore
├── requirements.txt
├── app.py
└── templates/
    └── index.html
```

```python
# File: app.py
from flask import Flask, render_template, jsonify
import os
import socket
import datetime

app = Flask(__name__)

@app.route("/")
def home():
    return render_template("index.html",
        hostname=socket.gethostname(),
        time=datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S"))

@app.route("/api/info")
def info():
    return jsonify({
        "hostname": socket.gethostname(),
        "ip": socket.gethostbyname(socket.gethostname()),
        "environment": os.environ.get("APP_ENV", "development"),
        "version": os.environ.get("APP_VERSION", "1.0.0")
    })

@app.route("/health")
def health():
    return jsonify({"status": "healthy"}), 200

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 5000))
    app.run(host="0.0.0.0", port=port, debug=False)
```

```
# File: requirements.txt
flask==3.0.0
gunicorn==21.2.0
```

```html
<!-- File: templates/index.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Dockerized Flask App</title>
    <style>
        body { font-family: Arial; background: #0d1117; color: #c9d1d9; padding: 40px; }
        .container { max-width: 600px; margin: 0 auto; }
        h1 { color: #58a6ff; }
        .card { background: #161b22; padding: 20px; border-radius: 8px; border: 1px solid #30363d; }
        .label { color: #8b949e; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Dockerized Flask App</h1>
        <div class="card">
            <p><span class="label">Hostname:</span> {{ hostname }}</p>
            <p><span class="label">Server Time:</span> {{ time }}</p>
            <p><span class="label">Status:</span> Running in Docker</p>
        </div>
    </div>
</body>
</html>
```

```dockerfile
# File: Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV APP_ENV=production
ENV APP_VERSION=1.0.0
ENV PORT=5000

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:5000/health || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

```
# File: .dockerignore
__pycache__
*.pyc
.git
.gitignore
venv/
.env
*.md
```

```bash
# Build
docker build -t flask-app:1.0 .

# Run
docker run -d --name flask-web -p 5000:5000 flask-app:1.0

# Test
curl http://localhost:5000
curl http://localhost:5000/api/info
curl http://localhost:5000/health
```

### Project 2: Docker Compose Multi-Container App

```bash
mkdir -p ~/project-compose && cd ~/project-compose
mkdir -p api nginx
```

```javascript
// File: api/server.js
const express = require('express');
const { MongoClient } = require('mongodb');

const app = express();
app.use(express.json());

const MONGO_URI = process.env.MONGO_URI || 'mongodb://mongo:27017/devops';
let db;

MongoClient.connect(MONGO_URI).then(client => {
  db = client.db();
  console.log('Connected to MongoDB');
}).catch(err => console.error('MongoDB connection failed:', err));

app.get('/api/health', (req, res) => {
  res.json({ status: 'healthy', db: db ? 'connected' : 'disconnected' });
});

app.get('/api/visits', async (req, res) => {
  await db.collection('visits').insertOne({ timestamp: new Date() });
  const count = await db.collection('visits').countDocuments();
  res.json({ total_visits: count, message: `This page has been visited ${count} times` });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`API running on port ${PORT}`));
```

```json
// File: api/package.json
{
  "name": "devops-api",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.2",
    "mongodb": "^6.3.0"
  }
}
```

```dockerfile
# File: api/Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY server.js .
EXPOSE 3000
CMD ["node", "server.js"]
```

```nginx
# File: nginx/default.conf
upstream api_backend {
    server api:3000;
}

server {
    listen 80;
    server_name localhost;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    location /api/ {
        proxy_pass http://api_backend/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```yaml
# File: docker-compose.yml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - api
    restart: unless-stopped

  api:
    build: ./api
    environment:
      - MONGO_URI=mongodb://mongo:27017/devops
      - PORT=3000
    depends_on:
      - mongo
    restart: unless-stopped

  mongo:
    image: mongo:7
    volumes:
      - mongo-data:/data/db
    restart: unless-stopped

volumes:
  mongo-data:
```

```bash
# Start everything
docker compose up -d --build

# Check status
docker compose ps

# Test the API
curl http://localhost/api/health
curl http://localhost/api/visits
curl http://localhost/api/visits  # Count increases!

# View logs
docker compose logs -f

# Stop everything
docker compose down
```

### Project 3: Multi-Stage Build (Production-Ready)

```dockerfile
# File: Dockerfile.production

# ============ Stage 1: Dependencies ============
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && \
    cp -R node_modules /prod_modules && \
    npm ci  # All deps for building

# ============ Stage 2: Build ============
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build
RUN npm run test

# ============ Stage 3: Production ============
FROM node:20-alpine AS production
WORKDIR /app

# Security: run as non-root user
RUN addgroup -g 1001 appgroup && \
    adduser -u 1001 -G appgroup -D appuser

# Copy only production dependencies and built files
COPY --from=deps /prod_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./

# Set ownership
RUN chown -R appuser:appgroup /app

USER appuser

ENV NODE_ENV=production
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
```

```bash
# Build production image
docker build -t myapp:prod -f Dockerfile.production .

# Check size difference
docker images | grep myapp

# Scan for vulnerabilities
docker scout cves myapp:prod

# Run
docker run -d -p 3000:3000 --name prod-app myapp:prod
```

---

## Quick Reference

| Command | Description |
|---------|-------------|
| `docker build -t name:tag .` | Build image from Dockerfile |
| `docker run -d -p 8080:80 name` | Run container (detached, port mapped) |
| `docker ps` | List running containers |
| `docker logs -f container` | Follow container logs |
| `docker exec -it container sh` | Shell into container |
| `docker compose up -d` | Start all services |
| `docker compose down` | Stop and remove all services |
| `docker compose logs -f` | Follow all service logs |
| `docker volume ls` | List volumes |
| `docker network ls` | List networks |
| `docker image prune -a` | Remove unused images |
| `docker system prune -a` | Clean up everything |
