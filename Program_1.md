📘 Prometheus & Grafana Monitoring Labs (Docker-Based)
👨‍🏫 Prepared For: Students (Beginner to Intermediate)
🎯 Objective:

To understand monitoring, metrics, and observability using:

Prometheus (metrics collection)
Grafana (visualization)
Docker (containerization)
🧪 LAB 1: Single Service Monitoring (Basic Implementation)
🎯 Objective
Create a simple Python web app
Expose metrics using Prometheus client
Visualize data in Grafana
🏗️ Architecture
[ Flask App ] → [ Prometheus ] → [ Grafana ]

📁 Project Structure
lab1/
├── docker-compose.yml
├── prometheus.yml
└── app/
    ├── app.py
    └── Dockerfile

🧾 Step 1: Understanding the Code (app.py)
from flask import Flask, Response
from prometheus_client import Counter, generate_latest, CONTENT_TYPE_LATEST

app = Flask(__name__)

REQUEST_COUNT = Counter('app_requests_total', 'Total HTTP Requests')

@app.route('/')
def home():
    REQUEST_COUNT.inc()
    return "Hello from Demo App!"

@app.route('/metrics')
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)

app.run(host='0.0.0.0', port=5000)

🧠 Code Explanation (Beginner Friendly)
🔑 Keywords & Identifiers
Term	Meaning
Flask	Web framework to create API
Counter	Prometheus metric type
generate_latest()	Generates metrics output
@app.route()	Defines API endpoint
REQUEST_COUNT	Variable (metric object)
🔍 Line-by-Line Explanation
1. Import Libraries
from flask import Flask, Response

Flask: used to create web server
Response: used to send custom HTTP response
2. Prometheus Setup
REQUEST_COUNT = Counter('app_requests_total', 'Total HTTP Requests')

Counter: counts number of requests
'app_requests_total': metric name
Description helps in Grafana
3. Home Endpoint
@app.route('/')
def home():
    REQUEST_COUNT.inc()

@app.route('/'): defines root URL
.inc(): increments counter
4. Metrics Endpoint
@app.route('/metrics')
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)

/metrics: required by Prometheus
generate_latest(): creates metrics output
CONTENT_TYPE_LATEST: ensures correct format
🐳 Dockerfile
FROM python:3.9-slim

WORKDIR /app
COPY . .

RUN pip install flask prometheus_client

CMD ["python", "app.py"]

🔍 Explanation
Command	Purpose
FROM	Base image
WORKDIR	Working directory
COPY	Copy files
RUN	Install dependencies
CMD	Run application
⚙️ Prometheus Configuration
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'app'
    static_configs:
      - targets: ['app:5000']

🔍 Explanation
Term	Meaning
scrape_interval	Time between data collection
targets	App location
app:5000	Docker service name
🐳 Docker Compose
version: '3'

services:

  app:
    build: ./app
    ports:
      - "5000:5000"

  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"

▶️ Run the Lab
docker compose up --build

🌐 Access
Tool	URL
App	http://localhost:5000

Prometheus	http://localhost:9090

Grafana	http://localhost:3000
📊 Grafana Dashboard Queries
app_requests_total
rate(app_requests_total[1m])

🎓 Learning Outcome
Understanding of metrics
Prometheus scraping
Basic dashboard creation
