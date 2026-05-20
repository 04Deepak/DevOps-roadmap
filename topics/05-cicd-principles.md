# 5. CI/CD Principles

## Table of Contents
- [What is CI/CD?](#what-is-cicd)
- [CI/CD Pipeline Stages](#cicd-pipeline-stages)
- [Testing Strategy](#testing-strategy)
- [Artifact Management](#artifact-management)
- [Code Quality](#code-quality)
- [Branching & Release Models](#branching--release-models)
- [Projects](#projects)

---

## What is CI/CD?

```
CI/CD = Continuous Integration + Continuous Delivery + Continuous Deployment

Developer                                                          Production
   │                                                                    │
   │   Push Code                                                        │
   ├──────────►  CI: Continuous Integration                             │
   │             ├── Pull latest code                                   │
   │             ├── Build the application                              │
   │             ├── Run automated tests                                │
   │             └── Report results ──────────►  Feedback to developer  │
   │                                                                    │
   │             CD: Continuous Delivery                                │
   │             ├── Package the application                            │
   │             ├── Deploy to staging                                  │
   │             ├── Run integration tests                              │
   │             └── Ready for release ───────►  Manual approval        │
   │                                                      │             │
   │             CD: Continuous Deployment                 │             │
   │             ├── Auto-deploy to production ◄───────────┘             │
   │             ├── Run smoke tests                                    │
   │             └── Monitor ─────────────────────────────────────────► │
   │                                                                    │
```

### Key Differences

| Term | Definition | Approval |
|------|-----------|----------|
| **Continuous Integration (CI)** | Automatically build and test every code change | Automatic |
| **Continuous Delivery (CD)** | Automatically prepare releases, deploy to staging | Manual approval to production |
| **Continuous Deployment (CD)** | Automatically deploy every change to production | Fully automatic |

### Benefits of CI/CD

```
Without CI/CD:                          With CI/CD:
┌─────────────────────┐                ┌─────────────────────┐
│ Manual builds        │                │ Automated builds     │
│ "Works on my machine"│                │ Consistent env       │
│ Deploy once a month  │                │ Deploy multiple/day  │
│ Find bugs in prod    │                │ Catch bugs early     │
│ Long integration     │                │ Small, fast merges   │
│ Risky releases       │                │ Safe, repeatable     │
│ Manual testing       │                │ Automated testing    │
└─────────────────────┘                └─────────────────────┘
```

---

## CI/CD Pipeline Stages

```
┌────────┐   ┌────────┐   ┌─────────┐   ┌─────────┐   ┌────────┐   ┌─────────┐
│ SOURCE │──►│ BUILD  │──►│  TEST   │──►│ PACKAGE │──►│ DEPLOY │──►│ MONITOR │
│        │   │        │   │         │   │         │   │        │   │         │
│ Git    │   │ Compile│   │ Unit    │   │ Docker  │   │ Stage  │   │ Metrics │
│ Push   │   │ Deps   │   │ Integ.  │   │ Image   │   │ Prod   │   │ Logs    │
│ PR     │   │ Lint   │   │ E2E     │   │ Artifact│   │ Canary │   │ Alerts  │
└────────┘   └────────┘   └─────────┘   └─────────┘   └────────┘   └─────────┘
```

### Stage 1: Source

```
Trigger: Code push, pull request, scheduled build

Actions:
- Pull the latest source code from the repository
- Determine what changed (which files, which services)
- Decide which pipeline to run (e.g., skip if only docs changed)
```

### Stage 2: Build

```
Actions:
- Install dependencies (npm install, pip install, etc.)
- Compile code (if applicable: TypeScript, Go, Java)
- Run linters (ESLint, Pylint, golangci-lint)
- Static analysis (SonarQube, CodeClimate)

Example (Node.js):
  npm ci                    # Install exact dependency versions
  npm run lint              # Check code style
  npm run build             # Compile/bundle
```

### Stage 3: Test

```
Test Pyramid:
                    ┌───────┐
                    │  E2E  │        Few, slow, expensive
                    │ Tests │        (Selenium, Cypress)
                   ┌┴───────┴┐
                   │Integration│     Medium amount
                   │  Tests   │     (API tests, DB tests)
                  ┌┴──────────┴┐
                  │  Unit Tests │   Many, fast, cheap
                  │             │   (Jest, pytest, JUnit)
                  └─────────────┘

Actions:
- Run unit tests (fast, isolated)
- Run integration tests (with real dependencies)
- Run security scanning (SAST/DAST)
- Generate code coverage report
```

### Stage 4: Package

```
Actions:
- Build Docker image
- Tag image with version (git SHA, semver)
- Push image to container registry
- Store build artifacts

Tagging Strategy:
  myapp:latest                  # Latest build
  myapp:1.2.3                   # Semantic version
  myapp:abc123f                 # Git commit SHA
  myapp:main-20240115-abc123f   # Branch + date + SHA
```

### Stage 5: Deploy

```
Environments:
  Development  → Auto-deploy on every push
  Staging      → Auto-deploy on merge to main
  Production   → Manual approval (or auto with CD)

Deployment Strategies:
  Rolling Update    → Replace instances one by one
  Blue-Green        → Switch traffic between two environments
  Canary            → Send small % of traffic to new version
```

### Stage 6: Monitor

```
Actions:
- Run smoke tests after deployment
- Monitor application metrics (latency, errors, throughput)
- Check logs for errors
- Automatic rollback if health checks fail
```

---

## Testing Strategy

### Unit Tests

```python
# File: test_calculator.py
import pytest
from calculator import Calculator

class TestCalculator:
    def setup_method(self):
        self.calc = Calculator()

    def test_add(self):
        assert self.calc.add(2, 3) == 5

    def test_add_negative(self):
        assert self.calc.add(-1, 1) == 0

    def test_divide(self):
        assert self.calc.divide(10, 2) == 5

    def test_divide_by_zero(self):
        with pytest.raises(ValueError):
            self.calc.divide(10, 0)
```

```bash
# Run tests
pytest test_calculator.py -v

# With coverage
pytest --cov=. --cov-report=html test_calculator.py
```

### Integration Tests

```python
# File: test_api_integration.py
import requests
import pytest

BASE_URL = "http://localhost:3000"

class TestAPIIntegration:
    def test_health_endpoint(self):
        response = requests.get(f"{BASE_URL}/health")
        assert response.status_code == 200
        assert response.json()["status"] == "healthy"

    def test_create_and_get_user(self):
        # Create user
        user_data = {"name": "Test User", "email": "test@example.com"}
        response = requests.post(f"{BASE_URL}/api/users", json=user_data)
        assert response.status_code == 201
        user_id = response.json()["id"]

        # Get user
        response = requests.get(f"{BASE_URL}/api/users/{user_id}")
        assert response.status_code == 200
        assert response.json()["name"] == "Test User"

    def test_database_connection(self):
        response = requests.get(f"{BASE_URL}/api/db-status")
        assert response.status_code == 200
        assert response.json()["connected"] is True
```

### E2E Tests (Example with Playwright)

```javascript
// File: tests/e2e/login.spec.js
const { test, expect } = require('@playwright/test');

test('user can login', async ({ page }) => {
  await page.goto('http://localhost:3000/login');
  await page.fill('#email', 'user@example.com');
  await page.fill('#password', 'password123');
  await page.click('button[type="submit"]');
  await expect(page).toHaveURL('/dashboard');
  await expect(page.locator('h1')).toContainText('Welcome');
});
```

---

## Artifact Management

```
What are Artifacts?
- Build outputs: compiled binaries, Docker images, JAR files
- Test results: JUnit XML, coverage reports
- Configuration files: deployment configs, Terraform plans

Where to Store:
┌─────────────────────────────────────────────┐
│ Artifact Type        │ Storage               │
├─────────────────────────────────────────────┤
│ Docker images        │ Docker Hub, ECR, GCR  │
│ npm packages         │ npm registry          │
│ Python packages      │ PyPI, private index   │
│ Generic artifacts    │ Nexus, Artifactory    │
│ Build logs/reports   │ S3, CI tool storage   │
└─────────────────────────────────────────────┘

Versioning Strategy:
  Semantic Versioning: MAJOR.MINOR.PATCH
  - MAJOR: Breaking changes (2.0.0)
  - MINOR: New features, backward compatible (1.1.0)
  - PATCH: Bug fixes (1.0.1)

  Example: 1.0.0 → 1.0.1 (bug fix) → 1.1.0 (new feature) → 2.0.0 (breaking)
```

---

## Code Quality

### Linting

```bash
# Python
pip install pylint flake8 black
pylint app.py                  # Code analysis
flake8 app.py                  # Style checking
black app.py                   # Auto-format code

# JavaScript/TypeScript
npm install -D eslint prettier
npx eslint src/                # Lint code
npx prettier --check src/      # Check formatting

# Go
golangci-lint run

# YAML (for DevOps configs)
pip install yamllint
yamllint docker-compose.yml
yamllint .github/workflows/
```

### Pre-commit Hooks

```bash
# Install pre-commit
pip install pre-commit
```

```yaml
# File: .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-merge-conflict
      - id: detect-private-key

  - repo: https://github.com/psf/black
    rev: 24.1.0
    hooks:
      - id: black

  - repo: https://github.com/pycqa/flake8
    rev: 7.0.0
    hooks:
      - id: flake8
```

```bash
# Install hooks
pre-commit install

# Now hooks run automatically on git commit
git add . && git commit -m "test"
# Output:
# Trim trailing whitespace.......Passed
# Fix end of files...............Passed
# Check YAML....................Passed
# black.........................Passed
# flake8........................Passed
```

---

## Branching & Release Models

### Trunk-Based Development

```
main (trunk): ──A──B──C──D──E──F──G──H──
                     │        │
         feature/x: ─┘        │
         (short-lived,        │
          merged in <1 day)   │
                              │
         feature/y: ──────────┘
         (short-lived)

Rules:
- Everyone commits to main (or very short-lived branches)
- Branches live for hours, not days/weeks
- Feature flags control what users see
- Continuous deployment from main
```

### Release Branches

```
main: ──A──B──C──D──E──F──G──H──I──J──
              │           │
   release/1.0: ──C'──D'  │
              (bug fixes)  │
                           │
              release/1.1: ──F'──G'
                           (bug fixes)

Rules:
- Cut a release branch when ready to ship
- Only bug fixes go to release branch
- Merge fixes back to main
- Good for scheduled releases
```

---

## Projects

### Project 1: CI/CD Pipeline Flowchart

```
Design a CI/CD pipeline for a microservice app:

┌─────────────────────────────────────────────────────────────────┐
│                    CI/CD Pipeline Design                         │
│                    E-Commerce Microservices                      │
└─────────────────────────────────────────────────────────────────┘

Services: user-service, product-service, order-service, api-gateway

Trigger: Push to any service directory
         PR to main branch

┌──────────────────────────────────────────────────────────┐
│ Stage 1: Detect Changes                                   │
│ ─────────────────────                                     │
│ - Identify which service(s) changed                       │
│ - Only build/test changed services                        │
│ - If shared-lib changed → build all services             │
└───────────────────────┬──────────────────────────────────┘
                        ▼
┌──────────────────────────────────────────────────────────┐
│ Stage 2: Build & Lint (per service, parallel)             │
│ ────────────────────────────────────                      │
│ - Install dependencies                                    │
│ - Run linter (ESLint / Pylint)                           │
│ - Compile TypeScript → JavaScript                        │
│ - Build Docker image                                      │
│ - Fail fast: stop if any service fails                   │
└───────────────────────┬──────────────────────────────────┘
                        ▼
┌──────────────────────────────────────────────────────────┐
│ Stage 3: Test (per service, parallel)                     │
│ ──────────────────────────────                            │
│ - Unit tests (pytest / jest)                             │
│ - Integration tests (with test DB)                       │
│ - Code coverage (minimum 80%)                            │
│ - Security scan (Trivy for Docker image)                 │
└───────────────────────┬──────────────────────────────────┘
                        ▼
┌──────────────────────────────────────────────────────────┐
│ Stage 4: Package                                          │
│ ───────────────                                           │
│ - Tag Docker image: service:commit-sha                   │
│ - Push to ECR registry                                    │
│ - Store test reports as artifacts                         │
└───────────────────────┬──────────────────────────────────┘
                        ▼
┌──────────────────────────────────────────────────────────┐
│ Stage 5: Deploy to Staging                                │
│ ──────────────────────────                                │
│ - Update Kubernetes deployment with new image            │
│ - Wait for rollout to complete                           │
│ - Run E2E tests against staging                          │
│ - Run performance tests (k6 / locust)                    │
└───────────────────────┬──────────────────────────────────┘
                        ▼
┌──────────────────────────────────────────────────────────┐
│ Stage 6: Deploy to Production (manual approval)           │
│ ───────────────────────────────────────────               │
│ - Require team lead approval                             │
│ - Canary deployment (10% → 50% → 100%)                  │
│ - Smoke tests after each step                            │
│ - Automatic rollback if error rate > 1%                  │
└───────────────────────┬──────────────────────────────────┘
                        ▼
┌──────────────────────────────────────────────────────────┐
│ Stage 7: Post-Deploy                                      │
│ ────────────────────                                      │
│ - Tag git commit with version                            │
│ - Update changelog                                        │
│ - Notify team (Slack/Teams)                              │
│ - Monitor dashboards for 30 minutes                      │
└──────────────────────────────────────────────────────────┘
```

### Project 2: Testing Strategy Document

```markdown
# Testing Strategy: Containerized Web Application

## Overview
This document outlines the testing strategy for our Dockerized web application
consisting of a Node.js API backend and a React frontend.

## Test Layers

### 1. Unit Tests (70% of tests)
- **Tool:** Jest
- **Run when:** Every commit
- **Coverage target:** 80%+
- **What to test:**
  - Individual functions and methods
  - Business logic
  - Data transformations
  - Error handling

### 2. Integration Tests (20% of tests)
- **Tool:** Supertest + Test containers
- **Run when:** Every PR
- **What to test:**
  - API endpoints with real database
  - Service-to-service communication
  - Database queries and migrations
  - Authentication flows

### 3. E2E Tests (10% of tests)
- **Tool:** Playwright
- **Run when:** Before deployment to staging
- **What to test:**
  - Critical user journeys (login, checkout, etc.)
  - Cross-browser compatibility
  - Mobile responsiveness

## CI Pipeline Test Execution

| Stage | Tests | Timeout | Fail Action |
|-------|-------|---------|-------------|
| Pre-commit | Linting, formatting | 30s | Block commit |
| CI - Build | Unit tests | 5min | Block merge |
| CI - Integration | Integration tests | 10min | Block merge |
| Pre-deploy | E2E tests | 15min | Block deployment |
| Post-deploy | Smoke tests | 2min | Auto-rollback |

## Test Environment
- Unit tests: In-memory mocks
- Integration tests: Docker Compose (real DB, real Redis)
- E2E tests: Staging environment (mirrors production)
```

---

## Key Takeaways

1. **CI/CD is a culture**, not just tools
2. **Automate everything** that can be automated
3. **Test pyramid**: many unit tests, fewer integration, fewest E2E
4. **Fail fast**: catch issues as early as possible in the pipeline
5. **Small, frequent changes** are safer than large, infrequent ones
6. **Every commit should be deployable** (or at least buildable)
7. **Monitor after deploy** — deployment is not the end of the pipeline
