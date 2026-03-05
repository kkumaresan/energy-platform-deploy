# Grafana Dashboard Improvements — Implementation Plan

## Overview
Enhance existing dashboard + create 2 new dashboards, adding Infinity plugin for REST API queries and InfluxDB write-back for historical trending.

---

## Step 1: Infrastructure Changes

### 1a. Docker Compose — Install Infinity Plugin
**File:** `energy-platform-deploy/docker-compose.yml`
- Add `GF_PLUGINS_PREINSTALL: yesoreyeram-infinity-datasource` to grafana environment
- Add `depends_on` entries for `analytics`, `ml-service`, `optimization`, `alert-service` (so Grafana can reach them)

### 1b. Infinity Datasource Provisioning
**New file:** `energy-platform-deploy/grafana/provisioning/datasources/infinity.yaml`
- Type: `yesoreyeram-infinity-datasource`
- Name: "Energy APIs"
- Access: proxy
- Auth: none (internal Docker network)
- Allowed hosts: analytics:8081, ml-service:8082, optimization:8083, alert-service:8084

---

## Step 2: InfluxDB Write-Back (2 services)

### 2a. Alert Service — Write alert events
**Files to modify:**
- `energy-alert-service/src/storage/influxdb_client.py` — Add `InfluxDBWriter` class (or extend existing) with write capability
- `energy-alert-service/src/monitor/watcher.py` — Write to InfluxDB inside `_check_alerts()` when alerts trigger/resolve

**New InfluxDB measurement:** `alert_events`
- Tags: `rule_name`, `machine_id`, `severity`
- Fields: `value` (float), `threshold` (float), `event_type` (string: "triggered"/"resolved")
- Timestamp: alert's triggered_at or resolved_at

### 2b. ML Service — Write anomaly scores & efficiency scores
**Files to modify:**
- `energy-ml-service/src/storage/influxdb_client.py` — Add write capability
- `energy-ml-service/src/inference/predictor.py` — Write results after computing anomalies and efficiency

**New measurement:** `anomaly_scores`
- Tags: `machine_id`
- Fields: `anomaly_score` (float), `is_anomaly` (int 0/1), `power_kw` (float)
- Timestamp: from the detection reading

**New measurement:** `efficiency_scores`
- Tags: `machine_id`, `machine_type`
- Fields: `score` (float), `utilization` (float), `idle_ratio` (float), `power_factor_score` (float), `temperature_score` (float)
- Timestamp: now (time of computation)

### 2c. Optimization Service — SKIP for MVP
Recommendations are on-demand static analyses; no historical trending value.

---

## Step 3: Enhance Existing Dashboard — "Factory Energy Monitor"

**File:** `energy-platform-deploy/grafana/dashboards/machine-energy.json`

### Add Template Variables:
1. **machine_id** — multi-value query variable from InfluxDB:
   ```flux
   import "influxdata/influxdb/schema"
   schema.tagValues(bucket: "energy_data", tag: "machine_id")
   ```
2. **machine_type** — multi-value query variable:
   ```flux
   import "influxdata/influxdb/schema"
   schema.tagValues(bucket: "energy_data", tag: "machine_type")
   ```

### Update Existing Panels:
- Add `|> filter(fn: (r) => contains(value: r.machine_id, set: ${machine_id:json}))` to all queries (when variable selected)

### Add New Panels (InfluxDB Flux):
3. **Current (Amperes)** — timeseries, 12w, field: `current`, unit: amp
4. **Compressor Flow Rate** — timeseries, 12w, field: `flow_rate`, unit: CFM
5. **Idle Waste Indicator** — stat panel, InfluxDB query joining machine_status (idle) + machine_power (>0.5 kW), showing count of idle-but-consuming machines

---

## Step 4: New Dashboard — "Energy Analytics & Cost"

**New file:** `energy-platform-deploy/grafana/dashboards/energy-analytics.json`

### Row 1: Summary Stats (Infinity → analytics API)
| Panel | Type | Size | Source | Endpoint |
|-------|------|------|--------|----------|
| Total Energy (24h) | stat | 6w | Infinity | `/analytics/cost?start=-24h` → `$.total_kwh` |
| Total Cost (24h) | stat | 6w | Infinity | `/analytics/cost?start=-24h` → `$.total_cost` |
| Load Factor | gauge | 6w | Infinity | `/analytics/peak-demand?start=-24h` → `$.load_factor` |
| Peak Demand | stat | 6w | Infinity | `/analytics/peak-demand?start=-24h` → `$.peak_kw` |

### Row 2: Energy Trends (InfluxDB Flux)
| Panel | Type | Size | Source |
|-------|------|------|--------|
| Daily Energy by Machine | timeseries | 24w | InfluxDB — 1h aggregated windows, grouped by machine_id |

### Row 3: Cost & Shift Analysis (Infinity)
| Panel | Type | Size | Source | Endpoint |
|-------|------|------|--------|----------|
| Cost Breakdown by Machine | barchart | 12w | Infinity | `/analytics/cost?start=-24h` → `$.machines[*]` |
| Shift Energy Comparison | barchart | 12w | Infinity | `/analytics/energy/shift?start=-25h` → `$[*]` |

### Row 4: Idle & Peak (Infinity)
| Panel | Type | Size | Source | Endpoint |
|-------|------|------|--------|----------|
| Idle Machines | table | 12w | Infinity | `/analytics/idle?start=-1h` → `$[*]` |
| Peak Demand Windows | table | 12w | Infinity | `/analytics/peak-demand?start=-24h` → `$.top_windows[*]` |

---

## Step 5: New Dashboard — "Alerts & ML Insights"

**New file:** `energy-platform-deploy/grafana/dashboards/alerts-ml.json`

### Row 1: Alert Status (Infinity → alert API)
| Panel | Type | Size | Source | Endpoint |
|-------|------|------|--------|----------|
| Critical Alerts | stat (red) | 6w | Infinity | `/alerts/summary` → `$.by_severity.critical` |
| Warning Alerts | stat (yellow) | 6w | Infinity | `/alerts/summary` → `$.by_severity.warning` |
| Info Alerts | stat (blue) | 6w | Infinity | `/alerts/summary` → `$.by_severity.info` |
| Total Active | stat | 6w | Infinity | `/alerts/summary` → `$.total_active` |

### Row 2: Active Alerts Table (Infinity)
| Panel | Type | Size | Source | Endpoint |
|-------|------|------|--------|----------|
| Active Alerts | table | 24w | Infinity | `/alerts/active` → `$.alerts[*]` — columns: severity, machine_id, title, value, threshold, triggered_at |

### Row 3: Alert Trend (InfluxDB — from write-back)
| Panel | Type | Size | Source |
|-------|------|------|--------|
| Alert Events Over Time | timeseries | 24w | InfluxDB `alert_events` measurement, count by severity per 15m window |

### Row 4: Efficiency Scores (Infinity → ML API)
| Panel | Type | Size | Source | Endpoint |
|-------|------|------|--------|----------|
| Machine Efficiency Scores | bargauge | 24w | Infinity | `/ml/efficiency` → `$[*]` — field: score, colored by threshold (>80 green, 60-80 yellow, <60 red) |

### Row 5: Efficiency Trend (InfluxDB — from write-back)
| Panel | Type | Size | Source |
|-------|------|------|--------|
| Efficiency Score Trend | timeseries | 24w | InfluxDB `efficiency_scores` measurement, grouped by machine_id |

### Row 6: Anomaly Overlay (InfluxDB — from write-back)
| Panel | Type | Size | Source |
|-------|------|------|--------|
| Anomaly Scores | timeseries | 24w | InfluxDB `anomaly_scores` measurement, with threshold line at detection boundary |

### Row 7: Optimization Recommendations (Infinity → optimization API)
| Panel | Type | Size | Source | Endpoint |
|-------|------|------|--------|----------|
| Optimization Suggestions | table | 24w | Infinity | `/optimization/suggestions?range=-24h` → `$.recommendations[*]` — columns: severity, machine_id, type, title, estimated_savings_kwh_per_day |

### Row 8: Alert History (Infinity)
| Panel | Type | Size | Source | Endpoint |
|-------|------|------|--------|----------|
| Recent Alert History | table | 24w | Infinity | `/alerts/history?limit=100` → `$.alerts[*]` — columns: severity, machine_id, rule_name, title, triggered_at, resolved_at |

---

## Step 6: Update SETUP.md
- Document Infinity plugin auto-installation
- Add new dashboard descriptions
- Note new InfluxDB measurements from write-back

---

## Files Changed Summary

| Repository | File | Action |
|------------|------|--------|
| energy-platform-deploy | docker-compose.yml | Modify (add plugin env + depends_on) |
| energy-platform-deploy | grafana/provisioning/datasources/infinity.yaml | **Create** |
| energy-platform-deploy | grafana/dashboards/machine-energy.json | Modify (add variables + 3 panels) |
| energy-platform-deploy | grafana/dashboards/energy-analytics.json | **Create** |
| energy-platform-deploy | grafana/dashboards/alerts-ml.json | **Create** |
| energy-platform-deploy | SETUP.md | Modify |
| energy-alert-service | src/storage/influxdb_client.py | Modify (add write) |
| energy-alert-service | src/monitor/watcher.py | Modify (write alert events) |
| energy-ml-service | src/storage/influxdb_client.py | Modify (add write) |
| energy-ml-service | src/inference/predictor.py | Modify (write scores) |

---

## Testing Plan
1. `docker compose up -d --build` — verify all 10 containers start
2. Open Grafana at localhost:3000 — verify Infinity plugin installed (Settings → Plugins)
3. Check "Energy APIs" datasource exists (Settings → Data Sources)
4. Dashboard 1: Verify machine_id/machine_type dropdowns work, new panels render
5. Dashboard 2: Verify Infinity panels load data from analytics API
6. Dashboard 3: Verify alert stats, efficiency scores, recommendations load via Infinity
7. Wait ~2 minutes for alert watcher to write events → verify alert trend chart populates
8. Call `POST /ml/train/all` then `GET /ml/efficiency` → verify efficiency trend populates
9. Call `GET /ml/anomalies` → verify anomaly scores appear in InfluxDB overlay panel
