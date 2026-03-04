# Energy Platform Deploy

Docker Compose deployment for the Energy Optimization Platform.

## Prerequisites

- Docker and Docker Compose
- Sibling repos cloned at the same directory level:
  ```
  iot-exploration/
  ├── energy-data-collector/
  ├── energy-ingestion-service/
  └── energy-platform-deploy/    ← you are here
  ```

## Quick Start

```bash
docker compose up -d
```

## Services

| Service         | Port  | URL                        |
|-----------------|-------|----------------------------|
| Grafana         | 3000  | http://localhost:3000       |
| InfluxDB        | 8086  | http://localhost:8086       |
| MQTT (Mosquitto)| 1883  | mqtt://localhost:1883       |
| MQTT WebSocket  | 9001  | ws://localhost:9001         |

## Default Credentials

| Service  | Username | Password          |
|----------|----------|-------------------|
| Grafana  | admin    | energy-admin      |
| InfluxDB | admin    | energy-admin-2026 |

## Commands

```bash
# Start all services
docker compose up -d

# View logs
docker compose logs -f

# View specific service logs
docker compose logs -f data-collector

# Stop all services
docker compose down

# Stop and remove volumes (reset all data)
docker compose down -v
```
