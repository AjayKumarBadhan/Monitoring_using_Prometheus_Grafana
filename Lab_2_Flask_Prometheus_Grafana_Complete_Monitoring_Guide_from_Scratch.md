# Flask + Prometheus + Grafana — Complete Monitoring Guide from Scratch

> A fully detailed, beginner-friendly guide covering every file, every line of code, and every configuration needed to build a production-style monitoring stack using Flask, Prometheus, and Grafana inside Docker.

---

## Table of Contents

1. [What Are We Building?](#1-what-are-we-building)
2. [Architecture Overview](#2-architecture-overview)
3. [Project Folder Structure](#3-project-folder-structure)
4. [Step 1 — Project Dependencies (`requirements.txt`)](#4-step-1--project-dependencies-requirementstxt)
5. [Step 2 — Frontend Flask App (`frontend/app.py`)](#5-step-2--frontend-flask-app-frontendapppy)
6. [Step 3 — Backend Flask App (`backend/app.py`)](#6-step-3--backend-flask-app-backendapppy)
7. [Step 4 — Dockerfile (Frontend & Backend)](#7-step-4--dockerfile-frontend--backend)
8. [Step 5 — Prometheus Configuration (`prometheus.yml`)](#8-step-5--prometheus-configuration-prometheusyml)
9. [Step 6 — Docker Compose (`docker-compose.yml`)](#9-step-6--docker-compose-docker-composeyml)
10. [Step 7 — Running the Stack](#10-step-7--running-the-stack)
11. [Step 8 — Building the Grafana Dashboard](#11-step-8--building-the-grafana-dashboard)
12. [Understanding the IDE Warning](#12-understanding-the-ide-warning)
13. [Understanding the `/metrics` Internal Server Error](#13-understanding-the-metrics-internal-server-error)
14. [Common Errors and Fixes](#14-common-errors-and-fixes)
15. [Key Concepts Reference](#15-key-concepts-reference)

---

## 1. What Are We Building?

We are building a **microservices monitoring stack** — a system where:

- A **Flask frontend** serves web pages to users
- A **Flask backend** handles API/business logic
- **Prometheus** automatically collects performance data (metrics) from both services
- **Grafana** displays that data as live, interactive dashboards

This is the exact pattern used by companies like Netflix, Uber, and Google to monitor thousands of services in real time.

| Tool | Role | Port |
|---|---|---|
| **Flask Frontend** | Serves the web UI | `5000` |
| **Flask Backend** | Handles API logic | `5001` |
| **Prometheus** | Collects and stores metrics | `9090` |
| **Grafana** | Visualizes metrics as charts | `3000` |
| **Docker Compose** | Runs all services together | — |

---

## 2. Architecture Overview

### High-Level Data Flow

```
 User Browser
      │
      ▼
┌─────────────────┐         ┌──────────────────────┐
│  Flask Frontend │────────▶│  Flask Backend       │
│  (port 5000)    │ HTTP API│  (port 5001)         │
│                 │         │                      │
│  /metrics  ◀───┼─────────┼─── /metrics          │
└────────┬────────┘         └──────────┬───────────┘
         │  exposes metrics             │  exposes metrics
         ▼                             ▼
┌────────────────────────────────────────────────────┐
│                    Prometheus                      │
│                    (port 9090)                     │
│                                                    │
│  Scrapes /metrics from both services every 15s     │
│  Stores time-series data internally                │
└──────────────────────┬─────────────────────────────┘
                       │  Grafana queries Prometheus
                       ▼
┌────────────────────────────────────────────────────┐
│                     Grafana                        │
│                    (port 3000)                     │
│                                                    │
│  Runs PromQL queries → Renders charts/dashboards   │
└────────────────────────────────────────────────────┘
```

### Docker Network Diagram

```
┌──────────────────────────────────────────────────────────┐
│                  Docker Internal Network                 │
│                                                          │
│  ┌──────────────┐    ┌──────────────┐                   │
│  │  frontend    │    │  backend     │                   │
│  │  container   │───▶│  container   │                   │
│  │  port: 5000  │    │  port: 5001  │                   │
│  └──────┬───────┘    └──────┬───────┘                   │
│         │                   │                            │
│         └─────────┬─────────┘                            │
│                   ▼                                      │
│          ┌────────────────┐                              │
│          │  prometheus    │                              │
│          │  container     │                              │
│          │  port: 9090    │                              │
│          └───────┬────────┘                              │
│                  │                                       │
│          ┌───────▼────────┐                              │
│          │  grafana       │                              │
│          │  container     │                              │
│          │  port: 3000    │                              │
│          └────────────────┘                              │
│                                                          │
└──────────────────────────────────────────────────────────┘
         ▲                ▲               ▲              ▲
    localhost:5000   localhost:5001  localhost:9090  localhost:3000
         (your browser can access all of these)
```

> **Important:** Inside Docker, containers talk to each other using their **service names** (e.g., `http://backend:5001`), not `localhost`. But from your browser, you use `localhost` with the mapped port.

---

## 3. Project Folder Structure

Your complete project should look like this:

```
my-monitoring-project/
│
├── frontend/
│   ├── app.py               ← Flask frontend application
│   ├── Dockerfile           ← Instructions to build the frontend container
│   └── requirements.txt     ← Python dependencies for frontend
│
├── backend/
│   ├── app.py               ← Flask backend application
│   ├── Dockerfile           ← Instructions to build the backend container
│   └── requirements.txt     ← Python dependencies for backend
│
├── prometheus/
│   └── prometheus.yml       ← Prometheus scrape configuration
│
└── docker-compose.yml       ← Ties all services together
```

> Create this structure before writing any code. Open your terminal and run:
> ```bash
> mkdir -p my-monitoring-project/frontend my-monitoring-project/backend my-monitoring-project/prometheus
> cd my-monitoring-project
> ```

---

## 4. Step 1 — Project Dependencies (`requirements.txt`)

Both the frontend and backend need the same Python packages. Create this file in **both** `frontend/` and `backend/` directories.

### `frontend/requirements.txt` and `backend/requirements.txt`

```
flask
requests
prometheus_client
```

### Line-by-Line Explanation

| Package | What it does | Why we need it |
|---|---|---|
| `flask` | The web framework that runs our app and handles HTTP routes | Core application server |
| `requests` | Allows Python to make HTTP calls to other services | Frontend uses this to call the backend API |
| `prometheus_client` | Official Python library to create, register, and expose Prometheus metrics | Powers all our `/metrics` endpoints |

### Why a Separate `requirements.txt` Instead of Installing in Dockerfile?

Using a `requirements.txt` file is best practice for three reasons:

1. **Reproducibility** — Anyone who clones your project installs the exact same packages
2. **IDE support** — Editors like VS Code read `requirements.txt` to resolve imports locally, eliminating yellow-underline warnings
3. **Separation of concerns** — The `Dockerfile` handles container setup; `requirements.txt` handles app dependencies

---

## 5. Step 2 — Frontend Flask App (`frontend/app.py`)

This is the main application file for the frontend service. It serves web pages, tracks metrics, and exposes those metrics to Prometheus.

### Complete Code

```python
# ============================================================
# frontend/app.py  — Flask Frontend with Prometheus Metrics
# ============================================================

# --- Section 1: Imports ---

import time                        # Used to measure how long each request takes
import random                      # Used to simulate variable latency (for demo purposes)
import requests                    # Used to call the backend API from the frontend

from flask import Flask, Response, jsonify   # Flask core: app factory, HTTP response, JSON helper

from prometheus_client import (
    Counter,           # Metric that only goes UP (e.g., total requests)
    Histogram,         # Metric that tracks distribution (e.g., request durations)
    generate_latest,   # Serializes all metrics into Prometheus text format
    CONTENT_TYPE_LATEST  # The correct MIME type: 'text/plain; version=0.0.4'
)

# --- Section 2: App Initialization ---

app = Flask(__name__)
# Flask(__name__) creates a new Flask application.
# __name__ tells Flask the name of the current module,
# which it uses to locate templates, static files, etc.

# --- Section 3: Define Prometheus Metrics ---

# COUNTER: Counts every incoming request to the frontend.
# It only ever increases — it never resets or decreases.
# Labels allow us to filter by HTTP method (GET/POST) and endpoint (/, /about, etc.)
REQUEST_COUNT = Counter(
    'frontend_requests_total',            # Metric name (shows up in Prometheus)
    'Total number of frontend requests',  # Human-readable description
    ['method', 'endpoint', 'status']      # Label dimensions for filtering
)

# HISTOGRAM: Measures how long each request takes to complete.
# Prometheus Histograms automatically create "buckets":
#   - How many requests took < 0.01s?
#   - How many took < 0.05s?
#   - How many took < 0.1s? etc.
# This lets us calculate percentiles (p50, p95, p99) later in Grafana.
REQUEST_LATENCY = Histogram(
    'frontend_latency_seconds',             # Metric name
    'Frontend request latency in seconds',  # Description
    ['method', 'endpoint'],                 # Labels
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5]  # Custom bucket boundaries in seconds
)

# --- Section 4: Application Routes ---

@app.route('/')
def index():
    """
    Homepage route.
    Every time a user visits '/', we:
      1. Record the start time
      2. Do some work (simulate processing with a small random delay)
      3. Record the end time
      4. Increment our counter and histogram with the result
    """
    start_time = time.time()   # Capture timestamp before processing begins

    # Simulate a small amount of work with random latency (0.01 to 0.3 seconds).
    # In a real app, this would be database queries, file reads, API calls, etc.
    time.sleep(random.uniform(0.01, 0.3))

    response_data = {
        "service": "frontend",
        "status": "running",
        "message": "Welcome to the Frontend Service!"
    }

    duration = time.time() - start_time   # How long the request took

    # Increment the request counter.
    # .labels() sets the label values for this specific observation.
    REQUEST_COUNT.labels(method='GET', endpoint='/', status='200').inc()

    # Record the latency in the histogram.
    # .observe(value) adds one data point to the histogram.
    REQUEST_LATENCY.labels(method='GET', endpoint='/').observe(duration)

    return jsonify(response_data), 200


@app.route('/health')
def health():
    """
    Health check endpoint.
    Used by Docker, load balancers, and orchestration tools (like Kubernetes)
    to check if the service is alive and ready to accept traffic.
    Returns a simple JSON response with HTTP 200.
    """
    REQUEST_COUNT.labels(method='GET', endpoint='/health', status='200').inc()
    return jsonify({"status": "healthy", "service": "frontend"}), 200


@app.route('/call-backend')
def call_backend():
    """
    Demonstrates inter-service communication.
    The frontend calls the backend API and returns the combined result.
    In Docker, we use the service name 'backend' (not localhost) to reach it.
    """
    start_time = time.time()

    try:
        # 'backend' is the Docker Compose service name — Docker resolves it to the correct IP.
        # This is equivalent to http://192.168.x.x:5001 but using a stable DNS name.
        response = requests.get('http://backend:5001/', timeout=5)
        backend_data = response.json()
        status_code = '200'
    except requests.exceptions.RequestException as e:
        # If the backend is unreachable, we catch the error gracefully
        # instead of crashing the frontend.
        backend_data = {"error": str(e)}
        status_code = '500'

    duration = time.time() - start_time
    REQUEST_COUNT.labels(method='GET', endpoint='/call-backend', status=status_code).inc()
    REQUEST_LATENCY.labels(method='GET', endpoint='/call-backend').observe(duration)

    return jsonify({
        "frontend": "running",
        "backend_response": backend_data
    })


# --- Section 5: Metrics Endpoint ---

@app.route('/metrics')
def metrics():
    """
    THIS IS THE MOST IMPORTANT ROUTE FOR PROMETHEUS.

    Prometheus scrapes this endpoint every 15 seconds (configured in prometheus.yml).
    It must return:
      - Content-Type: text/plain; version=0.0.4   ← set by CONTENT_TYPE_LATEST
      - Body: all metric data in Prometheus text format ← generated by generate_latest()

    If this returns HTML (text/html), Prometheus will reject it and mark the target as DOWN.
    That is exactly what was causing your earlier error.
    """
    return Response(
        generate_latest(),        # Collects all registered metrics and serializes them to text
        mimetype=CONTENT_TYPE_LATEST  # Sets the correct Content-Type header
    )


# --- Section 6: App Entry Point ---

if __name__ == '__main__':
    # This block only runs when you execute `python app.py` directly.
    # When running inside Docker via gunicorn or flask run, this is skipped.
    # debug=False is important in production — debug mode exposes sensitive info.
    app.run(host='0.0.0.0', port=5000, debug=False)
    # host='0.0.0.0' means "listen on all network interfaces inside the container"
    # Without this, Flask would only listen on 127.0.0.1 (localhost inside the container)
    # and Docker port mapping would not work.
```

### What the `/metrics` Output Looks Like

When Prometheus visits `http://localhost:5000/metrics`, it receives plain text like this:

```
# HELP frontend_requests_total Total number of frontend requests
# TYPE frontend_requests_total counter
frontend_requests_total{endpoint="/",method="GET",status="200"} 42.0
frontend_requests_total{endpoint="/health",method="GET",status="200"} 5.0

# HELP frontend_latency_seconds Frontend request latency in seconds
# TYPE frontend_latency_seconds histogram
frontend_latency_seconds_bucket{endpoint="/",le="0.01",method="GET"} 2.0
frontend_latency_seconds_bucket{endpoint="/",le="0.025",method="GET"} 8.0
frontend_latency_seconds_bucket{endpoint="/",le="0.05",method="GET"} 15.0
frontend_latency_seconds_bucket{endpoint="/",le="0.1",method="GET"} 28.0
frontend_latency_seconds_bucket{endpoint="/",le="+Inf",method="GET"} 42.0
frontend_latency_seconds_sum{endpoint="/",method="GET"} 5.34
frontend_latency_seconds_count{endpoint="/",method="GET"} 42.0
```

Each `# HELP` line is the description, `# TYPE` tells Prometheus the metric type, and each data line has the metric name, labels in `{}`, and the current value.

---

## 6. Step 3 — Backend Flask App (`backend/app.py`)

The backend is structured the same way as the frontend — it's a separate Flask service with its own metrics.

### Complete Code

```python
# ============================================================
# backend/app.py  — Flask Backend with Prometheus Metrics
# ============================================================

import time
import random

from flask import Flask, Response, jsonify

from prometheus_client import (
    Counter,
    Histogram,
    Gauge,             # NEW: Gauge is a metric that can go up AND down
    generate_latest,
    CONTENT_TYPE_LATEST
)

app = Flask(__name__)

# --- Define Prometheus Metrics ---

# Counter: total backend requests received
REQUEST_COUNT = Counter(
    'backend_requests_total',
    'Total number of backend requests',
    ['method', 'endpoint', 'status']
)

# Histogram: backend request latency
REQUEST_LATENCY = Histogram(
    'backend_latency_seconds',
    'Backend request latency in seconds',
    ['method', 'endpoint'],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0]
)

# Gauge: tracks a value that fluctuates — here we simulate active DB connections.
# Unlike a Counter, a Gauge can be set, incremented, or decremented freely.
# Real-world use: current memory usage, active threads, queue depth.
ACTIVE_CONNECTIONS = Gauge(
    'backend_active_connections',
    'Simulated number of active database connections'
)


# --- Routes ---

@app.route('/')
def index():
    """
    Main backend route. Simulates business logic processing.
    """
    start_time = time.time()

    # Simulate DB query time (between 5ms and 200ms)
    time.sleep(random.uniform(0.005, 0.2))

    # Simulate active connections fluctuating between 1 and 20.
    # In a real app: ACTIVE_CONNECTIONS.set(db.pool.active_count())
    ACTIVE_CONNECTIONS.set(random.randint(1, 20))

    duration = time.time() - start_time

    REQUEST_COUNT.labels(method='GET', endpoint='/', status='200').inc()
    REQUEST_LATENCY.labels(method='GET', endpoint='/').observe(duration)

    return jsonify({
        "service": "backend",
        "status": "running",
        "message": "Backend API is operational",
        "processing_time_ms": round(duration * 1000, 2)  # Convert to milliseconds for display
    }), 200


@app.route('/health')
def health():
    """
    Health check for Docker and orchestration tools.
    """
    REQUEST_COUNT.labels(method='GET', endpoint='/health', status='200').inc()
    return jsonify({"status": "healthy", "service": "backend"}), 200


@app.route('/data')
def get_data():
    """
    Simulates a data-fetching API endpoint (like reading from a database).
    """
    start_time = time.time()

    # Simulate heavier DB work (10ms to 500ms)
    time.sleep(random.uniform(0.01, 0.5))

    duration = time.time() - start_time
    REQUEST_COUNT.labels(method='GET', endpoint='/data', status='200').inc()
    REQUEST_LATENCY.labels(method='GET', endpoint='/data').observe(duration)

    return jsonify({
        "data": [
            {"id": 1, "name": "Item One"},
            {"id": 2, "name": "Item Two"},
            {"id": 3, "name": "Item Three"}
        ],
        "count": 3
    }), 200


# --- Metrics Endpoint (CRITICAL) ---

@app.route('/metrics')
def metrics():
    """
    Prometheus scrapes this endpoint to collect all metrics.
    Must return plain text with the correct Content-Type header.

    Common error: returning HTML here causes Prometheus to mark the target as DOWN.
    Fix: always use Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)
    """
    return Response(
        generate_latest(),
        mimetype=CONTENT_TYPE_LATEST
    )


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001, debug=False)
    # Note: backend runs on port 5001 to avoid conflict with the frontend on 5000
```

---

## 7. Step 4 — Dockerfile (Frontend & Backend)

A `Dockerfile` is a script that defines how to build a Docker image for your service. Think of it as a recipe: start with a base ingredient (Python), add your files, install dependencies, and specify how to run the app.

### `frontend/Dockerfile`

```dockerfile
# ============================================================
# frontend/Dockerfile
# ============================================================

# --- Line 1: Base Image ---
# We start FROM an official Python image.
# 'python:3.11-slim' means:
#   - Python version 3.11
#   - 'slim' variant = smaller image, fewer pre-installed tools
#   - Smaller images = faster builds, less disk space, smaller attack surface
FROM python:3.11-slim

# --- Line 2: Working Directory ---
# Sets the default directory inside the container for all subsequent commands.
# All COPY, RUN, and CMD instructions will use /app as their base path.
# It's like doing: cd /app
WORKDIR /app

# --- Line 3: Copy requirements FIRST (Docker Layer Caching Optimization) ---
# We copy requirements.txt BEFORE copying the rest of the code.
# Why? Docker caches each instruction as a "layer".
# If requirements.txt hasn't changed, Docker reuses the cached pip install layer.
# This makes rebuilds after code-only changes MUCH faster (seconds instead of minutes).
COPY requirements.txt .

# --- Line 4: Install Python Dependencies ---
# Installs all packages listed in requirements.txt.
# '--no-cache-dir' tells pip not to store its download cache inside the image,
# which keeps the final image smaller.
RUN pip install --no-cache-dir -r requirements.txt

# --- Line 5: Copy Application Code ---
# Now we copy the rest of the project files into the container.
# The '.' means "copy everything in the current directory (frontend/)"
# to the working directory inside the container (/app).
COPY . .

# --- Line 6: Expose Port ---
# Documents which port this container listens on.
# This is informational — it does NOT actually publish the port.
# Port publishing happens in docker-compose.yml with the 'ports' key.
EXPOSE 5000

# --- Line 7: Start Command ---
# The command Docker runs when the container starts.
# We use the Flask built-in dev server here.
# For production, replace this with gunicorn:
#   CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
CMD ["python", "app.py"]
```

### `backend/Dockerfile`

```dockerfile
# ============================================================
# backend/Dockerfile
# ============================================================

FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5001

# Backend runs on port 5001
CMD ["python", "app.py"]
```

> The backend Dockerfile is identical except for the port number. The port in `EXPOSE` must match the port in `app.run()` inside `app.py`.

---

## 8. Step 5 — Prometheus Configuration (`prometheus.yml`)

This file tells Prometheus **what to scrape** and **how often**. Without this file, Prometheus would collect no data at all.

### `prometheus/prometheus.yml`

```yaml
# ============================================================
# prometheus/prometheus.yml
# ============================================================

# --- Global Settings ---
global:
  # scrape_interval: How often Prometheus visits each /metrics endpoint.
  # 15 seconds is the standard default — a good balance between freshness and load.
  # You can lower this to 5s for faster dashboards, but it increases storage usage.
  scrape_interval: 15s

  # evaluation_interval: How often Prometheus evaluates alerting rules.
  # (We are not setting up alerts here, but this is required configuration.)
  evaluation_interval: 15s

# --- Scrape Jobs ---
# Each 'job' defines one or more targets (services) that Prometheus scrapes.
scrape_configs:

  # Job 1: Scrape Prometheus itself
  # This is optional but useful — it lets you monitor Prometheus's own performance.
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']   # Prometheus scrapes its own /metrics endpoint
        labels:
          service: 'prometheus'       # Custom label applied to all metrics from this target

  # Job 2: Scrape the Flask Frontend
  - job_name: 'frontend'
    # scrape_interval here overrides the global setting for this job only.
    # We scrape the frontend every 10 seconds for more responsive dashboards.
    scrape_interval: 10s

    static_configs:
      # 'frontend' is the Docker Compose service name.
      # Inside Docker's network, this resolves to the frontend container's IP.
      # Port 5000 is where our Flask app listens.
      # Prometheus will automatically append /metrics to this address.
      - targets: ['frontend:5000']
        labels:
          service: 'frontend'         # Label added to every metric from this service

  # Job 3: Scrape the Flask Backend
  - job_name: 'backend'
    scrape_interval: 10s

    static_configs:
      - targets: ['backend:5001']     # Backend service name + port
        labels:
          service: 'backend'
```

### How Prometheus Uses This File

When Prometheus starts, it reads `prometheus.yml` and builds a scrape schedule:

```
Every 10 seconds:
  → GET http://frontend:5000/metrics
  → GET http://backend:5001/metrics

Every 15 seconds:
  → GET http://localhost:9090/metrics (itself)

For each response:
  → Parse the text-format metrics
  → Store them in its time-series database with a timestamp
  → Make them queryable via PromQL
```

---

## 9. Step 6 — Docker Compose (`docker-compose.yml`)

`docker-compose.yml` is the master orchestration file. It defines all services, their relationships, their ports, and how they connect. This one file replaces running multiple `docker run` commands manually.

### `docker-compose.yml`

```yaml
# ============================================================
# docker-compose.yml  — Orchestrates all 4 services
# ============================================================

# 'version' specifies which Docker Compose file format to use.
version: '3.8'

services:

  # ──────────────────────────────────────────────
  # Service 1: Flask Frontend
  # ──────────────────────────────────────────────
  frontend:
    # 'build' tells Docker Compose to build an image from a Dockerfile.
    # 'context: ./frontend' means "use the ./frontend directory as the build context"
    # (all files in that folder are available to the Dockerfile's COPY commands)
    build:
      context: ./frontend
      dockerfile: Dockerfile

    # 'ports' maps a container port to a host (your machine) port.
    # Format: "HOST_PORT:CONTAINER_PORT"
    # "5000:5000" means: visiting localhost:5000 on your machine
    #              → forwards to port 5000 inside the frontend container
    ports:
      - "5000:5000"

    # 'networks' connects this service to a shared Docker network.
    # All services on the same network can reach each other by service name.
    networks:
      - monitoring

    # 'depends_on' controls startup order.
    # The frontend won't start until the backend container is running.
    # Note: this only waits for the container to START, not for the app inside to be READY.
    depends_on:
      - backend

    # 'restart' policy controls what happens if the container crashes.
    # 'unless-stopped' means: restart automatically on crash, but NOT if you manually stop it.
    restart: unless-stopped

  # ──────────────────────────────────────────────
  # Service 2: Flask Backend
  # ──────────────────────────────────────────────
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile

    ports:
      - "5001:5001"

    networks:
      - monitoring

    restart: unless-stopped

  # ──────────────────────────────────────────────
  # Service 3: Prometheus
  # ──────────────────────────────────────────────
  prometheus:
    # We use the official Prometheus Docker image from Docker Hub.
    # No need to build — Docker pulls this image automatically.
    image: prom/prometheus:latest

    ports:
      - "9090:9090"

    # 'volumes' mounts files or directories from your host into the container.
    # Format: "HOST_PATH:CONTAINER_PATH"
    # Here we mount our prometheus.yml config into the standard Prometheus config location.
    # ':ro' means read-only — Prometheus can read the file but cannot modify it.
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro

    # 'command' overrides the default startup command defined in the Docker image.
    # We tell Prometheus where to find our config file inside the container.
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'       # Where to store time-series data
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'

    networks:
      - monitoring

    restart: unless-stopped

  # ──────────────────────────────────────────────
  # Service 4: Grafana
  # ──────────────────────────────────────────────
  grafana:
    image: grafana/grafana:latest

    ports:
      - "3000:3000"

    # Environment variables configure Grafana at startup.
    # These set the default admin username and password.
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      # Disable sign-up for new users (only admin can create accounts)
      - GF_USERS_ALLOW_SIGN_UP=false

    # Named volume to persist Grafana data (dashboards, data sources, settings).
    # Without this, all your dashboards would be lost every time the container restarts.
    volumes:
      - grafana_data:/var/lib/grafana

    networks:
      - monitoring

    depends_on:
      - prometheus

    restart: unless-stopped

# ──────────────────────────────────────────────
# Networks
# ──────────────────────────────────────────────
networks:
  monitoring:
    # 'bridge' is the default Docker network driver.
    # It creates an isolated virtual network that all our services share.
    # Services on this network can reach each other by service name.
    driver: bridge

# ──────────────────────────────────────────────
# Volumes
# ──────────────────────────────────────────────
volumes:
  grafana_data:
    # Docker manages this volume. It persists on disk even after containers stop.
    # Location on Linux: /var/lib/docker/volumes/grafana_data/
```

---

## 10. Step 7 — Running the Stack

### Prerequisites

Make sure you have these installed:

```bash
# Check Docker
docker --version       # Should output: Docker version 24.x.x

# Check Docker Compose
docker compose version # Should output: Docker Compose version v2.x.x
```

### Start Everything

Navigate to your project root and run:

```bash
# Build all images and start all containers in the background (-d = detached mode)
docker compose up --build -d
```

**What happens step by step:**
1. Docker reads `docker-compose.yml`
2. Builds `frontend` image from `./frontend/Dockerfile`
3. Builds `backend` image from `./backend/Dockerfile`
4. Pulls `prom/prometheus:latest` from Docker Hub
5. Pulls `grafana/grafana:latest` from Docker Hub
6. Creates the `monitoring` network
7. Creates the `grafana_data` volume
8. Starts all 4 containers

### Verify All Services Are Running

```bash
docker compose ps
```

Expected output:

```
NAME                    IMAGE               STATUS          PORTS
frontend-1              frontend            Up 30s          0.0.0.0:5000->5000/tcp
backend-1               backend             Up 30s          0.0.0.0:5001->5001/tcp
prometheus-1            prom/prometheus     Up 30s          0.0.0.0:9090->9090/tcp
grafana-1               grafana/grafana     Up 30s          0.0.0.0:3000->3000/tcp
```

All four should show **Up**. If any shows **Exited**, check logs:

```bash
docker compose logs <service_name>
```

### Verify Each Service in Browser

| URL | Expected Response |
|---|---|
| `http://localhost:5000` | JSON with frontend service info |
| `http://localhost:5000/metrics` | Plain text Prometheus metrics |
| `http://localhost:5001` | JSON with backend service info |
| `http://localhost:5001/metrics` | Plain text Prometheus metrics |
| `http://localhost:9090/targets` | Prometheus targets page — all should show UP |
| `http://localhost:3000` | Grafana login page |

### Verify Prometheus Targets

Go to `http://localhost:9090/targets`. You should see:

```
frontend  (1/1 up)   frontend:5000  Last scrape: 3s ago   ✅
backend   (1/1 up)   backend:5001   Last scrape: 5s ago   ✅
```

If a target shows **DOWN**, the most common fixes are:
- The container isn't running (`docker compose ps`)
- The service name in `prometheus.yml` doesn't match the service name in `docker-compose.yml`
- The `/metrics` route is throwing an error (check `docker compose logs frontend`)

### Generate Traffic

Grafana charts need actual data. Generate requests:

```bash
# Send 30 requests to the frontend
for i in {1..30}; do
  curl -s http://localhost:5000 > /dev/null
  curl -s http://localhost:5000/health > /dev/null
  curl -s http://localhost:5001/data > /dev/null
  echo "Request $i sent"
  sleep 0.5
done
```

### Useful Management Commands

```bash
# View logs for all services live
docker compose logs -f

# View logs for a specific service
docker compose logs -f frontend

# Stop everything (containers stop but are not removed)
docker compose stop

# Stop and remove containers (volumes are kept)
docker compose down

# Stop, remove containers AND delete volumes (clean slate)
docker compose down -v

# Rebuild a single service without restarting others
docker compose up --build frontend

# Open a shell inside a running container
docker exec -it <container_name> /bin/bash

# Test metrics from inside the container
docker exec -it <frontend_container_name> curl localhost:5000/metrics
```

---

## 11. Step 8 — Building the Grafana Dashboard

### Open Grafana

Navigate to `http://localhost:3000` and log in:

- **Username:** `admin`
- **Password:** `admin`

### Add Prometheus Data Source

1. Click the **⚙️ gear icon** (Configuration) → **Data Sources**
2. Click **Add data source**
3. Select **Prometheus**
4. Set the URL to:

```
http://prometheus:9090
```

> Why `prometheus` and not `localhost`? — Grafana runs inside its own container. From its perspective, `localhost` is the Grafana container itself. To reach Prometheus (a different container), we use the Docker service name `prometheus`, which Docker resolves to the correct IP on the shared `monitoring` network.

5. Click **Save & Test** — you should see ✅ **Data source is working**

### Create the Dashboard

1. Click **➕ (Create)** → **Dashboard** → **Add new panel**

### Panel 1 — Frontend Request Rate (RPS)

**Query:**
```promql
rate(frontend_requests_total[1m])
```

**How this query works:**
- `frontend_requests_total` is a Counter — it only goes up (42, 43, 44...)
- Raw counter values aren't useful for dashboards (they just keep rising)
- `rate(...[1m])` converts it to "how many requests per second happened in the last 1 minute"
- If 60 requests happened in the last 60 seconds → `rate = 1.0 req/s`

**Settings:**
- Visualization: `Time series`
- Title: `Frontend Request Rate (RPS)`
- Unit: `requests/sec` (Standard Options → Unit → Throughput → requests/sec)

Click **Apply**.

### Panel 2 — Backend Request Rate (RPS)

**Query:**
```promql
rate(backend_requests_total[1m])
```

**Settings:**
- Visualization: `Time series`
- Title: `Backend Request Rate (RPS)`
- Unit: `requests/sec`

Click **Apply**.

### Panel 3 — Frontend Latency (p95)

**Query:**
```promql
histogram_quantile(0.95, rate(frontend_latency_seconds_bucket[1m]))
```

**How this query works:**
- `frontend_latency_seconds_bucket` contains many bucket counters (e.g., "how many requests finished in < 0.1s")
- `rate(...[1m])` calculates the per-second growth rate of each bucket
- `histogram_quantile(0.95, ...)` mathematically estimates the value below which 95% of requests complete
- Result: "95% of frontend requests finish in under X seconds"

**Settings:**
- Visualization: `Time series`
- Title: `Frontend Latency (p95)`
- Unit: `seconds (s)` (Standard Options → Unit → Time → seconds)

Click **Apply**.

### Panel 4 — Backend Latency (p95)

**Query:**
```promql
histogram_quantile(0.95, rate(backend_latency_seconds_bucket[1m]))
```

**Settings:**
- Visualization: `Time series`
- Title: `Backend Latency (p95)`
- Unit: `seconds (s)`

Click **Apply**.

### Panel 5 (Bonus) — Total Requests (Stat Panel)

**Query:**
```promql
frontend_requests_total
```

**Settings:**
- Visualization: `Stat` (shows a big single number, not a graph)
- Title: `Total Frontend Requests`
- Value: Last (shows the most recent value)

Click **Apply**.

### Panel 6 (Bonus) — Active Backend Connections (Gauge)

**Query:**
```promql
backend_active_connections
```

**Settings:**
- Visualization: `Gauge`
- Title: `Backend Active Connections`
- Min: `0`, Max: `20`

Click **Apply**.

### Arrange and Save

Drag panels to create this layout:

```
┌──────────────────────┬──────────────────────┐
│  Frontend RPS        │  Backend RPS         │
├──────────────────────┼──────────────────────┤
│  Frontend Latency    │  Backend Latency     │
├──────────────────────┼──────────────────────┤
│  Total Requests      │  Active Connections  │
└──────────────────────┴──────────────────────┘
```

Click 💾 **Save** → Name: `Microservices Monitoring Dashboard` → Click **Save**.

---

## 12. Understanding the IDE Warning

### What You See

A **yellow underline** under:

```python
from prometheus_client import Counter
```

### Why It Happens

Your IDE (VS Code, PyCharm) checks if Python packages are installed **on your local machine**. Since `prometheus_client` is only installed **inside Docker** (via the Dockerfile), the IDE can't find it locally — hence the warning.

```
Your Machine (IDE)          Docker Container
──────────────────          ─────────────────
No prometheus_client   ≠    prometheus_client ✅ INSTALLED
     ↓
IDE shows yellow warning
     ↓
App STILL RUNS FINE (inside Docker)
```

### The Fix (Three Options)

**Option 1 — Install locally (removes the warning):**

```bash
pip install prometheus_client
```

**Option 2 — Use a virtual environment:**

```bash
# Create a virtual environment in your project folder
python -m venv venv

# Activate it
source venv/bin/activate          # macOS / Linux
venv\Scripts\activate             # Windows

# Install all dependencies
pip install -r frontend/requirements.txt

# Now VS Code will detect the venv and stop showing the warning
```

After creating the venv, select it as your Python interpreter in VS Code:
`Ctrl+Shift+P` → `Python: Select Interpreter` → choose the `venv` option.

**Option 3 — Ignore it (safe if you only run via Docker):**

The warning has zero impact on how your app runs. It is a local IDE convenience feature, not a code error.

---

## 13. Understanding the `/metrics` Internal Server Error

### The Problem

Visiting `http://localhost:5000/metrics` returns a `500 Internal Server Error` page. Prometheus sees `text/html` instead of metric data and marks the target as **DOWN**.

### Root Causes and Fixes

#### Cause 1 — Missing Import (Most Common)

**Broken code:**

```python
# WRONG: CONTENT_TYPE_LATEST and generate_latest are not imported
from prometheus_client import Counter, Histogram

@app.route('/metrics')
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)
    # NameError: name 'generate_latest' is not defined ← CRASH
```

**Fixed code:**

```python
# CORRECT: All required names are imported
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST

@app.route('/metrics')
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)
```

#### Cause 2 — Wrong Return Type

**Broken code:**

```python
@app.route('/metrics')
def metrics():
    return generate_latest()
    # Flask tries to guess Content-Type → defaults to text/html
    # Prometheus receives HTML header → rejects the response
```

**Fixed code:**

```python
@app.route('/metrics')
def metrics():
    # Explicitly set Content-Type using the Response class
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)
```

#### Cause 3 — Container Not Rebuilt After Code Changes

**Symptom:** You fixed the code but the error persists.

**Cause:** Docker is running the old cached image. Your new code is not in the container.

**Fix:**

```bash
# Stop containers
docker compose down

# Force rebuild of all images
docker compose up --build
```

#### Cause 4 — Finding the Exact Error

If the error persists after all fixes, read the Python traceback:

```bash
docker compose logs frontend
```

Look for output like:

```
Traceback (most recent call last):
  File "/app/app.py", line 47, in metrics
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)
NameError: name 'CONTENT_TYPE_LATEST' is not defined
```

This tells you exactly which line failed and why.

---

## 14. Common Errors and Fixes

| Error | Symptom | Root Cause | Fix |
|---|---|---|---|
| Missing import | `NameError` on `/metrics` | `generate_latest` or `CONTENT_TYPE_LATEST` not imported | Add both to import statement |
| Wrong Content-Type | Prometheus sees `text/html` | Returning string instead of `Response` object | Use `Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)` |
| Container using old code | Fix doesn't take effect | Docker using cached image | Run `docker compose up --build` |
| Prometheus target DOWN | `/targets` shows red DOWN | Service unreachable or `/metrics` crashes | Check logs with `docker compose logs <service>` |
| No data in Grafana | Empty charts after setup | No requests sent to Flask | Run the traffic generator loop |
| Grafana "Bad Gateway" | Data source test fails | Wrong Prometheus URL | Use `http://prometheus:9090` not `http://localhost:9090` |
| `docker compose` not found | Command error | Old Docker installation | Use `docker-compose` (with hyphen) for older versions |
| Port already in use | Container won't start | Another app using port 5000/3000/9090 | Stop the conflicting app or change ports in compose file |
| IDE yellow underline | Editor warning | Package not installed locally | `pip install prometheus_client` locally |

---

## 15. Key Concepts Reference

### Prometheus Metric Types

| Type | Behavior | Best Used For | Example |
|---|---|---|---|
| **Counter** | Only goes up, resets on restart | Totals (requests, errors, bytes sent) | `http_requests_total` |
| **Gauge** | Can go up or down freely | Current values (memory, connections, queue size) | `active_threads` |
| **Histogram** | Tracks distribution via buckets | Latency, response sizes | `request_duration_seconds` |
| **Summary** | Quantiles computed in-app | Similar to Histogram, less flexible at query time | `rpc_duration_seconds` |

### PromQL Quick Reference

| Query | What It Computes | Use Case |
|---|---|---|
| `metric_name` | Raw current value | View counters or gauges directly |
| `rate(counter[1m])` | Per-second rate over 1 minute | Convert counters to "per second" |
| `histogram_quantile(0.95, rate(hist_bucket[1m]))` | 95th percentile latency | Latency SLO dashboards |
| `sum(rate(metric[1m]))` | Total rate across all instances | Aggregate traffic across services |
| `avg(metric)` | Average across all label values | Average load per service |
| `metric{label="value"}` | Filter by label | Isolate one endpoint or method |

### Project Files Quick Reference

| File | Location | Purpose |
|---|---|---|
| `app.py` | `frontend/`, `backend/` | Flask application with metrics |
| `requirements.txt` | `frontend/`, `backend/` | Python package dependencies |
| `Dockerfile` | `frontend/`, `backend/` | Container build instructions |
| `prometheus.yml` | `prometheus/` | Scrape targets and intervals |
| `docker-compose.yml` | Project root | Orchestrates all 4 services |

### Docker Compose Commands Reference

```bash
docker compose up --build -d    # Build and start all services (background)
docker compose down             # Stop and remove containers
docker compose down -v          # Also delete volumes (reset Grafana data)
docker compose ps               # List running containers and their status
docker compose logs <service>   # View logs for a service
docker compose logs -f          # Follow all logs in real time
docker compose up --build <svc> # Rebuild and restart one service only
docker exec -it <name> bash     # Open shell inside a container
```

---

## Summary — What You've Built

```
✅ frontend/app.py          — Flask app with Counter + Histogram metrics + /metrics route
✅ backend/app.py           — Flask app with Counter + Histogram + Gauge + /metrics route
✅ frontend/requirements.txt — Dependencies including prometheus_client
✅ backend/requirements.txt  — Same dependencies for backend
✅ frontend/Dockerfile       — Container build recipe for frontend
✅ backend/Dockerfile        — Container build recipe for backend
✅ prometheus/prometheus.yml — Scrape config targeting both Flask services
✅ docker-compose.yml        — Orchestrates all 4 services on a shared network
✅ Grafana Dashboard         — 6 panels showing RPS, latency, totals, and connections
```

**The complete monitoring pipeline:**

```
User Request
    │
    ▼
Flask App (records metrics in memory)
    │
    ▼  [every 10 seconds]
Prometheus scrapes /metrics (stores time-series data)
    │
    ▼  [on demand]
Grafana queries Prometheus (renders live charts)
    │
    ▼
You see beautiful dashboards 🎉
```

---

*Guide built for Flask 3.x · Prometheus 2.x · Grafana 10.x · Docker Compose v2*
