# Services Reference

## 1. energy-data-collector

**Language**: Python 3.12
**Role**: Reads (or simulates) factory sensor data and publishes to MQTT
**Docker service name**: `data-collector`
**Exposed ports**: None (internal only)

### Key Dependencies
- `paho-mqtt>=2.1` — MQTT client
- `pydantic>=2.0` — Data validation and settings
- `pyyaml>=6.0` — Machine configuration loading
- `pymodbus>=3.6` *(optional)* — Modbus TCP/RTU real sensor mode
- `asyncua>=1.1` *(optional)* — OPC-UA real sensor mode

### Collection Modes

| Mode | Description | Status |
|---|---|---|
| `simulator` | Generates synthetic data for 8 factory machines | Default / Active |
| `modbus` | Reads from Modbus TCP/RTU industrial devices | Future |
| `opc-ua` | Reads from OPC-UA servers | Future |

### Simulated Machines

| Machine ID | Type | Rated Power |
|---|---|---|
| CNC_01 | cnc | 15 kW |
| CNC_02 | cnc | 18 kW |
| COMP_01 | compressor | 22 kW |
| COMP_02 | compressor | 15 kW |
| HVAC_01 | hvac | 30 kW |
| PUMP_01 | pump | 7.5 kW |
| CONV_01 | conveyor | 3 kW |
| LIGHT_01 | lighting | 8 kW |

### Simulation Behavior
- Shift-based operation: day (06:00–14:00), evening (14:00–22:00), night (22:00–06:00)
- Machine state machine: `OFF → RUNNING → IDLE → FAULT`
- Power curves with Gaussian noise
- 1% probability of anomaly per reading (power spike or efficiency drop)
- Compressors emit additional metrics: pressure and flow rate

### MQTT Output
- **Topic**: `factory/{factory_id}/machines/{machine_id}/power`
- **QoS**: 1 (at-least-once)
- **Interval**: 5 seconds (configurable)

---

## 2. energy-ingestion-service

**Language**: Python 3.12
**Role**: Subscribes to MQTT, validates messages, batch-writes to InfluxDB
**Docker service name**: `ingestion`
**Exposed ports**: None (internal only)

### Key Dependencies
- `paho-mqtt>=2.1` — MQTT subscriber
- `influxdb-client>=1.40` — InfluxDB write client
- `pydantic>=2.0` — Message schema validation

### Behaviour
1. Subscribes to `factory/+/machines/+/power` with QoS 1
2. Validates each message against the `SensorMessage` Pydantic schema
3. Rejects messages missing required fields (`factory_id`, `machine_id`, `timestamp`, `metrics.power_kw`)
4. Batches valid points and flushes to InfluxDB every 5 seconds or 50 points, whichever comes first
5. Retries InfluxDB connection up to 30 times (2-second intervals) on startup

### InfluxDB Writes

**Measurement: `machine_power`**
- Tags: `factory_id`, `machine_id`, `machine_type`
- Fields: `power_kw`, `voltage`, `current`, `power_factor`, `temperature`, `pressure`, `flow_rate`

**Measurement: `machine_status`**
- Tags: `factory_id`, `machine_id`, `machine_type`
- Fields: `state` (running/idle/off/fault/maintenance)

---

## 3. energy-analytics-service

**Language**: Python 3.12
**Framework**: FastAPI + Uvicorn
**Role**: Computes aggregated energy analytics from InfluxDB
**Docker service name**: `analytics`
**Internal port**: 8081

### Key Dependencies
- `fastapi>=0.115`, `uvicorn>=0.34` — API server
- `influxdb-client>=1.40` — Flux query client
- `pandas>=2.2` — Data processing
- `apscheduler>=3.10` — Periodic job scheduling

### Capabilities
- **Hourly aggregation**: Mean power consumption grouped by hour
- **Daily aggregation**: Total kWh per machine per day
- **Shift aggregation**: Energy by shift (day/evening/night)
- **Idle detection**: Machines consuming <15% of rated power flagged as idle
- **Peak demand tracking**: Highest instantaneous demand per machine
- **Cost estimation**: Time-of-use tariff calculation (peak: £0.15/kWh, off-peak: £0.08/kWh; peak hours: 09:00–17:00)

### Tariff Configuration
- Peak rate: £0.15/kWh (configurable via `ANALYTICS_TARIFF_PEAK_RATE`)
- Off-peak rate: £0.08/kWh (configurable via `ANALYTICS_TARIFF_OFFPEAK_RATE`)
- Peak hours: 09:00–17:00 (configurable)

---

## 4. energy-ml-service

**Language**: Python 3.12
**Framework**: FastAPI + Uvicorn
**Role**: ML model training, anomaly detection, energy forecasting, efficiency scoring
**Docker service name**: `ml-service`
**Internal port**: 8082

### Key Dependencies
- `fastapi>=0.115`, `uvicorn>=0.34` — API server
- `scikit-learn>=1.5` — Isolation Forest anomaly detection
- `prophet>=1.1` — Time-series forecasting
- `pandas>=2.2`, `numpy>=2.0` — Data manipulation
- `joblib>=1.4` — Model serialization
- `influxdb-client>=1.40` — Training data queries

### ML Models

| Model | Algorithm | Purpose | Training Data |
|---|---|---|---|
| Anomaly Detector | Isolation Forest | Detect abnormal power readings | Last 7 days per machine |
| Energy Forecaster | Facebook Prophet | 24-hour power consumption forecast | Last 7 days per machine |
| Efficiency Scorer | Statistical heuristic | Score relative to rated capacity | Last 24 hours per machine |

### Model Storage
Trained model artifacts are persisted to `/app/model_artifacts/` (Docker volume: `ml-model-artifacts`), so models survive container restarts.

### Configuration
- Anomaly contamination: 5% (`ML_ANOMALY_CONTAMINATION=0.05`)
- Rolling window for anomaly features: 12 samples
- Forecast horizon: 24 hours (configurable per request)
- Minimum training data: 1 day

---

## 5. energy-optimization-service

**Language**: Python 3.12
**Framework**: FastAPI + Uvicorn
**Role**: Rule-based and algorithmic optimization recommendations
**Docker service name**: `optimization`
**Internal port**: 8083

### Key Dependencies
- `fastapi>=0.115`, `uvicorn>=0.34` — API server
- `influxdb-client>=1.40` — Historical data queries
- `pandas>=2.2` — Data analysis
- `pyyaml>=6.0` — Rule configuration

### Optimization Strategies

| Strategy | Description |
|---|---|
| Compressor optimization | Analyzes pressure and load factor; recommends load/unload cycle tuning |
| Schedule optimization | Suggests shift-based scheduling to maximize off-peak operation |
| Peak demand reduction | Identifies and recommends staggering high-demand machines during peak tariff hours |

---

## 6. energy-alert-service

**Language**: Python 3.12
**Framework**: FastAPI + Uvicorn
**Role**: Configurable alert rules with multi-channel notifications
**Docker service name**: `alert-service`
**Internal port**: 8084

### Key Dependencies
- `fastapi>=0.115`, `uvicorn>=0.34` — API server
- `influxdb-client>=1.40` — Condition evaluation queries
- `httpx>=0.28` — Webhook HTTP client
- `pandas>=2.2` — Data processing
- `pyyaml>=6.0` — Alert rule definitions

### Alert Rules (default configuration)

| Rule | Condition | Threshold | Duration | Severity | Channels |
|---|---|---|---|---|---|
| energy_spike | power_ratio > rated | 150% | 5 min | warning | console |
| idle_machine_waste | idle power > rated | 20% | 60 min | info | console |
| power_factor_drop | power factor < | 0.85 | 10 min | critical | console, webhook |
| high_temperature | temperature > | 80°C | 5 min | warning | console |

### Alert Lifecycle
1. Background watcher queries InfluxDB every 30 seconds
2. Each rule's condition is evaluated against current readings
3. Alert triggers only if condition persists beyond the configured duration threshold
4. Triggered alerts are stored in memory (max 1000 history entries)
5. Notification channels: `console` (logs) or `webhook` (HTTP POST)

### Machine Rated Power (for ratio calculations)

| Machine | Rated kW |
|---|---|
| CNC_01, CNC_02 | 15.0 |
| COMP_01, COMP_02 | 22.0 |
| HVAC_01 | 10.0 |
| PUMP_01 | 5.5 |
| CONV_01 | 3.0 |
| LIGHT_01 | 2.0 |

---

## 7. energy-api-gateway

**Language**: TypeScript (Node.js 22)
**Framework**: Fastify v5
**Role**: Public REST API — the single entry point for all external clients; hosts JWT auth, RBAC, and device management
**Docker service name**: `api-gateway`
**Exposed port**: 8080

### Key Dependencies
- `fastify>=5.2` — HTTP server
- `@fastify/jwt>=9.0` — JWT sign/verify plugin
- `@fastify/cors>=11.0` — CORS support (default: `*`)
- `@fastify/websocket>=11.0` — WebSocket support
- `better-sqlite3>=9.0` — Embedded SQLite for users and devices
- `bcryptjs>=2.4` — Password hashing (pure JS, no native deps beyond SQLite)
- `@influxdata/influxdb-client>=1.35` — Direct InfluxDB queries

### Authentication & RBAC

All `/api/v1/*` routes require a valid JWT bearer token. The gateway enforces a four-tier role hierarchy:

| Role | Rank | Capabilities |
|---|---|---|
| `viewer` | 0 | Read-only access to all data |
| `operator` | 1 | Read all data, acknowledge alerts |
| `plant_manager` | 2 | Configure devices, alert thresholds, trigger ML training |
| `admin` | 3 | Full access, user management, device deletion |

A default `admin` user is seeded from `ADMIN_INITIAL_PASSWORD` on first startup.

### Routing Behavior

| Route prefix | Handled by | Auth required |
|---|---|---|
| `/auth/*` | Built-in auth routes | Public (login/refresh) or JWT (me/logout) |
| `/api/v1/admin/users*` | Built-in user CRUD | admin only |
| `/api/v1/devices*` | Built-in device CRUD + health | role-dependent (see API reference) |
| `/api/v1/machines*` | Direct InfluxDB queries | viewer+ |
| `/api/v1/energy*` | Proxy → `analytics:8081` | viewer+ |
| `/api/v1/ml*` | Proxy → `ml-service:8082` | viewer+ (train: plant_manager+) |
| `/api/v1/optimization*` | Proxy → `optimization:8083` | viewer+ |
| `/api/v1/alerts*` | Proxy → `alert-service:8084` | viewer+ |

### Data Storage

The gateway persists two SQLite tables in `data/platform.db` (mounted via `gateway-db` Docker volume):
- `users` — platform accounts with hashed passwords and roles
- `devices` — registered factory machines with configuration and per-device alert threshold overrides

### Response Envelope
All responses follow a standard envelope:
```json
{
  "status": "success",
  "data": { }
}
```
Errors:
```json
{
  "status": "error",
  "error": {
    "code": "NOT_FOUND",
    "message": "Machine CNC_99 not found"
  }
}
```

---

## 8. energy-frontend

**Language**: TypeScript
**Framework**: React 18 + Vite 6
**Role**: Operations dashboard SPA — authentication, device management, analytics views, embedded Grafana panels, admin
**Docker service name**: `frontend`
**Exposed port**: 3001 (nginx)

### Key Dependencies
- `react>=18`, `react-router-dom>=6` — SPA framework and routing
- `@tanstack/react-query>=5` — Server state management with automatic polling
- `zustand>=5` — Client auth state with localStorage persistence
- `axios` — HTTP client with JWT attach + 401 refresh-and-retry interceptors
- `tailwindcss>=4` — Utility-first styling (via `@tailwindcss/vite`)
- `react-hook-form>=7`, `zod>=3` — Form validation
- `recharts>=2` — Charts for non-Grafana visualizations

### Pages

| Route | Min Role | Description |
|---|---|---|
| `/login` | public | JWT login form |
| `/` | viewer | Dashboard: Grafana overview panel + live status cards |
| `/devices` | viewer | Device table with health badges |
| `/devices/new` | plant_manager | Register a new device |
| `/devices/:id` | viewer | Device detail + Grafana panel embed |
| `/devices/:id/edit` | plant_manager | Edit device configuration |
| `/energy` | viewer | Energy analytics + cost summary |
| `/ml` | viewer | Anomaly timeline, forecast, efficiency table |
| `/optimization` | viewer | Optimization recommendation cards |
| `/alerts` | viewer | Active alerts + history (operator can acknowledge) |
| `/alerts/rules` | plant_manager | Per-device alert threshold editor |
| `/admin/users` | admin | User management table |
| `/admin/settings` | admin | Factory name and tariff configuration |

### nginx Proxy

In production (Docker), nginx on port 3001 proxies `/api/`, `/auth/`, and `/ws/` to `api-gateway:8080`. All unmatched routes fall back to `index.html` for React Router.

### Grafana Embedding

Grafana panels are embedded as `<iframe>` elements using the `kiosk` query parameter. Requires `GF_AUTH_ANONYMOUS_ENABLED=true` and `GF_SECURITY_ALLOW_EMBEDDING=true` on the Grafana container (both set in `docker-compose.yml`).

---

## 9. Infrastructure Services

### Mosquitto (Eclipse Mosquitto 2)

**Role**: MQTT message broker
**Ports**: 1883 (MQTT), 9001 (MQTT over WebSocket)
**Auth**: Anonymous access enabled (no credentials required in dev)
**Persistence**: `mosquitto-data` volume

### InfluxDB 2.x

**Role**: Time-series database for all sensor and derived data
**Port**: 8086
**Web UI**: http://localhost:8086
**Org**: `energy-platform`
**Bucket**: `energy_data`
**Retention**: 30 days
**Token**: `energy-dev-token-2026` (admin, read/write)
**Credentials**: `admin` / `energy-admin-2026`
**Persistence**: `influxdb-data` volume

### Grafana 11.x

**Role**: Real-time visualization dashboard
**Port**: 3000
**URL**: http://localhost:3000
**Credentials**: `admin` / `energy-admin`
**Pre-provisioned**: InfluxDB datasource + "Machine Energy Overview" dashboard
**Persistence**: `grafana-data` volume
