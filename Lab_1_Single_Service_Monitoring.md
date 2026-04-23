# LAB 1: Single Service Monitoring — Basic Implementation
### Monitoring a Demo App using Prometheus & Grafana on Docker Desktop (Windows)

---

## Table of Contents

1. [Objective](#1-objective)
2. [Prerequisites](#2-prerequisites)
3. [Architecture Overview](#3-architecture-overview)
4. [Project Folder Structure](#4-project-folder-structure)
5. [Step 1 — Create the Demo Application](#5-step-1--create-the-demo-application)
   - 5.1 [app/app.py — Full Code with Explanation](#51-appapppy--full-code-with-explanation)
   - 5.2 [app/Dockerfile — Full Code with Explanation](#52-appdockerfile--full-code-with-explanation)
6. [Step 2 — Configure Prometheus](#6-step-2--configure-prometheus)
   - 6.1 [prometheus.yml — Full Code with Explanation](#61-prometheusyml--full-code-with-explanation)
7. [Step 3 — Docker Compose Setup](#7-step-3--docker-compose-setup)
   - 7.1 [docker-compose.yml — Full Code with Explanation](#71-docker-composeyml--full-code-with-explanation)
8. [Step 4 — Run the Lab](#8-step-4--run-the-lab)
9. [Step 5 — Access Services](#9-step-5--access-services)
10. [Step 6 — Verify Metrics in Prometheus](#10-step-6--verify-metrics-in-prometheus)
11. [Step 7 — Configure Grafana and Build Dashboard](#11-step-7--configure-grafana-and-build-dashboard)
12. [Expected Output](#12-expected-output)
13. [How Everything Connects — End-to-End Flow](#13-how-everything-connects--end-to-end-flow)
14. [Common Errors and Fixes](#14-common-errors-and-fixes)
15. [Key Concepts Recap](#15-key-concepts-recap)

---

## 1. Objective

By the end of this lab, you will be able to:

- ✅ Set up **Prometheus** and **Grafana** using Docker Desktop on Windows
- ✅ Write a **Flask Python app** that exposes metrics at a `/metrics` endpoint
- ✅ Configure **Prometheus** to automatically scrape those metrics every 5 seconds
- ✅ Build a **Grafana dashboard** that visualizes your app's request count in real time
- ✅ Understand **how monitoring works end-to-end** in a containerized environment

This lab focuses on **a single service** (one demo app) so you can clearly understand each component before scaling up to multiple services.

---

## 2. Prerequisites

Before starting, make sure the following are in place:

| Requirement | How to Check | Install Link |
|---|---|---|
| **Docker Desktop (Windows)** | Open Docker Desktop → it should show "Running" in the taskbar | [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/) |
| **Browser** | Chrome or Edge (recommended) | Already installed on Windows |
| **Text Editor** | VS Code, Notepad++, or any editor | [code.visualstudio.com](https://code.visualstudio.com/) |
| **Python** *(optional)* | `python --version` in terminal | Only needed if running the app outside Docker |

> **Note for Windows Users:** Make sure Docker Desktop is set to use **Linux containers** (not Windows containers). Right-click the Docker icon in your system tray → if you see "Switch to Linux containers", click it. If it says "Switch to Windows containers", you are already on Linux containers (which is correct).

### Verify Docker is Working

Open **PowerShell** or **Command Prompt** and run:

```powershell
docker --version
docker compose version
```

Expected output:

```
Docker version 24.0.x, build xxxxxxx
Docker Compose version v2.x.x
```

If you see version numbers, Docker is ready. If you get `command not found`, Docker Desktop is not installed or not running.

---

## 3. Architecture Overview

### What Each Component Does

| Component | What It Is | What It Does in This Lab |
|---|---|---|
| **Demo App** | A small Python Flask web server | Serves a webpage and counts how many times it has been visited |
| **Prometheus** | An open-source monitoring system | Every 5 seconds, visits the demo app's `/metrics` page and records the numbers |
| **Grafana** | A data visualization platform | Reads Prometheus data and draws live charts on a dashboard |
| **Docker** | A container runtime | Runs all three components in isolated, portable containers |

### Simple Data Flow Diagram

```
  You (Browser)
       │
       │  visit http://localhost:5000
       ▼
┌──────────────────────────────┐
│         Demo App             │
│       (Flask / Python)       │
│                              │
│  GET /      → "Hello!"       │
│  GET /metrics → metric data  │
│                              │
│  app_requests_total = 42     │
└──────────────┬───────────────┘
               │
               │  Prometheus scrapes /metrics every 5 seconds
               ▼
┌──────────────────────────────┐
│          Prometheus          │
│                              │
│  Stores:                     │
│  app_requests_total 42       │
│  app_requests_total 47  ...  │
│  app_requests_total 53  ...  │
│                              │
│  (time-series database)      │
└──────────────┬───────────────┘
               │
               │  Grafana queries Prometheus using PromQL
               ▼
┌──────────────────────────────┐
│           Grafana            │
│                              │
│   📈 Live Chart              │
│   Request Count over Time    │
│                              │
└──────────────────────────────┘
```

### Docker Network Diagram

```
┌─────────────────────────────────────────────────────┐
│              Docker Internal Network                │
│                                                     │
│   ┌─────────────┐     ┌──────────────┐             │
│   │   app       │◄────│  prometheus  │             │
│   │ port: 5000  │     │  port: 9090  │             │
│   └─────────────┘     └──────┬───────┘             │
│                              │                      │
│                       ┌──────▼───────┐             │
│                        │   grafana    │             │
│                        │  port: 3000  │             │
│                        └──────────────┘             │
└─────────────────────────────────────────────────────┘
        ▲                    ▲                 ▲
  localhost:5000       localhost:9090     localhost:3000
  (your browser)       (your browser)    (your browser)
```

> **Key Insight:** Inside Docker, containers talk to each other using their **service names** (`app`, `prometheus`, `grafana`). But you access them from your browser using `localhost` and the mapped port number.

---

## 4. Project Folder Structure

Create the following folder and file layout on your Windows machine. Open **PowerShell** and run:

```powershell
# Create the project folder and subfolders
mkdir prometheus-grafana-lab
cd prometheus-grafana-lab
mkdir app

# Create empty files
New-Item docker-compose.yml
New-Item prometheus.yml
New-Item app\app.py
New-Item app\Dockerfile
```

Your project should now look like this:

```
prometheus-grafana-lab/          ← Root project folder
│
├── docker-compose.yml           ← Orchestrates all 3 services
├── prometheus.yml               ← Tells Prometheus what to scrape
│
└── app/                         ← Your demo application folder
    ├── app.py                   ← The Python Flask web app
    └── Dockerfile               ← Instructions to build the app container
```

> **Tip for VS Code users:** Open the entire folder in VS Code with:
> ```powershell
> code .
> ```
> This gives you a file tree on the left and lets you edit all files in one window.

---

## 5. Step 1 — Create the Demo Application

### 5.1 `app/app.py` — Full Code with Explanation

This is the heart of the lab. It is a tiny web application that does two things:
1. Serves a simple "Hello" page to users
2. Counts every visit and exposes that count at `/metrics` for Prometheus

Open `app/app.py` and paste the following complete code:

```python
# ============================================================
# app/app.py — Demo Flask Application with Prometheus Metrics
# ============================================================

# ── SECTION 1: IMPORTS ──────────────────────────────────────

from flask import Flask, Response
# Flask  → the web framework. It handles incoming HTTP requests
#          and routes them to the correct Python function.
# Response → a class that lets us build a custom HTTP response
#            with specific headers (like Content-Type).

from prometheus_client import Counter, generate_latest
# prometheus_client → the official Python library for Prometheus.
# Counter       → a metric type that ONLY goes up (perfect for counting requests).
#                 It never decreases and resets to 0 when the app restarts.
# generate_latest → a function that reads all registered metrics (like our Counter)
#                   and formats them as plain text that Prometheus can parse.

import time    # Standard Python library. Used to measure how long things take.
import random  # Standard Python library. Can be used later to simulate variable load.

# ── SECTION 2: APP INITIALIZATION ───────────────────────────

app = Flask(__name__)
# Creates a new Flask web application.
# __name__ is a special Python variable that holds the name of the current file.
# Flask uses it internally to find templates and static files.

# ── SECTION 3: DEFINE PROMETHEUS METRICS ────────────────────

REQUEST_COUNT = Counter(
    'app_requests_total',       # ← Metric name: this is how it appears in Prometheus
    'Total HTTP Requests'       # ← Description: shown in Prometheus UI as help text
)
# What this creates:
#   A counter called 'app_requests_total' that starts at 0.
#   Every time we call REQUEST_COUNT.inc(), it goes up by 1.
#   This counter is automatically registered with the prometheus_client library,
#   so generate_latest() will include it in the /metrics output.
#
# Why use a Counter and not a regular Python variable?
#   A regular variable (count = 0; count += 1) is only visible inside your app.
#   A Prometheus Counter is exposed over HTTP so external tools like Prometheus
#   can read it without accessing your source code.

# ── SECTION 4: APPLICATION ROUTES ───────────────────────────

@app.route('/')
def home():
    """
    Homepage route — the main page of our demo app.

    When a user visits http://localhost:5000/, this function runs.
    It increments the request counter by 1 and returns a plain text response.
    """
    REQUEST_COUNT.inc()
    # .inc() increments the counter by 1.
    # You can also use .inc(5) to increment by 5, or .inc(0.5) for fractions.
    # Here we just do .inc() which defaults to incrementing by 1.

    return "Hello from Demo App!"
    # Returns this string as an HTTP 200 response.
    # The browser displays: Hello from Demo App!


@app.route('/metrics')
def metrics():
    """
    Metrics endpoint — THIS IS WHAT PROMETHEUS SCRAPES.

    Prometheus is configured (in prometheus.yml) to visit this URL every 5 seconds.
    It reads the plain text output and stores it in its time-series database.

    The response must:
      1. Have Content-Type: text/plain (set by mimetype='text/plain')
      2. Contain the metric data in Prometheus exposition format

    If this returns HTML or anything else, Prometheus will reject it.
    """
    return Response(
        generate_latest(),      # Serializes all metrics (including REQUEST_COUNT) to text
        mimetype='text/plain'   # Sets the HTTP Content-Type header to text/plain
    )
    # What generate_latest() produces (example output):
    #
    # # HELP app_requests_total Total HTTP Requests
    # # TYPE app_requests_total counter
    # app_requests_total 7.0
    #
    # The first line is the description (help text).
    # The second line tells Prometheus it's a counter.
    # The third line is the actual value.

# ── SECTION 5: START THE SERVER ─────────────────────────────

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
    # host='0.0.0.0' → Listen on ALL network interfaces inside the container.
    #                   Without this, Flask only listens on 127.0.0.1 (localhost
    #                   inside the container), and Docker port mapping won't work.
    #                   Think of '0.0.0.0' as "accept connections from anywhere".
    #
    # port=5000       → The port number this app listens on.
    #                   This must match:
    #                     - EXPOSE 5000 in the Dockerfile
    #                     - "5000:5000" in docker-compose.yml
    #                     - 'app:5000' in prometheus.yml
```

### What the `/metrics` Endpoint Returns

When Prometheus visits `http://app:5000/metrics` (or you visit `http://localhost:5000/metrics` in your browser), it receives plain text like this:

```
# HELP app_requests_total Total HTTP Requests
# TYPE app_requests_total counter
app_requests_total 7.0
```

- `# HELP` — the description we passed to the Counter
- `# TYPE` — tells Prometheus this is a counter (not a gauge or histogram)
- `app_requests_total 7.0` — the metric name and its current value

Every time you refresh `localhost:5000`, the number after `app_requests_total` increases by 1.

---

### 5.2 `app/Dockerfile` — Full Code with Explanation

A `Dockerfile` is a set of instructions that tells Docker how to build a container image for your app. Think of it as a recipe: start with a base ingredient, add your files, install what you need, and specify how to run it.

Open `app/Dockerfile` and paste the following:

```dockerfile
# ============================================================
# app/Dockerfile — Container build instructions for the Demo App
# ============================================================

# ── INSTRUCTION 1: BASE IMAGE ───────────────────────────────
FROM python:3.9-slim
# FROM tells Docker: "start building from this existing image".
# 'python:3.9-slim' means:
#   - Use the official Python image
#   - Version 3.9 of Python
#   - 'slim' variant → stripped-down version with only Python and essentials
#                       (~45 MB vs ~300 MB for the full python:3.9 image)
#
# Why slim? Smaller image = faster to download, faster to start, smaller attack surface.
# The full Python image includes compilers, man pages, and tools we don't need.

# ── INSTRUCTION 2: WORKING DIRECTORY ────────────────────────
WORKDIR /app
# Sets the default working directory for all following instructions.
# All COPY, RUN, and CMD commands will use /app as their base path.
# If /app doesn't exist, Docker creates it automatically.
# Equivalent to: mkdir -p /app && cd /app

# ── INSTRUCTION 3: COPY ALL FILES ───────────────────────────
COPY . .
# COPY <source> <destination>
# First  '.'  → everything in the build context (the app/ folder on your machine)
# Second '.'  → the current WORKDIR inside the container (/app)
#
# So this copies:
#   app/app.py        → /app/app.py   (inside the container)
#   app/Dockerfile    → /app/Dockerfile
#
# Best Practice Note:
#   In production, you'd split this into two COPY steps:
#     COPY requirements.txt .      ← copy requirements first
#     RUN pip install ...          ← install dependencies (cached layer)
#     COPY . .                     ← copy app code last
#   This way, Docker only re-runs pip install when requirements change,
#   making rebuilds much faster. For this simple lab, one COPY is fine.

# ── INSTRUCTION 4: INSTALL DEPENDENCIES ─────────────────────
RUN pip install flask prometheus_client
# RUN executes a shell command during the image BUILD process (not at runtime).
# This installs two Python packages inside the container:
#
#   flask            → the web framework for our app
#   prometheus_client → the library that provides Counter, generate_latest, etc.
#
# After this instruction runs, these packages are permanently baked into the image.
# You do NOT need Python or pip on your Windows machine — everything happens inside Docker.

# ── INSTRUCTION 5: START COMMAND ────────────────────────────
CMD ["python", "app.py"]
# CMD defines the default command that runs when the container STARTS.
# (As opposed to RUN, which runs at build time.)
#
# ["python", "app.py"] is the exec form (preferred over shell form "python app.py")
# because it runs directly as a process — no shell wrapper means faster startup
# and Docker can properly send signals (like Ctrl+C) to stop the process.
#
# This runs: python /app/app.py
# Which starts the Flask server on host 0.0.0.0, port 5000.
```

### RUN vs CMD — Key Difference

| Instruction | When It Runs | Used For |
|---|---|---|
| `RUN` | During `docker build` | Installing packages, setting up the environment |
| `CMD` | When container starts (`docker run`) | Starting your application |

---

## 6. Step 2 — Configure Prometheus

### 6.1 `prometheus.yml` — Full Code with Explanation

This file is Prometheus's instruction manual. It tells Prometheus **where to go** to collect metrics, **how often** to collect them, and **what to call each source**.

Open `prometheus.yml` (in the project root, NOT inside `app/`) and paste:

```yaml
# ============================================================
# prometheus.yml — Prometheus Scrape Configuration
# ============================================================

# ── SECTION 1: GLOBAL SETTINGS ──────────────────────────────
global:
  scrape_interval: 5s
  # scrape_interval: How often Prometheus visits each target's /metrics endpoint.
  #
  # 5s = every 5 seconds.
  #
  # Why 5 seconds for this lab?
  #   It gives you near-real-time feedback as you refresh the demo app.
  #   You'll see the counter increase quickly in Grafana.
  #
  # In production, 15s–30s is more common to reduce load and storage usage.
  # Lower intervals = fresher data, but more storage and CPU usage.
  #
  # This setting applies to ALL scrape jobs unless overridden per-job.

# ── SECTION 2: SCRAPE JOBS ──────────────────────────────────
scrape_configs:

  - job_name: 'demo-app'
    # job_name: A label that identifies this group of targets.
    # This name appears in the Prometheus UI at /targets and is added as
    # a label to every metric scraped from this job:
    #   app_requests_total{job="demo-app"} 42
    #
    # Use descriptive names that identify the service (e.g., 'frontend', 'auth-service')

    static_configs:
      - targets: ['app:5000']
        # targets: A list of host:port addresses for Prometheus to scrape.
        #
        # 'app'   → This is the Docker Compose SERVICE NAME of our demo app container.
        #           Inside Docker's network, 'app' automatically resolves to the
        #           demo app container's internal IP address.
        #           If you used 'localhost:5000' here, it would fail — because
        #           from Prometheus's perspective, localhost means the Prometheus
        #           container itself, not the demo app container.
        #
        # ':5000' → The port our Flask app listens on.
        #
        # Prometheus will automatically call:
        #   GET http://app:5000/metrics
        # every 5 seconds and store whatever it gets.
```

### How Prometheus Uses This Configuration

```
Every 5 seconds, Prometheus does exactly this:

1. Opens the scrape job: 'demo-app'
2. Looks up target: 'app:5000'
3. Makes HTTP GET request to: http://app:5000/metrics
4. Reads the plain text response:
       app_requests_total 7.0
5. Stores it with a timestamp:
       app_requests_total{job="demo-app"} = 7.0 at 14:23:05
       app_requests_total{job="demo-app"} = 9.0 at 14:23:10
       app_requests_total{job="demo-app"} = 12.0 at 14:23:15
       ...
6. This sequence of (timestamp, value) pairs is a "time series"
   — the raw material for Grafana charts.
```

---

## 7. Step 3 — Docker Compose Setup

### 7.1 `docker-compose.yml` — Full Code with Explanation

`docker-compose.yml` is the master control file. Instead of running three separate `docker run` commands with many flags, this single file describes all three services, their configurations, and how they connect.

Open `docker-compose.yml` (in the project root) and paste:

```yaml
# ============================================================
# docker-compose.yml — Orchestrates all 3 services
# ============================================================

version: '3'
# Specifies the Docker Compose file format version.
# Version '3' is widely supported and works on all modern Docker Desktop versions.
# It enables features like named networks, volumes, and service dependencies.

services:
# The 'services' key is where you define all containers.
# Each key under 'services' is a service name — this becomes both:
#   1. The container's hostname inside Docker's network
#   2. The name used in 'depends_on' to set startup order

# ──────────────────────────────────────────────────────────
# SERVICE 1: Demo Application (our Flask Python app)
# ──────────────────────────────────────────────────────────
  app:
    build: ./app
    # 'build' tells Docker Compose to BUILD an image from a Dockerfile.
    # './app' is the path to the folder containing the Dockerfile.
    # Docker Compose will:
    #   1. Go into the ./app directory
    #   2. Find the Dockerfile there
    #   3. Execute each instruction (FROM, WORKDIR, COPY, RUN, CMD)
    #   4. Create a Docker image from the result
    #
    # Contrast with 'image:' (used below for prometheus/grafana) which
    # PULLS a pre-built image from Docker Hub instead of building one.

    container_name: demo-app
    # Sets a fixed, human-readable name for this container.
    # Without this, Docker generates a random name like 'prometheus-grafana-lab-app-1'.
    # With this, the container is always called 'demo-app'.
    # Useful when running:  docker logs demo-app  or  docker exec -it demo-app bash

    ports:
      - "5000:5000"
    # Maps a port on YOUR MACHINE (left) to a port INSIDE the container (right).
    # Format: "HOST_PORT:CONTAINER_PORT"
    #
    # "5000:5000" means:
    #   When you open http://localhost:5000 in your browser
    #   → Docker forwards it to port 5000 inside the demo-app container
    #   → Flask receives the request and responds
    #
    # Without this, the container is running but completely unreachable from your browser.

# ──────────────────────────────────────────────────────────
# SERVICE 2: Prometheus
# ──────────────────────────────────────────────────────────
  prometheus:
    image: prom/prometheus
    # 'image' tells Docker Compose to PULL this pre-built image from Docker Hub.
    # 'prom/prometheus' is the official Prometheus image maintained by the Prometheus team.
    # Docker will download it automatically on first run (requires internet).
    # No Dockerfile needed — the image already contains the Prometheus binary.

    container_name: prometheus
    # Names the container 'prometheus' for easy identification in Docker Desktop
    # and when running docker commands.

    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    # 'volumes' mounts files/folders from your machine INTO the container.
    # Format: "HOST_PATH:CONTAINER_PATH"
    #
    # './prometheus.yml'                → the file YOU created on your Windows machine
    # '/etc/prometheus/prometheus.yml'  → where Prometheus looks for its config by default
    #
    # Without this mount, Prometheus would use its default config (which scrapes nothing).
    # With this mount, Prometheus uses YOUR config (which scrapes the demo app every 5s).
    #
    # Any edit you make to prometheus.yml on your machine is immediately reflected
    # in the container — no rebuild needed (just restart the container).

    ports:
      - "9090:9090"
    # Exposes Prometheus's web UI at http://localhost:9090
    # You can browse metrics, run PromQL queries, and check scrape targets here.

# ──────────────────────────────────────────────────────────
# SERVICE 3: Grafana
# ──────────────────────────────────────────────────────────
  grafana:
    image: grafana/grafana
    # Official Grafana image from Docker Hub (maintained by Grafana Labs).
    # Contains the full Grafana web application — no Dockerfile needed.

    container_name: grafana
    # Names the container 'grafana'.

    ports:
      - "3000:3000"
    # Exposes Grafana's web UI at http://localhost:3000
    # Default login: admin / admin
```

### Visual Summary of docker-compose.yml

```
docker-compose.yml builds/runs:

  ┌─────────────────────────────────────────────────────┐
  │  Service: app                                       │
  │  Built from: ./app/Dockerfile                       │
  │  Container name: demo-app                           │
  │  Your machine:5000  ──→  Container:5000 (Flask)     │
  └─────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────┐
  │  Service: prometheus                                │
  │  Image pulled from: prom/prometheus (Docker Hub)    │
  │  Container name: prometheus                         │
  │  Config file mounted: ./prometheus.yml              │
  │  Your machine:9090  ──→  Container:9090 (Prometheus)│
  └─────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────┐
  │  Service: grafana                                   │
  │  Image pulled from: grafana/grafana (Docker Hub)    │
  │  Container name: grafana                            │
  │  Your machine:3000  ──→  Container:3000 (Grafana)   │
  └─────────────────────────────────────────────────────┘

  All three share the same internal Docker network →
  They can reach each other by service name.
```

---

## 8. Step 4 — Run the Lab

### Open Terminal in the Project Folder

On Windows, open **PowerShell** and navigate to your project:

```powershell
cd C:\path\to\prometheus-grafana-lab
```

Or, if you have the folder open in VS Code, use the integrated terminal:
`Terminal → New Terminal` (shortcut: `` Ctrl+` ``)

### Start All Services

```powershell
docker-compose up --build
```

**What `--build` does:**
- Tells Docker Compose to (re)build the `app` image from `./app/Dockerfile`
- Without `--build`, Docker uses a cached image — so if you changed `app.py`, the changes would not take effect
- Always use `--build` when you change any source file

**What happens after you run this command:**

```
Step 1: Docker reads docker-compose.yml
Step 2: Builds the 'app' image from ./app/Dockerfile
        → FROM python:3.9-slim    (downloads Python image ~45MB, first time only)
        → WORKDIR /app
        → COPY . .
        → RUN pip install flask prometheus_client
        → CMD ["python", "app.py"]
Step 3: Pulls prom/prometheus from Docker Hub   (first time only, ~80MB)
Step 4: Pulls grafana/grafana from Docker Hub   (first time only, ~300MB)
Step 5: Creates internal Docker network
Step 6: Starts all 3 containers
Step 7: Streams logs from all containers to your terminal
```

> **First run tip:** The first run may take 2–5 minutes to download images. Subsequent runs are much faster because Docker caches downloaded layers.

### What You'll See in the Terminal

```
demo-app     | * Serving Flask app 'app'
demo-app     | * Running on all addresses (0.0.0.0)
demo-app     | * Running on http://127.0.0.1:5000
prometheus   | ts=... level=info msg="Server is ready to receive web requests."
grafana      | logger=http.server t=... msg="HTTP Server Listen" addr=[::]:3000
```

When you see these lines, all three services are up. Leave this terminal open — closing it stops all containers.

### Run in Background (Optional)

If you want your terminal back, add `-d` (detached mode):

```powershell
docker-compose up --build -d
```

To stop background containers later:

```powershell
docker-compose down
```

---

## 9. Step 5 — Access Services

Open your browser and visit each URL to confirm all services are running:

### Demo App — `http://localhost:5000`

Expected output (plain text in browser):

```
Hello from Demo App!
```

Every time you load or refresh this page, `app_requests_total` increments by 1.

### Demo App Metrics — `http://localhost:5000/metrics`

Expected output (raw Prometheus metric text):

```
# HELP app_requests_total Total HTTP Requests
# TYPE app_requests_total counter
app_requests_total 3.0
```

The number at the end increases each time you visit `localhost:5000`.

### Prometheus — `http://localhost:9090`

Expected: The Prometheus web UI loads with a search bar at the top.

### Grafana — `http://localhost:3000`

Expected: The Grafana login page appears.

### Quick Verification Table

| URL | Expected | If Not Working |
|---|---|---|
| `http://localhost:5000` | "Hello from Demo App!" | Check `docker-compose logs app` |
| `http://localhost:5000/metrics` | Prometheus text format | Check `docker-compose logs app` |
| `http://localhost:9090` | Prometheus UI | Check `docker-compose logs prometheus` |
| `http://localhost:3000` | Grafana login | Check `docker-compose logs grafana` |

---

## 10. Step 6 — Verify Metrics in Prometheus

### Check Scrape Targets

1. Open `http://localhost:9090`
2. Click **Status** (top menu) → **Targets**

You should see:

```
demo-app (1/1 up)    app:5000    State: UP ✅    Last scrape: 2s ago
```

If you see **DOWN**, the most likely cause is that the demo app container isn't running or the `/metrics` endpoint is failing. Check:

```powershell
docker-compose logs app
```

### Run a PromQL Query

1. Click **Graph** in the top menu (or go to `http://localhost:9090/graph`)
2. In the query input box, type:

```promql
app_requests_total
```

3. Click **Execute**
4. Switch between the **Table** tab (current value) and **Graph** tab (value over time)

### Generate Data by Refreshing the App

Open a new browser tab and go to `http://localhost:5000`. Refresh it **10–15 times**.

Return to Prometheus and click **Execute** again. You will see the number has increased.

### Understanding the Query Result

When you type `app_requests_total` and execute, Prometheus returns something like:

```
app_requests_total{instance="app:5000", job="demo-app"}    15
```

- `app_requests_total` → the metric name we defined in `app.py`
- `instance="app:5000"` → automatically added by Prometheus, shows where it was scraped from
- `job="demo-app"` → automatically added from the `job_name` in `prometheus.yml`
- `15` → the current value (number of times the homepage was visited)

---

## 11. Step 7 — Configure Grafana and Build Dashboard

### Login to Grafana

1. Open `http://localhost:3000`
2. Enter credentials:
   - **Username:** `admin`
   - **Password:** `admin`
3. Grafana will ask you to set a new password — you can skip this for the lab by clicking **Skip**

### Add Prometheus as a Data Source

Before you can build charts, Grafana needs to know where to get data from.

**Step-by-step:**

1. Click the **⚙️ gear icon** in the left sidebar → **Data Sources**
2. Click **Add data source**
3. Select **Prometheus** from the list (it's usually at the top)
4. In the **URL** field, enter exactly:

```
http://prometheus:9090
```

> **Why `prometheus` and not `localhost`?**
> Grafana runs inside its own Docker container. From its perspective, `localhost` means "inside the Grafana container itself" — not the Prometheus container. To reach Prometheus, Grafana uses the Docker service name `prometheus`, which Docker automatically resolves to the correct IP address on the shared network.

5. Leave all other settings at their defaults
6. Scroll to the bottom and click **Save & Test**
7. You should see a green banner: ✅ **Data source connected and labels found.**

### Create a New Dashboard

1. Click **➕ (Create)** in the left sidebar → **Dashboard**
2. Click **Add new panel**

You are now in the **Panel Editor** — the main screen for configuring a single chart.

### Configure the Panel

The panel editor has three main areas:

```
┌────────────────────────────────────────────┐
│           Chart Preview (top)              │
│   Live preview updates as you type         │
├────────────────────────────────────────────┤
│           Query Editor (bottom left)       │
│   Where you write your PromQL query        │
├────────────────────────────────────────────┤
│           Panel Options (right sidebar)    │
│   Title, visualization type, units, etc.  │
└────────────────────────────────────────────┘
```

**In the Query Editor (bottom), enter:**

```promql
app_requests_total
```

**In the right sidebar, configure:**

| Setting | Value | Where to Find It |
|---|---|---|
| **Title** | `Total App Requests` | Panel Options → Title |
| **Visualization** | `Time series` | Top right dropdown |

The chart preview will update and show a line graph climbing upward — each step represents when you refreshed the demo app.

**Click Apply** (top right) to save the panel.

### Save the Dashboard

1. Click the 💾 **save icon** at the top of the dashboard
2. Enter a name:

```
Demo App Monitoring
```

3. Click **Save**

### Make the Dashboard Live

To see the chart update in real time:

1. Set **time range** (top right) to `Last 5 minutes`
2. Set **auto-refresh** (top right, clock icon) to `5s` (every 5 seconds)
3. Open a new tab → go to `http://localhost:5000` → refresh several times
4. Watch the Grafana chart update automatically!

---

## 12. Expected Output

After completing all steps, this is what you should see:

### Demo App (`localhost:5000`)

```
Hello from Demo App!
```

### Metrics Endpoint (`localhost:5000/metrics`)

```
# HELP app_requests_total Total HTTP Requests
# TYPE app_requests_total counter
app_requests_total 23.0
```

### Prometheus Targets (`localhost:9090/targets`)

```
Endpoint          State   Labels                    Last Scrape   Duration
http://app:5000   UP ✅   job="demo-app"            3s ago        2ms
```

### Prometheus Query (`localhost:9090/graph`)

After querying `app_requests_total`:

```
app_requests_total{instance="app:5000", job="demo-app"}   23
```

### Grafana Dashboard (`localhost:3000`)

A **Time Series chart** titled "Total App Requests" showing a steadily rising line — each step up represents a request to `localhost:5000`.

```
Request Count
    ▲
 25 │                              ●────
 20 │                    ●────────
 15 │           ●───────
 10 │   ●───────
  5 │───
  0 └──────────────────────────────────▶ Time
    14:10     14:11     14:12     14:13
```

---

## 13. How Everything Connects — End-to-End Flow

Here is the complete picture of what happens from when you run `docker-compose up` to seeing data in Grafana:

```
STEP 1 — You run docker-compose up --build
    Docker reads docker-compose.yml
    Builds demo-app image from ./app/Dockerfile
    Pulls prometheus and grafana images from Docker Hub
    Creates an internal network connecting all 3 containers
    Starts all 3 containers simultaneously

STEP 2 — Demo App starts
    Flask starts on port 5000 inside the container
    Registers the REQUEST_COUNT counter with prometheus_client
    Waits for incoming HTTP requests

STEP 3 — Prometheus starts
    Reads prometheus.yml (mounted from your machine)
    Sees job: demo-app, target: app:5000
    Starts a scrape scheduler: every 5 seconds, scrape http://app:5000/metrics

STEP 4 — You visit http://localhost:5000
    Browser sends request to localhost:5000
    Docker forwards it to port 5000 inside the demo-app container
    Flask's home() function runs
    REQUEST_COUNT.inc() → counter goes from 0 to 1
    Returns "Hello from Demo App!"

STEP 5 — Prometheus scrapes metrics (every 5 seconds)
    GET http://app:5000/metrics
    Flask's metrics() function runs
    generate_latest() serializes REQUEST_COUNT = 1
    Returns: app_requests_total 1.0
    Prometheus stores: {timestamp: T1, value: 1.0}

STEP 6 — You visit localhost:5000 a few more times
    Counter increments: 2, 3, 4, 5...
    Prometheus keeps scraping and storing each value with timestamps

STEP 7 — You open Grafana
    Log in → add Prometheus as data source (http://prometheus:9090)
    Grafana connects to Prometheus on the internal Docker network

STEP 8 — You create a dashboard panel with query: app_requests_total
    Grafana sends this PromQL query to Prometheus
    Prometheus returns all stored time-series data for app_requests_total
    Grafana plots it as a line graph: time on X-axis, request count on Y-axis

STEP 9 — Live monitoring
    Auto-refresh set to 5s
    Every 5 seconds, Grafana re-queries Prometheus
    Prometheus has new data from its latest scrape
    Chart updates in real time → you see the line going up as you generate requests
```

---

## 14. Common Errors and Fixes

### Error 1 — Port Already in Use

**Symptom:**

```
Error: bind: address already in use (port 5000 or 3000 or 9090)
```

**Cause:** Another app on your machine is using that port.

**Fix:**

```powershell
# Find what's using port 5000
netstat -ano | findstr :5000

# Kill the process (replace XXXX with the PID from above)
taskkill /PID XXXX /F
```

Or change the HOST port in `docker-compose.yml` (leave the CONTAINER port unchanged):

```yaml
ports:
  - "5001:5000"   # Changed host port from 5000 to 5001
```

Then access the app at `localhost:5001` instead.

---

### Error 2 — Prometheus Target Shows DOWN

**Symptom:** `localhost:9090/targets` shows `State: DOWN` with error `connection refused`

**Cause:** The demo app container isn't running, or `/metrics` is crashing.

**Fix:**

```powershell
# Check if all containers are running
docker-compose ps

# View demo app logs for errors
docker-compose logs app

# Restart just the app service
docker-compose restart app
```

---

### Error 3 — `/metrics` Returns Internal Server Error

**Symptom:** Visiting `localhost:5000/metrics` shows a `500 Internal Server Error` page.

**Cause:** The metrics function in `app.py` is missing imports or has a bug.

**Fix:** Make sure your `app.py` imports are exactly:

```python
from flask import Flask, Response
from prometheus_client import Counter, generate_latest
```

And the route is exactly:

```python
@app.route('/metrics')
def metrics():
    return Response(generate_latest(), mimetype='text/plain')
```

Then rebuild:

```powershell
docker-compose down
docker-compose up --build
```

---

### Error 4 — Grafana "Data source connected but no labels found"

**Symptom:** Grafana shows a yellow warning when testing the Prometheus data source.

**Cause:** Prometheus has no data yet (app hasn't been scraped yet).

**Fix:** Visit `localhost:5000` a few times to generate requests, wait 10 seconds for Prometheus to scrape, then test the data source again.

---

### Error 5 — Empty Graph in Grafana

**Symptom:** Dashboard panel shows no data / blank chart.

**Causes and fixes:**

| Cause | Fix |
|---|---|
| No traffic sent to app | Refresh `localhost:5000` several times |
| Wrong time range | Set time range to "Last 5 minutes" |
| Wrong query | Use exactly `app_requests_total` |
| Prometheus not connected | Re-test data source in Grafana settings |

---

### Error 6 — `docker-compose` Command Not Found

**Symptom:** `'docker-compose' is not recognized as a command`

**Cause:** You have a newer version of Docker that uses `docker compose` (with a space, not a hyphen).

**Fix:** Replace all `docker-compose` commands with `docker compose`:

```powershell
docker compose up --build     # instead of docker-compose up --build
docker compose down           # instead of docker-compose down
docker compose logs app       # instead of docker-compose logs app
```

---

## 15. Key Concepts Recap

### What Is a Counter?

A **Counter** is a Prometheus metric type that only increases. It starts at 0 when your app starts and goes up every time you call `.inc()`. It resets to 0 if the app restarts.

```python
REQUEST_COUNT = Counter('app_requests_total', 'Total HTTP Requests')
REQUEST_COUNT.inc()   # Goes from 0 → 1 → 2 → 3 → ...
```

Counters are used for things that only grow: total requests, total errors, total bytes sent.

### What Is `generate_latest()`?

`generate_latest()` is a function from `prometheus_client` that serializes all registered metrics into a plain text string in Prometheus's exposition format. Every metric you create (`Counter`, `Gauge`, `Histogram`) is automatically included.

### What Is PromQL?

PromQL (Prometheus Query Language) is how you ask Prometheus for data. In this lab we used the simplest possible query — just the metric name:

```promql
app_requests_total
```

This returns the current value of the counter. More advanced queries (used in later labs) use functions like `rate()` and `histogram_quantile()`.

### What Does `scrape_interval: 5s` Mean?

It means Prometheus visits each target's `/metrics` endpoint every 5 seconds and stores the returned values. The smaller the interval, the more frequently data is recorded — but it also increases storage and CPU usage.

### Why Use `host='0.0.0.0'` in Flask?

Inside a Docker container, `127.0.0.1` (localhost) only refers to the container itself. If Flask listens on `127.0.0.1:5000`, Docker cannot forward external traffic to it. `0.0.0.0` means "listen on all interfaces" — this allows Docker to forward requests from your browser into the container.

### Docker Compose Commands Reference

```powershell
# Build images and start all containers (foreground — shows logs)
docker-compose up --build

# Build images and start all containers (background)
docker-compose up --build -d

# Stop and remove all containers
docker-compose down

# View logs for all services
docker-compose logs

# View logs for one service (live)
docker-compose logs -f app

# Check running containers and their status
docker-compose ps

# Rebuild and restart one service only
docker-compose up --build app
```

---

## Lab Summary

In this lab you:

| Task | File | Key Concept |
|---|---|---|
| Created a Flask app that counts requests | `app/app.py` | `Counter`, `generate_latest`, `/metrics` route |
| Containerized the app | `app/Dockerfile` | `FROM`, `WORKDIR`, `COPY`, `RUN`, `CMD` |
| Configured Prometheus to scrape the app | `prometheus.yml` | `scrape_interval`, `job_name`, `targets` |
| Orchestrated all services with Docker | `docker-compose.yml` | `build`, `image`, `ports`, `volumes` |
| Verified metrics collection | Prometheus UI | PromQL: `app_requests_total` |
| Visualized data in a live dashboard | Grafana UI | Data source, panel, time series |

**The monitoring pipeline you built:**

```
Browser visit → Flask increments counter
                       ↓ (every 5s)
               Prometheus scrapes /metrics
                       ↓ (on demand)
               Grafana queries Prometheus
                       ↓
               Live chart updates in real time ✅
```

---

*LAB 1 Complete — Single Service Monitoring | Prometheus + Grafana + Docker Desktop (Windows)*
