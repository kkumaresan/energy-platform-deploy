# Configuration Reference

All configuration is supplied via environment variables. The `.env.example` file in the root of `energy-platform-deploy` provides a template for the `.env` file used by Docker Compose.

---

## Shared Infrastructure

These variables are consumed by the Docker-managed InfluxDB and Grafana containers directly.

| Variable | Default | Description |
|---|---|---|
| `DOCKER_INFLUXDB_INIT_USERNAME` | `admin` | InfluxDB admin username |
| `DOCKER_INFLUXDB_INIT_PASSWORD` | `energy-admin-2026` | InfluxDB admin password |
| `DOCKER_INFLUXDB_INIT_ORG` | `energy-platform` | InfluxDB organisation name |
| `DOCKER_INFLUXDB_INIT_BUCKET` | `energy_data` | Default bucket |
| `DOCKER_INFLUXDB_INIT_ADMIN_TOKEN` | `energy-dev-token-2026` | Admin API token |
| `DOCKER_INFLUXDB_INIT_RETENTION` | `30d` | Data retention period |
| `GF_SECURITY_ADMIN_USER` | `admin` | Grafana admin username |
| `GF_SECURITY_ADMIN_PASSWORD` | `energy-admin` | Grafana admin password |
| `GF_SECURITY_ALLOW_EMBEDDING` | `true` | Allows Grafana panels to be embedded via `<iframe>` in the frontend |
| `GF_AUTH_ANONYMOUS_ENABLED` | `true` | Enables anonymous (unauthenticated) viewer access — required for iframe embedding |
| `GF_AUTH_ANONYMOUS_ORG_NAME` | `Main Org.` | Grafana org for anonymous sessions |
| `GF_AUTH_ANONYMOUS_ORG_ROLE` | `Viewer` | Role granted to anonymous sessions |

---

## energy-data-collector

Prefix: `COLLECTOR_`

| Variable | Default | Description |
|---|---|---|
| `COLLECTOR_MODE` | `simulator` | Collection mode: `simulator`, `modbus`, `opc-ua` |
| `COLLECTOR_MQTT_HOST` | `mosquitto` | MQTT broker hostname |
| `COLLECTOR_MQTT_PORT` | `1883` | MQTT broker port |
| `COLLECTOR_MQTT_CLIENT_ID` | `energy-data-collector` | MQTT client identifier |
| `COLLECTOR_MQTT_TOPIC_PREFIX` | `factory` | Root MQTT topic prefix |
| `COLLECTOR_FACTORY_ID` | `factory_01` | Factory identifier (appears in MQTT topics and payloads) |
| `COLLECTOR_COLLECT_INTERVAL_SECONDS` | `5` | Interval between sensor readings (seconds) |
| `COLLECTOR_MACHINES_CONFIG` | `src/config/machines.yaml` | Path to machine registry YAML |

---

## energy-ingestion-service

Prefix: `INGESTION_`

| Variable | Default | Description |
|---|---|---|
| `INGESTION_MQTT_HOST` | `mosquitto` | MQTT broker hostname |
| `INGESTION_MQTT_PORT` | `1883` | MQTT broker port |
| `INGESTION_MQTT_CLIENT_ID` | `energy-ingestion-service` | MQTT client identifier |
| `INGESTION_MQTT_TOPIC` | `factory/+/machines/+/power` | MQTT subscription topic (wildcard) |
| `INGESTION_INFLUXDB_URL` | `http://influxdb:8086` | InfluxDB base URL |
| `INGESTION_INFLUXDB_TOKEN` | `energy-dev-token-2026` | InfluxDB API token |
| `INGESTION_INFLUXDB_ORG` | `energy-platform` | InfluxDB organisation |
| `INGESTION_INFLUXDB_BUCKET` | `energy_data` | InfluxDB bucket |
| `INGESTION_BATCH_SIZE` | `50` | Max points per write batch |
| `INGESTION_FLUSH_INTERVAL_MS` | `5000` | Flush interval in milliseconds |

---

## energy-analytics-service

Prefix: `ANALYTICS_`

| Variable | Default | Description |
|---|---|---|
| `ANALYTICS_INFLUXDB_URL` | `http://influxdb:8086` | InfluxDB base URL |
| `ANALYTICS_INFLUXDB_TOKEN` | `energy-dev-token-2026` | InfluxDB API token |
| `ANALYTICS_INFLUXDB_ORG` | `energy-platform` | InfluxDB organisation |
| `ANALYTICS_INFLUXDB_BUCKET` | `energy_data` | InfluxDB bucket |
| `ANALYTICS_API_HOST` | `0.0.0.0` | API bind address |
| `ANALYTICS_API_PORT` | `8081` | API listen port |
| `ANALYTICS_TARIFF_PEAK_RATE` | `0.15` | Peak electricity tariff (£/kWh) |
| `ANALYTICS_TARIFF_OFFPEAK_RATE` | `0.08` | Off-peak electricity tariff (£/kWh) |
| `ANALYTICS_TARIFF_PEAK_START_HOUR` | `9` | Peak period start hour (24h, UTC) |
| `ANALYTICS_TARIFF_PEAK_END_HOUR` | `17` | Peak period end hour (24h, UTC) |
| `ANALYTICS_IDLE_POWER_THRESHOLD_PCT` | `0.15` | Fraction of rated power below which a machine is considered idle |

---

## energy-ml-service

Prefix: `ML_`

| Variable | Default | Description |
|---|---|---|
| `ML_INFLUXDB_URL` | `http://influxdb:8086` | InfluxDB base URL |
| `ML_INFLUXDB_TOKEN` | `energy-dev-token-2026` | InfluxDB API token |
| `ML_INFLUXDB_ORG` | `energy-platform` | InfluxDB organisation |
| `ML_INFLUXDB_BUCKET` | `energy_data` | InfluxDB bucket |
| `ML_API_HOST` | `0.0.0.0` | API bind address |
| `ML_API_PORT` | `8082` | API listen port |
| `ML_MODEL_ARTIFACTS_DIR` | `/app/model_artifacts` | Directory for persisted model files |
| `ML_ANOMALY_CONTAMINATION` | `0.05` | Expected fraction of anomalies (Isolation Forest) |
| `ML_ANOMALY_WINDOW_SIZE` | `12` | Rolling window size for anomaly feature computation |
| `ML_FORECAST_HORIZON_HOURS` | `24` | Default forecast horizon in hours |
| `ML_FORECAST_MIN_TRAINING_DAYS` | `1` | Minimum days of data required to train forecast model |
| `ML_EFFICIENCY_LOOKBACK` | `-24h` | InfluxDB relative time for efficiency scoring |

---

## energy-optimization-service

Prefix: `OPTIMIZATION_`

| Variable | Default | Description |
|---|---|---|
| `OPTIMIZATION_INFLUXDB_URL` | `http://influxdb:8086` | InfluxDB base URL |
| `OPTIMIZATION_INFLUXDB_TOKEN` | `energy-dev-token-2026` | InfluxDB API token |
| `OPTIMIZATION_INFLUXDB_ORG` | `energy-platform` | InfluxDB organisation |
| `OPTIMIZATION_INFLUXDB_BUCKET` | `energy_data` | InfluxDB bucket |
| `OPTIMIZATION_API_HOST` | `0.0.0.0` | API bind address |
| `OPTIMIZATION_API_PORT` | `8083` | API listen port |
| `OPTIMIZATION_TARIFF_PEAK_RATE` | `0.15` | Peak electricity tariff (£/kWh) |
| `OPTIMIZATION_TARIFF_OFFPEAK_RATE` | `0.08` | Off-peak electricity tariff (£/kWh) |
| `OPTIMIZATION_TARIFF_PEAK_START_HOUR` | `9` | Peak period start hour |
| `OPTIMIZATION_TARIFF_PEAK_END_HOUR` | `17` | Peak period end hour |

---

## energy-alert-service

Prefix: `ALERT_`

| Variable | Default | Description |
|---|---|---|
| `ALERT_INFLUXDB_URL` | `http://influxdb:8086` | InfluxDB base URL |
| `ALERT_INFLUXDB_TOKEN` | `energy-dev-token-2026` | InfluxDB API token |
| `ALERT_INFLUXDB_ORG` | `energy-platform` | InfluxDB organisation |
| `ALERT_INFLUXDB_BUCKET` | `energy_data` | InfluxDB bucket |
| `ALERT_API_HOST` | `0.0.0.0` | API bind address |
| `ALERT_API_PORT` | `8084` | API listen port |
| `ALERT_CHECK_INTERVAL_SECONDS` | `30` | How often to evaluate alert rules |
| `ALERT_HISTORY_MAX` | `1000` | Maximum alert history entries to retain in memory |
| `ALERT_WEBHOOK_URL` | *(unset)* | HTTP endpoint for webhook notifications |
| `ALERT_WEBHOOK_ENABLED` | `false` | Enable webhook notifications |

---

## energy-api-gateway

No prefix — variables are used directly.

| Variable | Default | Description |
|---|---|---|
| `API_PORT` | `8080` | Gateway listen port |
| `API_HOST` | `0.0.0.0` | Gateway bind address |
| `INFLUXDB_URL` | `http://influxdb:8086` | InfluxDB base URL |
| `INFLUXDB_TOKEN` | `energy-dev-token-2026` | InfluxDB API token |
| `INFLUXDB_ORG` | `energy-platform` | InfluxDB organisation |
| `INFLUXDB_BUCKET` | `energy_data` | InfluxDB bucket |
| `ANALYTICS_SERVICE_URL` | `http://analytics:8081` | Analytics service base URL |
| `ML_SERVICE_URL` | `http://ml-service:8082` | ML service base URL |
| `OPTIMIZATION_SERVICE_URL` | `http://optimization:8083` | Optimization service base URL |
| `ALERT_SERVICE_URL` | `http://alert-service:8084` | Alert service base URL |
| `CORS_ORIGIN` | `*` | Allowed CORS origin(s) |
| `JWT_SECRET` | `dev-secret-change-in-prod` | Secret used to sign/verify JWTs. **Must be changed in production.** |
| `ADMIN_INITIAL_PASSWORD` | `admin-change-me` | Password for the seeded `admin` account on first startup. Ignored on subsequent starts. |
| `DB_PATH` | `./data/platform.db` | Path to the SQLite database file (users + devices). Mounted via `gateway-db` Docker volume. |

---

## energy-frontend

Build-time arguments passed via `docker build --build-arg` (or `args:` in `docker-compose.yml`).

| Variable | Default | Description |
|---|---|---|
| `VITE_API_URL` | `/` | Base URL for REST API calls. Empty string or `/` means same-origin (nginx proxies `/api` and `/auth` to the gateway). |
| `VITE_GRAFANA_URL` | `http://localhost:3000` | Public URL of the Grafana instance used for iframe embedding. |
| `VITE_WS_URL` | `ws://localhost:8080` | WebSocket base URL for the live alert stream. |

---

## Notes

- All secrets (tokens, passwords) in `.env.example` are **development defaults**. Replace them before any non-local deployment.
- The InfluxDB token `energy-dev-token-2026` is statically configured across all services. In production, use per-service tokens with minimal required permissions (read vs. write).
- Tariff rates default to UK market rates (£). Update `TARIFF_PEAK_RATE`, `TARIFF_OFFPEAK_RATE`, and peak hours for other markets or TOU tariff structures.
