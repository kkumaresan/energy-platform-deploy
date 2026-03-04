# Energy Optimization Platform — Local Setup Guide

Step-by-step instructions to run the full platform locally using Docker Compose.

## Prerequisites

- **Docker Desktop** (v24+) with Docker Compose v2
- **Git** for cloning repositories
- ~4 GB free RAM (ML service with Prophet needs ~1 GB)

## 1. Clone All Repositories

All repos must be cloned into the **same parent directory** as siblings.

```bash
mkdir -p iot-exploration && cd iot-exploration

git clone git@github.com:kkumaresan/energy-platform-deploy.git
git clone git@github.com:kkumaresan/energy-data-collector.git
git clone git@github.com:kkumaresan/energy-ingestion-service.git
git clone git@github.com:kkumaresan/energy-analytics-service.git
git clone git@github.com:kkumaresan/energy-ml-service.git
git clone git@github.com:kkumaresan/energy-api-gateway.git
git clone git@github.com:kkumaresan/energy-optimization-service.git
git clone git@github.com:kkumaresan/energy-alert-service.git
git clone git@github.com:kkumaresan/energy-prototype.git
```

Your directory structure should look like:

```
iot-exploration/
├── energy-platform-deploy/       ← Docker Compose orchestration (you are here)
├── energy-data-collector/        ← Sensor simulator / collector (Python)
├── energy-ingestion-service/     ← MQTT → InfluxDB writer (Python)
├── energy-analytics-service/     ← Energy analytics API (Python/FastAPI)
├── energy-ml-service/            ← ML models — anomaly, forecast, efficiency (Python/FastAPI)
├── energy-api-gateway/           ← Public API gateway (TypeScript/Fastify)
├── energy-optimization-service/  ← Optimization engine (Python/FastAPI)
├── energy-alert-service/         ← Alert rules and notifications (Python/FastAPI)
└── energy-prototype/             ← Architecture docs and schemas
```

## 2. Start the Platform

```bash
cd energy-platform-deploy
docker compose up -d --build
```

First run will take a few minutes to build all service images.

## 3. Verify Services Are Running

```bash
docker compose ps
```

All 10 containers should show `running`:

| Container              | Service            | Port  | Description                          |
|------------------------|--------------------|-------|--------------------------------------|
| energy-mosquitto       | MQTT Broker        | 1883  | Eclipse Mosquitto (+ WebSocket 9001) |
| energy-influxdb        | Time-Series DB     | 8086  | InfluxDB 2.x                         |
| energy-grafana         | Dashboards         | 3000  | Grafana 11.x                         |
| energy-data-collector  | Data Collector     | —     | Simulates 8 factory machines         |
| energy-ingestion       | Ingestion Service  | —     | Writes MQTT messages to InfluxDB     |
| energy-analytics       | Analytics Service  | 8081  | Energy aggregation, idle detection   |
| energy-ml-service      | ML Service         | 8082  | Anomaly, forecast, efficiency models |
| energy-optimization    | Optimization       | 8083  | Compressor, schedule, peak reduction |
| energy-alert-service   | Alert Service      | 8084  | Alert rules, notifications           |
| energy-api-gateway     | API Gateway        | 8080  | Public REST API                      |

## 4. Access the UIs

| UI        | URL                      | Username | Password          |
|-----------|--------------------------|----------|-------------------|
| Grafana   | http://localhost:3000     | admin    | energy-admin      |
| InfluxDB  | http://localhost:8086     | admin    | energy-admin-2026 |

The Grafana dashboard ("Machine Energy Overview") is auto-provisioned and should show live data within 10 seconds of startup.

## 5. Test the APIs

### Health check
```bash
curl http://localhost:8080/api/v1/health
```

### List machines
```bash
curl http://localhost:8080/api/v1/machines
```

### Energy usage (last 1 hour)
```bash
curl "http://localhost:8080/api/v1/energy/usage?range=-1h"
```

### Analytics — hourly energy
```bash
curl "http://localhost:8080/api/v1/energy/hourly?range=-1h"
```

### ML — train all models (wait ~2 min for data to accumulate first)
```bash
curl -X POST http://localhost:8080/api/v1/ml/train
```

### ML — anomaly detection
```bash
curl "http://localhost:8080/api/v1/ml/anomalies/CNC_01?range=-1h"
```

### ML — energy forecast
```bash
curl "http://localhost:8080/api/v1/ml/forecast/CNC_01?horizon=24"
```

### ML — efficiency scores
```bash
curl http://localhost:8080/api/v1/ml/efficiency
```

### Optimization — all suggestions
```bash
curl "http://localhost:8080/api/v1/optimization/suggestions?range=-1h"
```

### Optimization — compressor analysis
```bash
curl "http://localhost:8080/api/v1/optimization/compressor?range=-1h"
```

### Optimization — schedule suggestions
```bash
curl "http://localhost:8080/api/v1/optimization/schedule?range=-1h"
```

### Alerts — active alerts
```bash
curl http://localhost:8080/api/v1/alerts
```

### Alerts — alert history
```bash
curl "http://localhost:8080/api/v1/alerts/history?limit=20"
```

### Alerts — configured rules
```bash
curl http://localhost:8080/api/v1/alerts/rules
```

## 6. Common Commands

```bash
# View logs for all services
docker compose logs -f

# View logs for a specific service
docker compose logs -f data-collector
docker compose logs -f ml-service

# Restart a single service after code changes
docker compose up -d --build data-collector

# Stop all services (preserves data)
docker compose down

# Stop and delete all data (full reset)
docker compose down -v

# Rebuild everything from scratch
docker compose down -v && docker compose up -d --build
```

## 7. Troubleshooting

### Services keep restarting
Check logs for the failing service:
```bash
docker compose logs --tail=50 <service-name>
```

### InfluxDB connection errors on first start
The ingestion and analytics services retry connections on startup. If InfluxDB is slow to initialize, they will reconnect automatically within ~60 seconds.

### ML training returns "Insufficient data"
The simulator needs time to generate enough data points. Wait at least 2 minutes after starting the platform before training models.

### Port conflicts
If ports 1883, 3000, 8080–8084, or 8086 are in use, stop the conflicting process or edit `docker-compose.yml` to remap ports.

## Repository Overview

| Repository                  | Language   | Purpose                                             |
|-----------------------------|------------|-----------------------------------------------------|
| energy-platform-deploy      | YAML       | Docker Compose, Mosquitto/Grafana config             |
| energy-data-collector       | Python     | Sensor data simulator (8 machines, shift patterns)   |
| energy-ingestion-service    | Python     | MQTT subscriber → InfluxDB writer                    |
| energy-analytics-service    | Python     | Energy aggregation, idle detection, peak demand, cost |
| energy-ml-service           | Python     | Anomaly detection, forecasting, efficiency scoring    |
| energy-api-gateway          | TypeScript | Public REST API (proxies to internal services)        |
| energy-optimization-service | Python     | Compressor, schedule, peak demand optimization        |
| energy-alert-service        | Python     | Configurable alert rules, webhook notifications       |
| energy-prototype            | Docs       | Architecture docs, schemas, project overview          |
