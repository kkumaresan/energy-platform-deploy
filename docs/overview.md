# Energy Optimization Platform — Project Overview

## Objective

AI-driven energy optimization platform for SME (small-to-medium enterprise) factories. The platform ingests real-time sensor telemetry from factory floor machines, analyzes energy consumption patterns, detects inefficiencies and anomalies, and generates actionable optimization recommendations — all visualized through a live Grafana dashboard.

## Key Goals

- **Real-time monitoring**: Collect machine power, voltage, current, temperature, and status every 5 seconds
- **Anomaly detection**: Identify abnormal power consumption using ML (Isolation Forest)
- **Energy forecasting**: Predict 24-hour energy consumption per machine (Prophet time-series)
- **Efficiency scoring**: Score each machine's energy efficiency relative to its rated capacity
- **Optimization recommendations**: Generate actionable suggestions for compressor tuning, shift scheduling, and peak demand reduction
- **Alerting**: Rule-based configurable alerts with webhook notification support
- **Visualization**: Auto-provisioned Grafana dashboards with live sensor data

## Target Users

Factory operators and energy managers at SME manufacturing facilities.

## Current Status

The platform runs entirely on **synthetic simulation** (8 virtual machines with realistic shift patterns and anomaly injection). It is MVP-ready for pilot deployment. Real sensor integration via Modbus TCP/RTU and OPC-UA is planned for future phases.

## Repository Structure

All repositories must be siblings in the same parent directory:

```
iot-exploration/
├── energy-platform-deploy/       # Docker Compose orchestration (this repo)
├── energy-data-collector/        # Sensor simulator / Modbus / OPC-UA collector (Python)
├── energy-ingestion-service/     # MQTT → InfluxDB writer (Python)
├── energy-analytics-service/     # Energy aggregation, idle detection, cost API (Python/FastAPI)
├── energy-ml-service/            # Anomaly detection, forecasting, efficiency (Python/FastAPI)
├── energy-api-gateway/           # Public REST API gateway (TypeScript/Fastify)
├── energy-optimization-service/  # Compressor, schedule, peak optimization (Python/FastAPI)
├── energy-alert-service/         # Alert rules, notifications (Python/FastAPI)
└── energy-prototype/             # Architecture docs and JSON schemas
```

## Tech Stack Summary

| Layer | Technology |
|---|---|
| Sensor data ingestion | Python 3.12, paho-mqtt |
| Message broker | Eclipse Mosquitto 2 (MQTT) |
| Time-series database | InfluxDB 2.x (Flux query language) |
| Analytics & optimization | Python 3.12, FastAPI, pandas |
| ML models | scikit-learn (Isolation Forest), Prophet (forecasting) |
| API gateway | TypeScript, Node.js 22, Fastify v5 |
| Visualization | Grafana 11.x |
| Deployment | Docker Compose (future: k3s/Kubernetes) |

## Development Phases

| Phase | Scope | Status |
|---|---|---|
| 1 | Data pipeline: simulator → MQTT → InfluxDB → Grafana | Complete |
| 2 | Analytics API: aggregation, idle detection, cost estimation | Complete |
| 3 | ML models: anomaly detection, forecasting, efficiency scoring | Complete |
| 4 | Optimization engine and alert service | Complete |
| 5 | Real sensor integration: Modbus TCP/RTU, OPC-UA | Planned |
| 6 | Kubernetes (k3s) migration for multi-factory deployment | Planned |
