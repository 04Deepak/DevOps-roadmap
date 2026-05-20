# 6. CI/CD Tools

## Table of Contents
- [Jenkins](#jenkins)
- [GitHub Actions](#github-actions)
- [GitLab CI](#gitlab-ci)
- [Tool Comparison](#tool-comparison)
- [Projects](#projects)

---

## Jenkins

### Installation

```bash
# Method 1: Docker (Recommended for learning)
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts

# Get initial admin password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword

# Open: http://localhost:8080
# Paste the password → Install suggested plugins → Create admin user
```

```bash
# Method 2: Direct installation (Ubuntu)
# Add Jenkins repository
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Java and Jenkins
sudo apt update
sudo apt install -y fontconfig openjdk-17-jre jenkins

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Get password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Jenkins Pipeline (Declarative)

```groovy
// File: Jenkinsfile
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'myapp'
        DOCKER_TAG = "${BUILD_NUMBER}"
        REGISTRY = 'docker.io/yourusername'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "Building branch: ${env.BRANCH_NAME}"
                echo "Build number: ${env.BUILD_NUMBER}"
            }
        }

        stage('Build') {
            steps {
                sh 'npm ci'
                sh 'npm run lint'
                sh 'npm run build'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test -- --coverage'
            }
            post {
                always {
                    junit 'test-results/*.xml'
                    publishHTML([
                        reportDir: 'coverage',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG} ."
                sh "docker tag ${REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG} ${REGISTRY}/${DOCKER_IMAGE}:latest"
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh "docker push ${REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG}"
                    sh "docker push ${REGISTRY}/${DOCKER_IMAGE}:latest"
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    ssh deploy@staging-server \
                    "docker pull ${REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG} && \
                     docker stop myapp || true && \
                     docker rm myapp || true && \
                     docker run -d --name myapp -p 3000:3000 \
                       ${REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG}"
                '''
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message 'Deploy to production?'
                ok 'Deploy'
            }
            steps {
                sh '''
                    ssh deploy@production-server \
                    "docker pull ${REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG} && \
                     docker stop myapp || true && \
                     docker rm myapp || true && \
                     docker run -d --name myapp -p 3000:3000 \
                       ${REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG}"
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline succeeded!'
            // slackSend channel: '#deployments', message: "Build #${BUILD_NUMBER} succeeded"
        }
        failure {
            echo 'Pipeline failed!'
            // slackSend channel: '#deployments', message: "Build #${BUILD_NUMBER} FAILED", color: 'danger'
        }
        always {
            cleanWs()
        }
    }
}
```

### Jenkins Parallel Stages

```groovy
pipeline {
    agent any

    stages {
        stage('Build & Test in Parallel') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm run test:unit'
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'npm run test:integration'
                    }
                }
                stage('Lint') {
                    steps {
                        sh 'npm run lint'
                    }
                }
            }
        }
    }
}
```

---

## GitHub Actions

### Basic Workflow

```yaml
# File: .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '20'

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests
        run: npm test -- --coverage

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/
```

### Complete CI/CD Pipeline

```yaml
# File: .github/workflows/cicd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ============ Job 1: Build & Test ============
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Unit tests
        run: npm test -- --coverage --forceExit

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: |
            coverage/
            test-results/

  # ============ Job 2: Build Docker Image ============
  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ============ Job 3: Security Scan ============
  security:
    needs: build
    runs-on: ubuntu-latest
    if: github.event_name != 'pull_request'
    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

  # ============ Job 4: Deploy to Staging ============
  deploy-staging:
    needs: [build, security]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://staging.example.com

    steps:
      - name: Deploy to staging
        run: |
          echo "Deploying to staging..."
          # kubectl set image deployment/myapp \
          #   myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}

  # ============ Job 5: Deploy to Production ============
  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://example.com

    steps:
      - name: Deploy to production
        run: |
          echo "Deploying to production..."
          # kubectl set image deployment/myapp \
          #   myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
```

### GitHub Actions - Matrix Strategy

```yaml
# Test across multiple versions and operating systems
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest]
        node-version: [18, 20, 22]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci
      - run: npm test
```

### GitHub Actions - Secrets & Variables

```yaml
# Using secrets in workflows
steps:
  - name: Deploy with secrets
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
    run: |
      echo "Deploying with AWS credentials..."
      # Never echo secrets!

# To add secrets:
# GitHub → Repository → Settings → Secrets and variables → Actions → New repository secret
```

### GitHub Actions - Reusable Workflows

```yaml
# File: .github/workflows/reusable-docker.yml
name: Reusable Docker Build

on:
  workflow_call:
    inputs:
      image-name:
        required: true
        type: string
      dockerfile:
        required: false
        type: string
        default: 'Dockerfile'
    secrets:
      registry-password:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and push
        run: |
          docker build -t ${{ inputs.image-name }} -f ${{ inputs.dockerfile }} .
          docker push ${{ inputs.image-name }}
```

```yaml
# File: .github/workflows/main.yml - Using the reusable workflow
jobs:
  build-api:
    uses: ./.github/workflows/reusable-docker.yml
    with:
      image-name: ghcr.io/myorg/api:latest
      dockerfile: api/Dockerfile
    secrets:
      registry-password: ${{ secrets.GITHUB_TOKEN }}
```

---

## GitLab CI

```yaml
# File: .gitlab-ci.yml
stages:
  - build
  - test
  - package
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE
  NODE_VERSION: "20"

# Cache node_modules between jobs
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/

# ============ Build Stage ============
build:
  stage: build
  image: node:${NODE_VERSION}
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour

# ============ Test Stage ============
unit-tests:
  stage: test
  image: node:${NODE_VERSION}
  script:
    - npm ci
    - npm run test:unit -- --coverage
  coverage: '/All files\s*\|\s*(\d+\.?\d*)\s*\|/'
  artifacts:
    reports:
      junit: test-results/junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura.xml

lint:
  stage: test
  image: node:${NODE_VERSION}
  script:
    - npm ci
    - npm run lint

# ============ Package Stage ============
docker-build:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker push $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA
    - docker tag $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA $DOCKER_IMAGE:latest
    - docker push $DOCKER_IMAGE:latest
  only:
    - main

# ============ Deploy Stages ============
deploy-staging:
  stage: deploy
  image: alpine:latest
  script:
    - echo "Deploying $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA to staging..."
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - main

deploy-production:
  stage: deploy
  image: alpine:latest
  script:
    - echo "Deploying $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA to production..."
  environment:
    name: production
    url: https://example.com
  when: manual
  only:
    - main
```

---

## Tool Comparison

| Feature | Jenkins | GitHub Actions | GitLab CI |
|---------|---------|----------------|-----------|
| **Hosting** | Self-hosted | Cloud (GitHub) | Cloud or Self-hosted |
| **Config** | Jenkinsfile (Groovy) | YAML | YAML |
| **Setup** | Complex | Zero setup | Zero setup (GitLab) |
| **Plugins** | 1800+ plugins | Marketplace Actions | Built-in features |
| **Cost** | Free (self-hosted) | Free tier available | Free tier available |
| **Runners** | Master-Slave agents | GitHub-hosted or self-hosted | Shared or self-hosted |
| **Best for** | Enterprise, complex | GitHub repos, open source | GitLab repos, all-in-one |
| **Learning curve** | Steep | Easy | Moderate |

---

## Projects

### Project 1: GitHub Actions CI Pipeline

```bash
# Create project structure
mkdir -p ci-demo/.github/workflows ci-demo/src ci-demo/tests
cd ci-demo && git init
```

```json
// File: package.json
{
  "name": "ci-demo",
  "version": "1.0.0",
  "scripts": {
    "start": "node src/app.js",
    "test": "jest --forceExit",
    "lint": "echo 'Linting passed'"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "jest": "^29.7.0",
    "supertest": "^6.3.3"
  }
}
```

```javascript
// File: src/app.js
const express = require('express');
const app = express();

app.get('/', (req, res) => res.json({ message: 'Hello CI/CD!' }));
app.get('/health', (req, res) => res.json({ status: 'healthy' }));
app.get('/add/:a/:b', (req, res) => {
  const sum = parseInt(req.params.a) + parseInt(req.params.b);
  res.json({ result: sum });
});

module.exports = app;

if (require.main === module) {
  app.listen(3000, () => console.log('Server on port 3000'));
}
```

```javascript
// File: tests/app.test.js
const request = require('supertest');
const app = require('../src/app');

describe('API Tests', () => {
  test('GET / returns hello message', async () => {
    const res = await request(app).get('/');
    expect(res.status).toBe(200);
    expect(res.body.message).toBe('Hello CI/CD!');
  });

  test('GET /health returns healthy', async () => {
    const res = await request(app).get('/health');
    expect(res.status).toBe(200);
    expect(res.body.status).toBe('healthy');
  });

  test('GET /add/2/3 returns 5', async () => {
    const res = await request(app).get('/add/2/3');
    expect(res.status).toBe(200);
    expect(res.body.result).toBe(5);
  });
});
```

```dockerfile
# File: Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY src/ ./src/
EXPOSE 3000
CMD ["node", "src/app.js"]
```

```yaml
# File: .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm test

  build-docker:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image
        run: docker build -t ci-demo:${{ github.sha }} .
      - name: Test Docker image
        run: |
          docker run -d --name test-app -p 3000:3000 ci-demo:${{ github.sha }}
          sleep 3
          curl -f http://localhost:3000/health
          docker stop test-app
```

```bash
# Push to GitHub and watch the pipeline run!
git add .
git commit -m "Add CI pipeline"
git remote add origin git@github.com:yourusername/ci-demo.git
git push -u origin main

# View pipeline: GitHub → Actions tab
```

### Project 2: Jenkins Pipeline with Docker

```groovy
// File: Jenkinsfile
pipeline {
    agent any

    environment {
        APP_NAME = 'ci-demo'
        DOCKER_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'echo "Build #${BUILD_NUMBER} - Branch: ${BRANCH_NAME}"'
            }
        }

        stage('Install & Lint') {
            steps {
                sh 'npm ci'
                sh 'npm run lint'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${APP_NAME}:${DOCKER_TAG} ."
            }
        }

        stage('Test Docker Image') {
            steps {
                sh """
                    docker run -d --name ${APP_NAME}-test -p 3001:3000 ${APP_NAME}:${DOCKER_TAG}
                    sleep 3
                    curl -f http://localhost:3001/health
                    docker stop ${APP_NAME}-test
                    docker rm ${APP_NAME}-test
                """
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            steps {
                sh """
                    docker stop ${APP_NAME} || true
                    docker rm ${APP_NAME} || true
                    docker run -d --name ${APP_NAME} -p 3000:3000 ${APP_NAME}:${DOCKER_TAG}
                """
                echo "Deployed ${APP_NAME}:${DOCKER_TAG}"
            }
        }
    }

    post {
        always {
            sh "docker rmi ${APP_NAME}:${DOCKER_TAG} || true"
            cleanWs()
        }
        success { echo 'Pipeline succeeded!' }
        failure { echo 'Pipeline failed!' }
    }
}
```
