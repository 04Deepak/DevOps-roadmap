# 14. End-to-End DevOps Capstone Project

## Table of Contents
- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Step 1: Application Setup](#step-1-application-setup)
- [Step 2: Dockerize All Services](#step-2-dockerize-all-services)
- [Step 3: Git Repository & Branching Strategy](#step-3-git-repository--branching-strategy)
- [Step 4: CI/CD Pipeline with GitHub Actions](#step-4-cicd-pipeline-with-github-actions)
- [Step 5: Terraform Infrastructure](#step-5-terraform-infrastructure)
- [Step 6: Kubernetes Manifests & Helm Charts](#step-6-kubernetes-manifests--helm-charts)
- [Step 7: ArgoCD GitOps Setup](#step-7-argocd-gitops-setup)
- [Step 8: Monitoring with Prometheus & Grafana](#step-8-monitoring-with-prometheus--grafana)
- [Step 9: Security Scanning in Pipeline](#step-9-security-scanning-in-pipeline)
- [Step 10: Documentation Requirements](#step-10-documentation-requirements)
- [Evaluation Rubric](#evaluation-rubric)
- [Projects](#projects)

---

## Project Overview

In this capstone project, you will build a complete DevOps pipeline for a microservices application from scratch. This project ties together everything you have learned throughout the course: containerization, CI/CD, infrastructure as code, Kubernetes, GitOps, monitoring, and security.

### What You Will Build

```
A simple e-commerce backend with three microservices:

1. Product Service (Node.js)   - Manages product catalog (CRUD operations)
2. Order Service (Python/Flask) - Handles order placement and tracking
3. API Gateway (Node.js)        - Routes requests to the correct service

Supporting infrastructure:
- PostgreSQL database for persistent storage
- Redis for caching
- NGINX Ingress for external access
```

### Learning Objectives

```
By completing this project, you will demonstrate proficiency in:

  [ ] Writing Dockerfiles and docker-compose for multi-service applications
  [ ] Setting up Git workflows with branching strategies
  [ ] Building CI/CD pipelines with automated testing and deployment
  [ ] Provisioning cloud infrastructure with Terraform
  [ ] Deploying to Kubernetes with Helm charts
  [ ] Implementing GitOps with ArgoCD
  [ ] Setting up monitoring and alerting with Prometheus and Grafana
  [ ] Integrating security scanning into CI/CD pipelines
  [ ] Writing operational documentation
```

---

## Architecture

### System Architecture Diagram

```
                         Internet
                            |
                     [Load Balancer]
                            |
                    [NGINX Ingress Controller]
                            |
               +------------+------------+
               |                         |
        /api/products             /api/orders
               |                         |
    +----------+----------+   +----------+----------+
    |  Product Service    |   |   Order Service     |
    |  (Node.js:3001)     |   |   (Python:3002)     |
    |  Replicas: 2-5      |   |   Replicas: 2-5     |
    +----------+----------+   +----------+----------+
               |                         |
               +----+              +-----+
                    |              |
              [PostgreSQL]    [PostgreSQL]
              (products_db)   (orders_db)
                    |              |
                    +----- [Redis Cache] -----+
                    
    Monitoring Stack:
    [Prometheus] --> [Grafana] --> [Alertmanager]
         |
    [Node Exporter + kube-state-metrics]
    
    GitOps:
    [GitHub Repo] <--- [ArgoCD] ---> [Kubernetes Cluster]
```

### Technology Stack

```
Component          | Technology       | Version
-------------------|-----------------|----------
Product Service    | Node.js/Express | 20 LTS
Order Service      | Python/Flask    | 3.12
Database           | PostgreSQL      | 16
Cache              | Redis           | 7
Container Runtime  | Docker          | 24+
Orchestration      | Kubernetes      | 1.28+
CI/CD              | GitHub Actions  | v4
IaC                | Terraform       | 1.7+
GitOps             | ArgoCD          | 2.10+
Monitoring         | Prometheus      | 2.50+
Dashboards         | Grafana         | 10+
Security Scan      | Trivy           | 0.50+
Package Manager    | Helm            | 3.14+
```

---

## Step 1: Application Setup

### Product Service (Node.js)

```javascript
// product-service/src/index.js
const express = require('express');
const { Pool } = require('pg');
const redis = require('redis');
const promClient = require('prom-client');

const app = express();
app.use(express.json());

// --- Prometheus Metrics ---
const register = new promClient.Registry();
promClient.collectDefaultMetrics({ register });

const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 2, 5],
  registers: [register],
});

const httpRequestsTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  registers: [register],
});

// Middleware: track request metrics
app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer();
  res.on('finish', () => {
    const route = req.route ? req.route.path : req.path;
    end({ method: req.method, route, status_code: res.statusCode });
    httpRequestsTotal.inc({ method: req.method, route, status_code: res.statusCode });
  });
  next();
});

// --- Database Connection ---
const pool = new Pool({
  host: process.env.DB_HOST || 'localhost',
  port: process.env.DB_PORT || 5432,
  database: process.env.DB_NAME || 'products_db',
  user: process.env.DB_USER || 'postgres',
  password: process.env.DB_PASSWORD || 'postgres',
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// --- Redis Connection ---
const redisClient = redis.createClient({
  url: `redis://${process.env.REDIS_HOST || 'localhost'}:${process.env.REDIS_PORT || 6379}`,
});
redisClient.on('error', (err) => console.error('Redis error:', err));
redisClient.connect().catch(console.error);

// --- Initialize Database ---
async function initDB() {
  const client = await pool.connect();
  try {
    await client.query(`
      CREATE TABLE IF NOT EXISTS products (
        id SERIAL PRIMARY KEY,
        name VARCHAR(255) NOT NULL,
        description TEXT,
        price DECIMAL(10,2) NOT NULL,
        stock INTEGER DEFAULT 0,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      )
    `);
    console.log('Database initialized');
  } finally {
    client.release();
  }
}

// --- Routes ---
// Health check
app.get('/health', async (req, res) => {
  try {
    await pool.query('SELECT 1');
    res.json({ status: 'healthy', service: 'product-service' });
  } catch (err) {
    res.status(503).json({ status: 'unhealthy', error: err.message });
  }
});

// Prometheus metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

// Get all products
app.get('/api/products', async (req, res) => {
  try {
    // Check Redis cache first
    const cached = await redisClient.get('products:all');
    if (cached) {
      return res.json(JSON.parse(cached));
    }
    const result = await pool.query(
      'SELECT * FROM products ORDER BY created_at DESC'
    );
    // Cache for 60 seconds
    await redisClient.setEx('products:all', 60, JSON.stringify(result.rows));
    res.json(result.rows);
  } catch (err) {
    console.error('Error fetching products:', err);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Get product by ID
app.get('/api/products/:id', async (req, res) => {
  try {
    const { id } = req.params;
    const cached = await redisClient.get(`products:${id}`);
    if (cached) {
      return res.json(JSON.parse(cached));
    }
    const result = await pool.query('SELECT * FROM products WHERE id = $1', [id]);
    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'Product not found' });
    }
    await redisClient.setEx(`products:${id}`, 60, JSON.stringify(result.rows[0]));
    res.json(result.rows[0]);
  } catch (err) {
    console.error('Error fetching product:', err);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Create product
app.post('/api/products', async (req, res) => {
  try {
    const { name, description, price, stock } = req.body;
    if (!name || price === undefined) {
      return res.status(400).json({ error: 'Name and price are required' });
    }
    const result = await pool.query(
      'INSERT INTO products (name, description, price, stock) VALUES ($1, $2, $3, $4) RETURNING *',
      [name, description || '', price, stock || 0]
    );
    // Invalidate cache
    await redisClient.del('products:all');
    res.status(201).json(result.rows[0]);
  } catch (err) {
    console.error('Error creating product:', err);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Update product
app.put('/api/products/:id', async (req, res) => {
  try {
    const { id } = req.params;
    const { name, description, price, stock } = req.body;
    const result = await pool.query(
      `UPDATE products SET name = COALESCE($1, name), description = COALESCE($2, description),
       price = COALESCE($3, price), stock = COALESCE($4, stock), updated_at = CURRENT_TIMESTAMP
       WHERE id = $5 RETURNING *`,
      [name, description, price, stock, id]
    );
    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'Product not found' });
    }
    await redisClient.del('products:all');
    await redisClient.del(`products:${id}`);
    res.json(result.rows[0]);
  } catch (err) {
    console.error('Error updating product:', err);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Delete product
app.delete('/api/products/:id', async (req, res) => {
  try {
    const { id } = req.params;
    const result = await pool.query(
      'DELETE FROM products WHERE id = $1 RETURNING *', [id]
    );
    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'Product not found' });
    }
    await redisClient.del('products:all');
    await redisClient.del(`products:${id}`);
    res.status(204).send();
  } catch (err) {
    console.error('Error deleting product:', err);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// --- Start Server ---
const PORT = process.env.PORT || 3001;

initDB().then(() => {
  app.listen(PORT, () => {
    console.log(`Product service running on port ${PORT}`);
  });
}).catch((err) => {
  console.error('Failed to initialize database:', err);
  process.exit(1);
});

module.exports = app;
```

```json
// product-service/package.json
{
  "name": "product-service",
  "version": "1.0.0",
  "description": "Product catalog microservice",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js",
    "test": "jest --coverage",
    "lint": "eslint src/"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.12.0",
    "redis": "^4.6.13",
    "prom-client": "^15.1.0"
  },
  "devDependencies": {
    "jest": "^29.7.0",
    "supertest": "^6.3.4",
    "nodemon": "^3.0.3",
    "eslint": "^8.56.0"
  }
}
```

### Order Service (Python/Flask)

```python
# order-service/src/app.py
import os
import logging
from datetime import datetime
from flask import Flask, jsonify, request
import psycopg2
from psycopg2.extras import RealDictCursor
from psycopg2 import pool
from prometheus_flask_instrumentator import FlaskInstrumentator
import requests

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# --- Database Connection Pool ---
db_pool = psycopg2.pool.ThreadedConnectionPool(
    minconn=2,
    maxconn=20,
    host=os.getenv('DB_HOST', 'localhost'),
    port=os.getenv('DB_PORT', '5432'),
    dbname=os.getenv('DB_NAME', 'orders_db'),
    user=os.getenv('DB_USER', 'postgres'),
    password=os.getenv('DB_PASSWORD', 'postgres'),
)

PRODUCT_SERVICE_URL = os.getenv('PRODUCT_SERVICE_URL', 'http://localhost:3001')

# --- Prometheus Metrics ---
FlaskInstrumentator().instrument(app)

# --- Initialize Database ---
def init_db():
    conn = db_pool.getconn()
    try:
        with conn.cursor() as cur:
            cur.execute("""
                CREATE TABLE IF NOT EXISTS orders (
                    id SERIAL PRIMARY KEY,
                    product_id INTEGER NOT NULL,
                    quantity INTEGER NOT NULL DEFAULT 1,
                    total_price DECIMAL(10,2) NOT NULL,
                    status VARCHAR(50) DEFAULT 'pending',
                    customer_email VARCHAR(255),
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
            """)
            conn.commit()
            logger.info("Database initialized")
    finally:
        db_pool.putconn(conn)


# --- Routes ---
@app.route('/health')
def health():
    try:
        conn = db_pool.getconn()
        with conn.cursor() as cur:
            cur.execute('SELECT 1')
        db_pool.putconn(conn)
        return jsonify({'status': 'healthy', 'service': 'order-service'})
    except Exception as e:
        return jsonify({'status': 'unhealthy', 'error': str(e)}), 503


@app.route('/api/orders', methods=['GET'])
def get_orders():
    conn = db_pool.getconn()
    try:
        with conn.cursor(cursor_factory=RealDictCursor) as cur:
            cur.execute('SELECT * FROM orders ORDER BY created_at DESC')
            orders = cur.fetchall()
        # Convert datetime objects for JSON serialization
        for order in orders:
            order['created_at'] = order['created_at'].isoformat()
            order['updated_at'] = order['updated_at'].isoformat()
            order['total_price'] = float(order['total_price'])
        return jsonify(orders)
    finally:
        db_pool.putconn(conn)


@app.route('/api/orders/<int:order_id>', methods=['GET'])
def get_order(order_id):
    conn = db_pool.getconn()
    try:
        with conn.cursor(cursor_factory=RealDictCursor) as cur:
            cur.execute('SELECT * FROM orders WHERE id = %s', (order_id,))
            order = cur.fetchone()
        if not order:
            return jsonify({'error': 'Order not found'}), 404
        order['created_at'] = order['created_at'].isoformat()
        order['updated_at'] = order['updated_at'].isoformat()
        order['total_price'] = float(order['total_price'])
        return jsonify(order)
    finally:
        db_pool.putconn(conn)


@app.route('/api/orders', methods=['POST'])
def create_order():
    data = request.get_json()
    if not data or 'product_id' not in data or 'quantity' not in data:
        return jsonify({'error': 'product_id and quantity are required'}), 400

    product_id = data['product_id']
    quantity = data['quantity']
    customer_email = data.get('customer_email', '')

    # Verify product exists and get price from Product Service
    try:
        resp = requests.get(
            f'{PRODUCT_SERVICE_URL}/api/products/{product_id}',
            timeout=5
        )
        if resp.status_code == 404:
            return jsonify({'error': 'Product not found'}), 404
        resp.raise_for_status()
        product = resp.json()
    except requests.exceptions.RequestException as e:
        logger.error(f'Failed to fetch product: {e}')
        return jsonify({'error': 'Product service unavailable'}), 503

    total_price = float(product['price']) * quantity

    conn = db_pool.getconn()
    try:
        with conn.cursor(cursor_factory=RealDictCursor) as cur:
            cur.execute(
                """INSERT INTO orders (product_id, quantity, total_price, customer_email, status)
                   VALUES (%s, %s, %s, %s, 'pending') RETURNING *""",
                (product_id, quantity, total_price, customer_email)
            )
            order = cur.fetchone()
            conn.commit()
        order['created_at'] = order['created_at'].isoformat()
        order['updated_at'] = order['updated_at'].isoformat()
        order['total_price'] = float(order['total_price'])
        return jsonify(order), 201
    finally:
        db_pool.putconn(conn)


@app.route('/api/orders/<int:order_id>/status', methods=['PUT'])
def update_order_status(order_id):
    data = request.get_json()
    if not data or 'status' not in data:
        return jsonify({'error': 'status is required'}), 400

    valid_statuses = ['pending', 'confirmed', 'shipped', 'delivered', 'cancelled']
    if data['status'] not in valid_statuses:
        return jsonify({'error': f'Invalid status. Must be one of: {valid_statuses}'}), 400

    conn = db_pool.getconn()
    try:
        with conn.cursor(cursor_factory=RealDictCursor) as cur:
            cur.execute(
                """UPDATE orders SET status = %s, updated_at = CURRENT_TIMESTAMP
                   WHERE id = %s RETURNING *""",
                (data['status'], order_id)
            )
            order = cur.fetchone()
            conn.commit()
        if not order:
            return jsonify({'error': 'Order not found'}), 404
        order['created_at'] = order['created_at'].isoformat()
        order['updated_at'] = order['updated_at'].isoformat()
        order['total_price'] = float(order['total_price'])
        return jsonify(order)
    finally:
        db_pool.putconn(conn)


if __name__ == '__main__':
    init_db()
    port = int(os.getenv('PORT', 3002))
    app.run(host='0.0.0.0', port=port, debug=os.getenv('FLASK_DEBUG', 'false') == 'true')
```

```
# order-service/requirements.txt
flask==3.0.2
psycopg2-binary==2.9.9
prometheus-flask-instrumentator==6.1.0
requests==2.31.0
gunicorn==21.2.0
pytest==8.0.2
pytest-flask==1.3.0
```

### Product Service Tests

```javascript
// product-service/src/__tests__/products.test.js
const request = require('supertest');

// Mock dependencies before requiring the app
jest.mock('pg', () => {
  const mPool = {
    connect: jest.fn().mockResolvedValue({
      query: jest.fn().mockResolvedValue({ rows: [] }),
      release: jest.fn(),
    }),
    query: jest.fn(),
  };
  return { Pool: jest.fn(() => mPool) };
});

jest.mock('redis', () => ({
  createClient: jest.fn(() => ({
    connect: jest.fn().mockResolvedValue(undefined),
    get: jest.fn().mockResolvedValue(null),
    setEx: jest.fn().mockResolvedValue('OK'),
    del: jest.fn().mockResolvedValue(1),
    on: jest.fn(),
  })),
}));

const app = require('../index');
const { Pool } = require('pg');

describe('Product Service API', () => {
  let mockPool;

  beforeEach(() => {
    mockPool = new Pool();
    jest.clearAllMocks();
  });

  describe('GET /health', () => {
    it('should return healthy status', async () => {
      mockPool.query.mockResolvedValueOnce({ rows: [{ '?column?': 1 }] });
      const res = await request(app).get('/health');
      expect(res.statusCode).toBe(200);
      expect(res.body.status).toBe('healthy');
    });
  });

  describe('GET /api/products', () => {
    it('should return a list of products', async () => {
      const mockProducts = [
        { id: 1, name: 'Widget', price: 9.99, stock: 100 },
        { id: 2, name: 'Gadget', price: 19.99, stock: 50 },
      ];
      mockPool.query.mockResolvedValueOnce({ rows: mockProducts });

      const res = await request(app).get('/api/products');
      expect(res.statusCode).toBe(200);
      expect(Array.isArray(res.body)).toBe(true);
    });
  });

  describe('POST /api/products', () => {
    it('should create a product', async () => {
      const newProduct = { name: 'New Widget', price: 29.99, stock: 10 };
      mockPool.query.mockResolvedValueOnce({
        rows: [{ id: 3, ...newProduct }],
      });

      const res = await request(app)
        .post('/api/products')
        .send(newProduct);
      expect(res.statusCode).toBe(201);
    });

    it('should return 400 if name is missing', async () => {
      const res = await request(app)
        .post('/api/products')
        .send({ price: 9.99 });
      expect(res.statusCode).toBe(400);
    });
  });
});
```

---

## Step 2: Dockerize All Services

### Product Service Dockerfile

```dockerfile
# product-service/Dockerfile
# Stage 1: Build dependencies
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci --only=production

# Stage 2: Production image
FROM node:20-alpine
WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 appgroup && \
    adduser -u 1001 -G appgroup -s /bin/sh -D appuser

# Copy dependencies from builder
COPY --from=builder /app/node_modules ./node_modules
COPY src/ ./src/
COPY package.json ./

# Set ownership
RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 3001

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3001/health || exit 1

CMD ["node", "src/index.js"]
```

### Order Service Dockerfile

```dockerfile
# order-service/Dockerfile
# Stage 1: Build
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# Stage 2: Production
FROM python:3.12-slim
WORKDIR /app

# Create non-root user
RUN groupadd -g 1001 appgroup && \
    useradd -u 1001 -g appgroup -m appuser

# Install runtime dependencies only
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 curl && \
    rm -rf /var/lib/apt/lists/*

# Copy installed packages from builder
COPY --from=builder /install /usr/local
COPY src/ ./src/

RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 3002

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3002/health || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:3002", "--workers", "4", "--timeout", "120", "src.app:app"]
```

### Docker Compose for Local Development

```yaml
# docker-compose.yaml
version: '3.8'

services:
  # --- Databases ---
  products-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: products_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - products-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  orders-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: orders_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5433:5432"
    volumes:
      - orders-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # --- Cache ---
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # --- Application Services ---
  product-service:
    build:
      context: ./product-service
      dockerfile: Dockerfile
    ports:
      - "3001:3001"
    environment:
      PORT: 3001
      DB_HOST: products-db
      DB_PORT: 5432
      DB_NAME: products_db
      DB_USER: postgres
      DB_PASSWORD: postgres
      REDIS_HOST: redis
      REDIS_PORT: 6379
    depends_on:
      products-db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3001/health"]
      interval: 15s
      timeout: 5s
      retries: 3

  order-service:
    build:
      context: ./order-service
      dockerfile: Dockerfile
    ports:
      - "3002:3002"
    environment:
      PORT: 3002
      DB_HOST: orders-db
      DB_PORT: 5432
      DB_NAME: orders_db
      DB_USER: postgres
      DB_PASSWORD: postgres
      PRODUCT_SERVICE_URL: http://product-service:3001
    depends_on:
      orders-db:
        condition: service_healthy
      product-service:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3002/health"]
      interval: 15s
      timeout: 5s
      retries: 3

volumes:
  products-db-data:
  orders-db-data:
```

```bash
# Build and run all services locally
docker compose up --build -d

# Verify all services are healthy
docker compose ps

# Test the services
curl http://localhost:3001/health
curl http://localhost:3002/health

# Create a product
curl -X POST http://localhost:3001/api/products \
  -H "Content-Type: application/json" \
  -d '{"name": "Test Widget", "price": 19.99, "stock": 100}'

# Create an order
curl -X POST http://localhost:3002/api/orders \
  -H "Content-Type: application/json" \
  -d '{"product_id": 1, "quantity": 2, "customer_email": "test@example.com"}'

# View logs
docker compose logs -f product-service
docker compose logs -f order-service

# Tear down
docker compose down -v
```

---

## Step 3: Git Repository & Branching Strategy

### Repository Structure

```
devops-capstone/
  product-service/
    src/
    Dockerfile
    package.json
  order-service/
    src/
    Dockerfile
    requirements.txt
  k8s/
    base/
      product-service/
      order-service/
      redis/
      kustomization.yaml
    overlays/
      dev/
      staging/
      production/
  helm/
    capstone-app/
      Chart.yaml
      values.yaml
      values-dev.yaml
      values-staging.yaml
      values-production.yaml
      templates/
  terraform/
    environments/
      dev/
      staging/
      production/
    modules/
      vpc/
      eks/
  .github/
    workflows/
      ci.yaml
      cd-dev.yaml
      cd-staging.yaml
      cd-production.yaml
  docker-compose.yaml
  README.md
```

### Git Branching Strategy (GitHub Flow)

```
main (protected)
  |
  +-- feature/add-product-search     (feature branch)
  |     |
  |     +-- PR -> main (after review and CI passes)
  |
  +-- feature/order-notifications    (feature branch)
  |     |
  |     +-- PR -> main (after review and CI passes)
  |
  +-- fix/product-cache-invalidation (bugfix branch)
        |
        +-- PR -> main (after review and CI passes)

Branch naming conventions:
  feature/*   - New features
  fix/*       - Bug fixes
  hotfix/*    - Emergency production fixes
  chore/*     - Maintenance tasks (dependency updates, refactoring)

Rules:
  - main branch is always deployable
  - All changes go through pull requests
  - CI must pass before merge
  - At least 1 code review required
  - Squash merge to keep history clean
```

### Git Setup Commands

```bash
# Initialize the repository
git init devops-capstone
cd devops-capstone

# Create .gitignore
cat > .gitignore << 'EOF'
# Node
node_modules/
coverage/
.env

# Python
__pycache__/
*.pyc
.venv/
*.egg-info/

# Terraform
.terraform/
*.tfstate
*.tfstate.backup
*.tfvars
!*.tfvars.example

# Docker
.docker/

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db
EOF

# Create initial commit
git add .
git commit -m "Initial project structure"

# Create and push to GitHub
gh repo create devops-capstone --public --source=. --push

# Protect main branch (requires GitHub CLI)
gh api repos/{owner}/devops-capstone/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["ci"]}' \
  --field enforce_admins=true \
  --field required_pull_request_reviews='{"required_approving_review_count":1}'
```

---

## Step 4: CI/CD Pipeline with GitHub Actions

### CI Pipeline (Runs on Every Pull Request)

```yaml
# .github/workflows/ci.yaml
name: CI Pipeline

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  PRODUCT_IMAGE: ghcr.io/${{ github.repository }}/product-service
  ORDER_IMAGE: ghcr.io/${{ github.repository }}/order-service

jobs:
  # --- Lint and Test Product Service ---
  test-product-service:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: product-service
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: product-service/package-lock.json

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests
        run: npm test

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: product-coverage
          path: product-service/coverage/

  # --- Lint and Test Order Service ---
  test-order-service:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: order-service
    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
          cache-dependency-path: order-service/requirements.txt

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run linter
        run: |
          pip install flake8
          flake8 src/ --max-line-length=120

      - name: Run tests
        run: pytest src/tests/ -v --tb=short

  # --- Build and Push Docker Images ---
  build-images:
    needs: [test-product-service, test-order-service]
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write
    outputs:
      image_tag: ${{ steps.meta.outputs.version }}
    steps:
      - uses: actions/checkout@v4

      - name: Set image tag
        id: meta
        run: |
          TAG="v1.0.${{ github.run_number }}-$(echo ${{ github.sha }} | cut -c1-7)"
          echo "version=$TAG" >> "$GITHUB_OUTPUT"
          echo "Image tag: $TAG"

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Product Service
        uses: docker/build-push-action@v5
        with:
          context: ./product-service
          push: true
          tags: |
            ${{ env.PRODUCT_IMAGE }}:${{ steps.meta.outputs.version }}
            ${{ env.PRODUCT_IMAGE }}:latest

      - name: Build and push Order Service
        uses: docker/build-push-action@v5
        with:
          context: ./order-service
          push: true
          tags: |
            ${{ env.ORDER_IMAGE }}:${{ steps.meta.outputs.version }}
            ${{ env.ORDER_IMAGE }}:latest

  # --- Security Scanning ---
  security-scan:
    needs: build-images
    runs-on: ubuntu-latest
    steps:
      - name: Run Trivy on Product Service
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: '${{ env.PRODUCT_IMAGE }}:latest'
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

      - name: Run Trivy on Order Service
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: '${{ env.ORDER_IMAGE }}:latest'
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

  # --- Update Kubernetes Manifests for Dev ---
  deploy-dev:
    needs: [build-images, security-scan]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Update dev image tags
        run: |
          cd k8s/overlays/dev
          kustomize edit set image \
            product-service=${{ env.PRODUCT_IMAGE }}:${{ needs.build-images.outputs.image_tag }}
          kustomize edit set image \
            order-service=${{ env.ORDER_IMAGE }}:${{ needs.build-images.outputs.image_tag }}

      - name: Commit and push
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add k8s/overlays/dev/
          git commit -m "deploy(dev): update images to ${{ needs.build-images.outputs.image_tag }}" || true
          git push
```

---

## Step 5: Terraform Infrastructure

### VPC Module

```hcl
# terraform/modules/vpc/main.tf
variable "project_name" {
  type    = string
  default = "capstone"
}

variable "environment" {
  type = string
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.project_name}-${var.environment}-vpc"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# Public Subnets (for load balancers and NAT gateways)
resource "aws_subnet" "public" {
  count                   = length(var.availability_zones)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name                                           = "${var.project_name}-${var.environment}-public-${count.index}"
    "kubernetes.io/role/elb"                        = "1"
    "kubernetes.io/cluster/${var.project_name}-${var.environment}" = "shared"
  }
}

# Private Subnets (for EKS worker nodes)
resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 100)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name                                           = "${var.project_name}-${var.environment}-private-${count.index}"
    "kubernetes.io/role/internal-elb"               = "1"
    "kubernetes.io/cluster/${var.project_name}-${var.environment}" = "shared"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "${var.project_name}-${var.environment}-igw"
  }
}

# NAT Gateway (for private subnet internet access)
resource "aws_eip" "nat" {
  domain = "vpc"
  tags = {
    Name = "${var.project_name}-${var.environment}-nat-eip"
  }
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id
  tags = {
    Name = "${var.project_name}-${var.environment}-nat"
  }
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  tags = { Name = "${var.project_name}-${var.environment}-public-rt" }
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }
  tags = { Name = "${var.project_name}-${var.environment}-private-rt" }
}

resource "aws_route_table_association" "public" {
  count          = length(var.availability_zones)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = length(var.availability_zones)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}

# Outputs
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}
```

### EKS Module

```hcl
# terraform/modules/eks/main.tf
variable "project_name" { type = string }
variable "environment" { type = string }
variable "vpc_id" { type = string }
variable "private_subnet_ids" { type = list(string) }
variable "kubernetes_version" {
  type    = string
  default = "1.28"
}
variable "node_instance_types" {
  type    = list(string)
  default = ["t3.medium"]
}
variable "node_desired_size" {
  type    = number
  default = 2
}
variable "node_min_size" {
  type    = number
  default = 1
}
variable "node_max_size" {
  type    = number
  default = 5
}

# EKS Cluster IAM Role
resource "aws_iam_role" "eks_cluster" {
  name = "${var.project_name}-${var.environment}-eks-cluster-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "eks.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_cluster.name
}

# EKS Cluster
resource "aws_eks_cluster" "main" {
  name     = "${var.project_name}-${var.environment}"
  role_arn = aws_iam_role.eks_cluster.arn
  version  = var.kubernetes_version

  vpc_config {
    subnet_ids              = var.private_subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = true
  }

  depends_on = [aws_iam_role_policy_attachment.eks_cluster_policy]

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# Node Group IAM Role
resource "aws_iam_role" "eks_nodes" {
  name = "${var.project_name}-${var.environment}-eks-node-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "eks_worker_node_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.eks_nodes.name
}

resource "aws_iam_role_policy_attachment" "eks_cni_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.eks_nodes.name
}

resource "aws_iam_role_policy_attachment" "ecr_read_only" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.eks_nodes.name
}

# Managed Node Group
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.project_name}-${var.environment}-nodes"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = var.private_subnet_ids
  instance_types  = var.node_instance_types

  scaling_config {
    desired_size = var.node_desired_size
    min_size     = var.node_min_size
    max_size     = var.node_max_size
  }

  update_config {
    max_unavailable = 1
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_worker_node_policy,
    aws_iam_role_policy_attachment.eks_cni_policy,
    aws_iam_role_policy_attachment.ecr_read_only,
  ]

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

output "cluster_name" {
  value = aws_eks_cluster.main.name
}

output "cluster_endpoint" {
  value = aws_eks_cluster.main.endpoint
}

output "cluster_ca_certificate" {
  value = aws_eks_cluster.main.certificate_authority[0].data
}
```

### Dev Environment Configuration

```hcl
# terraform/environments/dev/main.tf
terraform {
  required_version = ">= 1.7.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.30"
    }
  }

  backend "s3" {
    bucket         = "capstone-terraform-state"
    key            = "dev/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      Project     = "capstone"
      Environment = "dev"
      ManagedBy   = "terraform"
    }
  }
}

module "vpc" {
  source       = "../../modules/vpc"
  project_name = "capstone"
  environment  = "dev"
  vpc_cidr     = "10.0.0.0/16"
}

module "eks" {
  source              = "../../modules/eks"
  project_name        = "capstone"
  environment         = "dev"
  vpc_id              = module.vpc.vpc_id
  private_subnet_ids  = module.vpc.private_subnet_ids
  node_instance_types = ["t3.medium"]
  node_desired_size   = 2
  node_min_size       = 1
  node_max_size       = 3
}

output "cluster_name" {
  value = module.eks.cluster_name
}

output "vpc_id" {
  value = module.vpc.vpc_id
}
```

```bash
# Deploy infrastructure
cd terraform/environments/dev

terraform init
terraform plan -out=tfplan
terraform apply tfplan

# Configure kubectl
aws eks update-kubeconfig --name capstone-dev --region us-east-1
kubectl get nodes
```

---

## Step 6: Kubernetes Manifests & Helm Charts

### Helm Chart Structure

```yaml
# helm/capstone-app/Chart.yaml
apiVersion: v2
name: capstone-app
description: DevOps Capstone E-commerce Application
type: application
version: 0.1.0
appVersion: "1.0.0"
dependencies:
  - name: postgresql
    version: "14.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    alias: productsDb
    condition: productsDb.enabled
  - name: postgresql
    version: "14.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    alias: ordersDb
    condition: ordersDb.enabled
  - name: redis
    version: "18.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

```yaml
# helm/capstone-app/values.yaml (default values)
# Product Service
productService:
  replicaCount: 2
  image:
    repository: ghcr.io/your-org/devops-capstone/product-service
    tag: latest
    pullPolicy: IfNotPresent
  service:
    type: ClusterIP
    port: 80
    targetPort: 3001
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 250m
      memory: 256Mi
  env:
    DB_HOST: "{{ .Release.Name }}-productsdb-postgresql"
    DB_PORT: "5432"
    DB_NAME: "products_db"
    DB_USER: "postgres"
    REDIS_HOST: "{{ .Release.Name }}-redis-master"
    REDIS_PORT: "6379"

# Order Service
orderService:
  replicaCount: 2
  image:
    repository: ghcr.io/your-org/devops-capstone/order-service
    tag: latest
    pullPolicy: IfNotPresent
  service:
    type: ClusterIP
    port: 80
    targetPort: 3002
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 250m
      memory: 256Mi

# Ingress
ingress:
  enabled: true
  className: nginx
  host: capstone.local

# Databases
productsDb:
  enabled: true
  auth:
    postgresPassword: postgres
    database: products_db

ordersDb:
  enabled: true
  auth:
    postgresPassword: postgres
    database: orders_db

# Redis
redis:
  enabled: true
  auth:
    enabled: false
```

### Helm Deployment Template

```yaml
# helm/capstone-app/templates/product-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "capstone-app.fullname" . }}-product-service
  labels:
    app.kubernetes.io/component: product-service
    {{- include "capstone-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.productService.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/component: product-service
      {{- include "capstone-app.selectorLabels" . | nindent 6 }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app.kubernetes.io/component: product-service
        {{- include "capstone-app.selectorLabels" . | nindent 8 }}
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3001"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: product-service
          image: "{{ .Values.productService.image.repository }}:{{ .Values.productService.image.tag }}"
          imagePullPolicy: {{ .Values.productService.image.pullPolicy }}
          ports:
            - containerPort: 3001
              protocol: TCP
          env:
            - name: PORT
              value: "3001"
            - name: DB_HOST
              value: "{{ .Release.Name }}-productsdb-postgresql"
            - name: DB_PORT
              value: "5432"
            - name: DB_NAME
              value: "products_db"
            - name: DB_USER
              value: "postgres"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Release.Name }}-productsdb-postgresql
                  key: postgres-password
            - name: REDIS_HOST
              value: "{{ .Release.Name }}-redis-master"
            - name: REDIS_PORT
              value: "6379"
          readinessProbe:
            httpGet:
              path: /health
              port: 3001
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 3001
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          resources:
            {{- toYaml .Values.productService.resources | nindent 12 }}
```

```yaml
# helm/capstone-app/templates/ingress.yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "capstone-app.fullname" . }}
  labels:
    {{- include "capstone-app.labels" . | nindent 4 }}
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: {{ .Values.ingress.className }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /api/products(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: {{ include "capstone-app.fullname" . }}-product-service
                port:
                  number: 80
          - path: /api/orders(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: {{ include "capstone-app.fullname" . }}-order-service
                port:
                  number: 80
{{- end }}
```

```bash
# Install the chart locally
helm dependency update helm/capstone-app/
helm install capstone helm/capstone-app/ \
  --namespace capstone-dev \
  --create-namespace \
  -f helm/capstone-app/values-dev.yaml

# Verify deployment
kubectl get all -n capstone-dev
helm status capstone -n capstone-dev

# Upgrade with new values
helm upgrade capstone helm/capstone-app/ \
  --namespace capstone-dev \
  -f helm/capstone-app/values-dev.yaml

# Rollback if needed
helm rollback capstone 1 -n capstone-dev
```

---

## Step 7: ArgoCD GitOps Setup

### ArgoCD Application for Each Environment

```yaml
# argocd/dev-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: capstone-dev
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/devops-capstone.git
    targetRevision: main
    path: helm/capstone-app
    helm:
      releaseName: capstone
      valueFiles:
        - values.yaml
        - values-dev.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: capstone-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

```yaml
# argocd/production-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: capstone-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/devops-capstone.git
    targetRevision: main
    path: helm/capstone-app
    helm:
      releaseName: capstone
      valueFiles:
        - values.yaml
        - values-production.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: capstone-production
  # Production: NO automated sync -- manual approval required
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=120s

# Get initial admin password
ARGOCD_PASS=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD admin password: $ARGOCD_PASS"

# Apply applications
kubectl apply -f argocd/dev-application.yaml
kubectl apply -f argocd/production-application.yaml

# Access dashboard
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
echo "ArgoCD dashboard: https://localhost:8080"
```

---

## Step 8: Monitoring with Prometheus & Grafana

### Install Prometheus Stack via Helm

```bash
# Add Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install with custom values
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f monitoring-values.yaml
```

```yaml
# monitoring-values.yaml
prometheus:
  prometheusSpec:
    retention: 15d
    resources:
      requests:
        memory: 512Mi
        cpu: 250m
      limits:
        memory: 1Gi
        cpu: 500m
    # Scrape application metrics
    additionalScrapeConfigs:
      - job_name: 'product-service'
        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names: ['capstone-dev', 'capstone-staging', 'capstone-production']
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port, __meta_kubernetes_pod_ip]
            action: replace
            target_label: __address__
            regex: (.+);(.+)
            replacement: $2:$1

grafana:
  enabled: true
  adminPassword: admin123
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: 'custom'
          orgId: 1
          folder: 'Capstone'
          type: file
          disableDeletion: false
          editable: true
          options:
            path: /var/lib/grafana/dashboards/custom

alertmanager:
  config:
    global:
      resolve_timeout: 5m
    route:
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: 'slack-notifications'
      routes:
        - match:
            severity: critical
          receiver: 'slack-critical'
    receivers:
      - name: 'slack-notifications'
        slack_configs:
          - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
            channel: '#alerts'
            title: '{{ .GroupLabels.alertname }}'
            text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
      - name: 'slack-critical'
        slack_configs:
          - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
            channel: '#alerts-critical'
```

### Custom Alerting Rules

```yaml
# alerting-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: capstone-alerts
  namespace: monitoring
spec:
  groups:
    - name: capstone.application
      rules:
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{status_code=~"5..", namespace=~"capstone-.*"}[5m]))
            /
            sum(rate(http_requests_total{namespace=~"capstone-.*"}[5m]))
            > 0.05
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "High error rate in {{ $labels.namespace }}"
            description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"

        - alert: HighLatency
          expr: |
            histogram_quantile(0.95,
              sum(rate(http_request_duration_seconds_bucket{namespace=~"capstone-.*"}[5m])) by (le, namespace)
            ) > 0.5
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High p95 latency in {{ $labels.namespace }}"
            description: "P95 latency is {{ $value }}s (threshold: 0.5s)"

        - alert: PodCrashLooping
          expr: |
            rate(kube_pod_container_status_restarts_total{namespace=~"capstone-.*"}[15m]) > 0.1
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Pod {{ $labels.pod }} is crash-looping"

        - alert: PodNotReady
          expr: |
            kube_pod_status_ready{condition="true", namespace=~"capstone-.*"} == 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} is not ready"
```

```bash
# Access Grafana
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80 &
echo "Grafana: http://localhost:3000 (admin/admin123)"

# Access Prometheus
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090 &
echo "Prometheus: http://localhost:9090"

# Verify metrics are being scraped
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {scrapeUrl, health}'
```

---

## Step 9: Security Scanning in Pipeline

### Trivy Container Scanning

```yaml
# .github/workflows/security-scan.yaml
name: Security Scan

on:
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'    # Weekly scan on Mondays at 6 AM

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Scan Dockerfiles for misconfigurations
      - name: Scan Dockerfiles
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'config'
          scan-ref: '.'
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

      # Scan for secrets in code
      - name: Scan for secrets
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          scanners: 'secret'
          format: 'table'
          exit-code: '1'

      # Scan Kubernetes manifests
      - name: Scan K8s manifests
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'config'
          scan-ref: './k8s'
          format: 'table'
          exit-code: '0'           # Warning only for K8s misconfigs
          severity: 'CRITICAL,HIGH'

      # Scan Terraform files
      - name: Scan Terraform
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'config'
          scan-ref: './terraform'
          format: 'table'
          exit-code: '0'
          severity: 'CRITICAL,HIGH'

  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Scan Node.js dependencies
      - name: Scan Product Service dependencies
        working-directory: product-service
        run: |
          npm audit --production --audit-level=high

      # Scan Python dependencies
      - name: Scan Order Service dependencies
        run: |
          pip install safety
          safety check -r order-service/requirements.txt
```

### OPA/Gatekeeper Policy Example

```yaml
# k8s/policies/require-resource-limits.yaml
# Ensures all containers have resource limits defined
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresources
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResources
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresources

        violation[{"msg": msg}] {
          container := input.review.object.spec.template.spec.containers[_]
          not container.resources.limits
          msg := sprintf("Container '%v' must have resource limits defined", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.template.spec.containers[_]
          not container.resources.requests
          msg := sprintf("Container '%v' must have resource requests defined", [container.name])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: require-resource-limits
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
    namespaces:
      - capstone-dev
      - capstone-staging
      - capstone-production
```

---

## Step 10: Documentation Requirements

Every capstone project must include the following documentation.

### Required Documents

```
1. README.md
   - Project overview and architecture diagram
   - Technology stack with versions
   - Local development setup instructions (docker compose up)
   - How to deploy to each environment
   - API documentation (endpoints, request/response examples)
   - Troubleshooting guide

2. RUNBOOK.md
   - Common failure scenarios and how to resolve them
   - How to access logs and metrics
   - How to rollback a deployment
   - On-call escalation contacts
   - Database backup and restore procedures

3. ARCHITECTURE.md
   - System architecture diagram
   - Service communication patterns
   - Data flow diagrams
   - Infrastructure diagram (AWS resources)
   - Security architecture (network policies, RBAC)

4. ADR/ (Architecture Decision Records)
   - ADR-001: Why microservices over monolith
   - ADR-002: Database per service pattern
   - ADR-003: GitOps with ArgoCD over FluxCD
   - ADR-004: Helm over raw Kustomize
```

### ADR Template

```markdown
# ADR-001: Microservices Architecture

## Status
Accepted

## Context
We need to decide between a monolithic application and a microservices
architecture for the capstone e-commerce platform.

## Decision
We will use a microservices architecture with three services:
Product Service, Order Service, and API Gateway.

## Consequences
Positive:
- Independent deployment and scaling of services
- Technology flexibility (Node.js for products, Python for orders)
- Team autonomy (different teams can own different services)

Negative:
- Increased operational complexity
- Service-to-service communication latency
- Distributed debugging is harder
- Need for service discovery and load balancing

## Alternatives Considered
1. Monolith: Simpler to start, but harder to scale and deploy
2. Serverless: Lower ops overhead, but vendor lock-in concerns
```

---

## Evaluation Rubric

### Grading Criteria

```
Category                    | Points | Criteria
----------------------------|--------|------------------------------------------
Application Code            | 15     | Clean, tested, working microservices
                            |        | - Unit tests with >70% coverage
                            |        | - Health check endpoints
                            |        | - Prometheus metrics instrumented

Containerization            | 10     | Multi-stage Dockerfiles
                            |        | - Non-root user
                            |        | - Health checks in Dockerfile
                            |        | - docker-compose works locally

CI/CD Pipeline              | 20     | Complete GitHub Actions pipeline
                            |        | - Lint, test, build, scan, deploy stages
                            |        | - Automated image tagging
                            |        | - Environment promotion (dev -> staging)

Infrastructure (Terraform)  | 15     | VPC + EKS provisioned via Terraform
                            |        | - Modular structure
                            |        | - Remote state backend
                            |        | - Environment separation

Kubernetes + Helm           | 15     | Working Helm chart with env-specific values
                            |        | - Resource limits defined
                            |        | - Readiness/liveness probes
                            |        | - Ingress configured

GitOps (ArgoCD)             | 10     | ArgoCD managing deployments
                            |        | - Auto-sync for dev
                            |        | - Manual sync for production
                            |        | - Self-healing enabled

Monitoring                  | 10     | Prometheus + Grafana operational
                            |        | - Application metrics dashboard
                            |        | - At least 2 custom alerting rules
                            |        | - SLO tracking visible

Security                    | 5      | Trivy scanning in pipeline
                            |        | - No critical vulnerabilities
                            |        | - Secrets not hardcoded

Total                       | 100    |
```

### Bonus Points

```
Bonus Category              | Points | Criteria
----------------------------|--------|------------------------------------------
Canary Deployment           | +5     | Working canary with NGINX Ingress weights
Chaos Experiment            | +5     | At least one Litmus chaos experiment run
Load Testing                | +5     | k6 load test with SLO thresholds
Complete Documentation      | +5     | All four documents with ADRs
HPA Auto-scaling            | +3     | HPA configured and tested
Network Policies            | +2     | K8s network policies restricting traffic
```

---

## Projects

### Project: Complete Capstone Implementation

```
This is the single capstone project. Follow all 10 steps above to
build the complete pipeline. Here is the recommended order and
timeline for a 2-week sprint:

Week 1:
  Day 1-2: Steps 1-2 (Application code + Docker)
    - Get both services running locally with docker compose
    - Write and run unit tests
    - Verify health endpoints and metrics

  Day 3-4: Steps 3-4 (Git + CI/CD)
    - Set up repository with branching strategy
    - Create CI pipeline that runs on every PR
    - Verify lint, test, build, and scan stages pass

  Day 5: Step 5 (Terraform)
    - Write VPC and EKS modules
    - Deploy dev environment
    - Configure kubectl to connect to the cluster

Week 2:
  Day 6-7: Steps 6-7 (Helm + ArgoCD)
    - Create Helm chart with environment-specific values
    - Install ArgoCD and create Application CRDs
    - Verify GitOps workflow (push to Git -> auto-deploy)

  Day 8-9: Steps 8-9 (Monitoring + Security)
    - Install Prometheus and Grafana
    - Create dashboards and alerting rules
    - Add security scanning to CI pipeline

  Day 10: Step 10 (Documentation + Polish)
    - Write all required documentation
    - Test the full pipeline end-to-end
    - Fix any issues found

Verification Checklist:
  [ ] docker compose up works locally
  [ ] CI pipeline passes on a test PR
  [ ] Terraform provisions infrastructure without errors
  [ ] Helm installs the app on Kubernetes
  [ ] ArgoCD shows the app as Synced and Healthy
  [ ] Grafana dashboard shows application metrics
  [ ] Trivy scan runs in the pipeline
  [ ] All documentation is complete

Submit:
  - GitHub repository URL
  - Screenshots of: ArgoCD dashboard, Grafana dashboard, CI pipeline
  - Brief write-up: what you learned, what was hardest, what you would
    improve with more time
```

---

## Summary

This capstone project integrates all major DevOps concepts from the course:

| Step | Topic                  | Tools Used                          |
|------|------------------------|-------------------------------------|
| 1    | Application Code       | Node.js, Python, Express, Flask     |
| 2    | Containerization       | Docker, docker-compose              |
| 3    | Version Control        | Git, GitHub, branching strategy     |
| 4    | CI/CD                  | GitHub Actions                      |
| 5    | Infrastructure as Code | Terraform, AWS (VPC, EKS)           |
| 6    | Container Orchestration| Kubernetes, Helm                    |
| 7    | GitOps                 | ArgoCD                              |
| 8    | Monitoring & Alerting  | Prometheus, Grafana, Alertmanager   |
| 9    | Security               | Trivy, OPA/Gatekeeper               |
| 10   | Documentation          | README, Runbooks, ADRs              |

**Key Takeaways:**
- A complete DevOps pipeline has many moving parts that must work together
- Start simple and iterate (get docker-compose working before Kubernetes)
- Automation compounds over time: invest early in CI/CD and GitOps
- Monitoring is not optional: you cannot manage what you cannot measure
- Security scanning must be continuous, not a one-time check
- Documentation is part of the deliverable, not an afterthought
