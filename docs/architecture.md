# System Architecture

## High-Level Data Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Factory Floor                                 │
│  CNC_01  CNC_02  COMP_01  COMP_02  HVAC_01  PUMP_01  CONV_01  LIGHT_01 │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ (simulated by energy-data-collector)
                           ▼
               ┌───────────────────────┐
               │   energy-data-collector│
               │   Python 3.12         │
               │   Simulator / Modbus  │
               │   / OPC-UA            │
               └───────────┬───────────┘
                           │ MQTT publish
                           │ topic: factory/{id}/machines/{id}/power
                           │ QoS: 1, interval: 5s
                           ▼
               ┌───────────────────────┐
               │   Mosquitto (MQTT)    │
               │   Eclipse Mosquitto 2 │
               │   Port: 1883, 9001(WS)│
               └───────────┬───────────┘
                           │ MQTT subscribe
                           │ topic: factory/+/machines/+/power
                           ▼
               ┌───────────────────────┐
               │  energy-ingestion-    │
               │  service              │
               │  Python 3.12          │
               │  Validates & batches  │
               └───────────┬───────────┘
                           │ HTTP (InfluxDB line protocol)
                           │ Batch: 50 points / 5s flush
                           ▼
               ┌───────────────────────┐
               │  InfluxDB 2.x         │
               │  Port: 8086           │
               │  Bucket: energy_data  │
               │  Retention: 30 days   │
               └────┬──────┬──────┬────┘
                    │      │      │
         ┌──────────┘  ┌───┘  ┌───┘
         ▼             ▼      ▼
┌──────────────┐ ┌───────┐ ┌────────────┐ ┌──────────────┐
│  analytics   │ │  ml-  │ │optimization│ │ alert-service│
│  :8081       │ │service│ │  :8083     │ │  :8084       │
│  FastAPI     │ │ :8082 │ │  FastAPI   │ │  FastAPI     │
│  Python      │ │FastAPI│ │  Python    │ │  Python      │
└──────┬───────┘ └───┬───┘ └─────┬──────┘ └──────┬───────┘
       │             │           │                │
       └─────────────┴─────┬─────┴────────────────┘
                           │ HTTP proxy
                           ▼
               ┌───────────────────────┐
               │  energy-api-gateway   │
               │  TypeScript/Fastify   │
               │  Port: 8080           │
               └───────────┬───────────┘
                           │ HTTP REST API (JWT-protected)
                     ┌─────┴────────────┐
                     ▼                  ▼
              ┌─────────────┐     ┌─────────┐
              │  energy-    │     │ Grafana │
              │  frontend   │     │  :3000  │
              │  React SPA  │     │(embedded│
              │  :3001      │     │ via     │
              └─────────────┘     │ iframe) │
                                  └─────────┘
```

## Service Dependency Graph

```
mosquitto ◄── data-collector (publishes)
mosquitto ──► ingestion      (subscribes → writes InfluxDB)
influxdb  ◄── ingestion
influxdb  ◄── analytics      (queries)
influxdb  ◄── ml-service     (trains + queries)
influxdb  ◄── optimization   (queries)
influxdb  ◄── alert-service  (queries, every 30s)
influxdb  ◄── api-gateway    (direct queries for machines endpoints)
influxdb  ◄── grafana        (datasource)

analytics    ◄── api-gateway (HTTP proxy)
ml-service   ◄── api-gateway (HTTP proxy)
optimization ◄── api-gateway (HTTP proxy)
alert-service◄── api-gateway (HTTP proxy)

frontend     ◄── api-gateway  (nginx proxy: /api, /auth, /ws)
frontend     ──► grafana      (Grafana panels embedded via iframe)
sqlite       ◄── api-gateway  (users + devices — local volume)
```

## Communication Patterns

### MQTT (asynchronous, event-driven)

| Publisher | Broker | Subscriber | Topic | QoS | Frequency |
|---|---|---|---|---|---|
| data-collector | Mosquitto :1883 | ingestion | `factory/+/machines/+/power` | 1 | 5 seconds |

### HTTP REST (synchronous, request-response)

| Client | Target | Purpose |
|---|---|---|
| api-gateway | analytics :8081 | Proxy analytics queries |
| api-gateway | ml-service :8082 | Proxy ML train/inference |
| api-gateway | optimization :8083 | Proxy optimization queries |
| api-gateway | alert-service :8084 | Proxy alert queries |
| api-gateway | influxdb :8086 | Direct machine data queries |
| analytics | influxdb :8086 | Flux queries |
| ml-service | influxdb :8086 | Training data + inference |
| optimization | influxdb :8086 | Flux queries |
| alert-service | influxdb :8086 | Alert condition evaluation |
| grafana | influxdb :8086 | Dashboard datasource |

### Alert Notifications (optional)

| Trigger | Channel | Destination |
|---|---|---|
| Alert rule match | console | Docker logs |
| Alert rule match | webhook | `ALERT_WEBHOOK_URL` (HTTP POST) |

## Networking

All services run on the default Docker Compose bridge network. Services communicate using Docker DNS names (e.g., `http://influxdb:8086`, `mqtt://mosquitto:1883`). Only the following ports are exposed to the host:

| Port | Service | Protocol |
|---|---|---|
| 1883 | Mosquitto | MQTT |
| 9001 | Mosquitto | MQTT over WebSocket |
| 3000 | Grafana | HTTP |
| 3001 | energy-frontend | HTTP (nginx) |
| 8080 | api-gateway | HTTP REST + WebSocket |
| 8086 | InfluxDB | HTTP |

Internal service ports (8081–8084) are NOT exposed externally; all external access routes through the API gateway on port 8080.

## Key Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Deployment | Docker Compose → k3s | Single-node pilot first, scale later |
| Messaging | MQTT (Mosquitto) | Lightweight pub/sub, decoupled producers/consumers |
| Database | InfluxDB 2.x | Native time-series optimization, 30-day retention, Flux queries |
| Backend | Python 3.12 | ML ecosystem (scikit-learn, Prophet, pandas) |
| API Gateway | TypeScript / Fastify v5 | High-performance Node.js, type safety |
| Auth | JWT (`@fastify/jwt`) + bcryptjs | Stateless, standard, no extra service required |
| User/device store | SQLite (better-sqlite3) | Embedded, zero-ops, sufficient for single-factory MVP |
| RBAC | Role-rank comparison | Simple hierarchy: viewer < operator < plant_manager < admin |
| Frontend | React 18 + Vite 6 + Tailwind v4 | Modern SPA stack; fast dev cycle; shadcn/ui component primitives |
| Grafana integration | Anonymous auth + iframe embedding | Reuses existing dashboards; no duplication of chart code |
| ML: Anomaly | Isolation Forest | Unsupervised, low training data requirements |
| ML: Forecast | Prophet | Handles seasonality/shift patterns, minimal feature engineering |
| Data source | Simulator | MVP iteration without hardware dependency |
| API style | REST (JSON) | Universal, simple client integration |
| Visualization | Grafana + custom React UI | Grafana for raw metrics; React UI for device/auth/recommendations |
