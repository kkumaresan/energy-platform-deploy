# API Reference

All endpoints are served by the API gateway at `http://localhost:8080`.

Base path: `/api/v1`

---

## Authentication

The platform uses JWT bearer tokens. Obtain a token via `POST /auth/login`, then pass it in every subsequent request header:

```
Authorization: Bearer <accessToken>
```

Access tokens expire in **15 minutes**. Use `POST /auth/refresh` to obtain a new one without re-entering credentials.

### `POST /auth/login`
Authenticate and receive tokens. **Public — no auth required.**

**Request body**
```json
{ "username": "admin", "password": "admin-change-me" }
```

**Response**
```json
{
  "status": "success",
  "data": {
    "accessToken": "<jwt>",
    "refreshToken": "<jwt>",
    "user": { "id": 1, "username": "admin", "email": null, "role": "admin" }
  }
}
```

### `POST /auth/refresh`
Exchange a refresh token for a new access token. **Public — no auth required.**

**Request body**
```json
{ "refreshToken": "<jwt>" }
```

**Response**
```json
{ "status": "success", "data": { "accessToken": "<new-jwt>" } }
```

### `POST /auth/logout`
Client-side only in this MVP (tokens are stateless). Returns 200. **Requires auth.**

### `GET /auth/me`
Returns the currently authenticated user profile. **Requires auth.**

**Response**
```json
{ "status": "success", "data": { "id": 1, "username": "admin", "role": "admin" } }
```

---

## Admin — User Management

> Requires role: **admin**

### `GET /api/v1/admin/users`
List all platform users.

### `POST /api/v1/admin/users`
Create a new user.

**Request body**
```json
{ "username": "jane", "password": "secret", "email": "jane@example.com", "role": "operator" }
```
Roles: `admin`, `plant_manager`, `operator`, `viewer`

### `PATCH /api/v1/admin/users/:id`
Update a user's role or email.

**Request body** — any subset of `{ "role", "email" }`

### `DELETE /api/v1/admin/users/:id`
Delete a user. Cannot delete the last admin.

---

## Devices

> All device endpoints require auth. Minimum role noted per endpoint.

### `GET /api/v1/devices`
List all registered devices. **Role: viewer+**

**Response**
```json
{
  "status": "success",
  "data": [
    {
      "machine_id": "CNC_01",
      "factory_id": "factory_01",
      "machine_type": "cnc",
      "rated_power_kw": 15,
      "collection_mode": "simulator",
      "alert_overrides": null,
      "created_at": "2026-03-01T08:00:00Z",
      "updated_at": "2026-03-01T08:00:00Z"
    }
  ]
}
```

### `POST /api/v1/devices`
Register a new device. **Role: plant_manager+**

**Request body**
```json
{
  "machine_id": "CNC_03",
  "factory_id": "factory_01",
  "machine_type": "cnc",
  "rated_power_kw": 18,
  "collection_mode": "simulator"
}
```

### `GET /api/v1/devices/:id`
Single device details. **Role: viewer+**

### `PATCH /api/v1/devices/:id`
Update device configuration or collection mode. **Role: plant_manager+**

**Request body** — any subset of `{ "machine_type", "rated_power_kw", "collection_mode" }`

### `DELETE /api/v1/devices/:id`
Deregister a device. **Role: admin**

### `GET /api/v1/devices/:id/health`
Returns live connectivity status based on last InfluxDB reading. **Role: viewer+**

**Response**
```json
{
  "status": "success",
  "data": {
    "machine_id": "CNC_01",
    "lastSeen": "2026-03-04T10:29:55Z",
    "isOnline": true,
    "gapMinutes": 0.08
  }
}
```
`isOnline` is `true` if `gapMinutes` ≤ 2.

### `PATCH /api/v1/devices/:id/thresholds`
Override alert thresholds for a specific device. **Role: plant_manager+**

**Request body** — key/value threshold overrides stored as JSON:
```json
{ "power_spike_ratio": 1.8, "idle_power_pct": 0.25 }
```

---

## Health

> All `/api/v1/*` endpoints require `Authorization: Bearer <token>` unless stated otherwise.



### `GET /api/v1/health`
Gateway health check.

**Response**
```json
{ "status": "success", "data": { "service": "api-gateway", "status": "ok" } }
```

---

## Machines

### `GET /api/v1/machines`
List all machines with current power reading and status.

**Response**
```json
{
  "status": "success",
  "data": [
    {
      "machine_id": "CNC_01",
      "machine_type": "cnc",
      "factory_id": "factory_01",
      "current_power_kw": 12.4,
      "status": "running",
      "last_seen": "2026-03-04T10:30:00Z"
    }
  ]
}
```

### `GET /api/v1/machines/:id`
Single machine details.

### `GET /api/v1/machines/:id/energy`
Raw energy readings for a machine.

**Query params**
| Param | Default | Description |
|---|---|---|
| `start` | `-1h` | InfluxDB relative or absolute time |

---

## Energy Analytics

> Proxied to `analytics:8081`

### `GET /api/v1/energy/usage`
Raw energy usage from InfluxDB.

**Query params**
| Param | Default | Description |
|---|---|---|
| `period` | `hourly` | `hourly` or `daily` |
| `start` | `-24h` | Start time |

### `GET /api/v1/energy/hourly`
Mean power per machine aggregated by hour.

**Query params**: `start` (default: `-2h`)

**Response**
```json
{
  "status": "success",
  "data": [
    { "machine_id": "CNC_01", "hour": "2026-03-04T09:00:00Z", "mean_power_kw": 11.2, "energy_kwh": 11.2 }
  ]
}
```

### `GET /api/v1/energy/daily`
Total kWh per machine per day.

**Query params**: `start` (default: `-25h`)

### `GET /api/v1/energy/shift`
Energy consumption broken down by shift (day/evening/night).

**Query params**: `start` (default: `-25h`)

**Shifts**
- Day: 06:00–14:00
- Evening: 14:00–22:00
- Night: 22:00–06:00

### `GET /api/v1/energy/idle`
Machines currently consuming below 15% of their rated power while in a non-off state.

**Query params**: `start` (default: `-1h`)

### `GET /api/v1/energy/peak-demand`
Peak instantaneous demand per machine within the time window.

**Query params**: `start` (default: `-24h`)

### `GET /api/v1/energy/cost`
Estimated energy cost using time-of-use tariff rates.

**Query params**: `start` (default: `-24h`)

**Response includes**
- `peak_cost` — Cost during peak hours (09:00–17:00) at £0.15/kWh
- `offpeak_cost` — Cost during off-peak hours at £0.08/kWh
- `total_cost`

---

## ML Models

> Proxied to `ml-service:8082`

### `POST /api/v1/ml/train`
Train all ML models (anomaly + forecast) for all machines.

**Query params**: `start` (default: `-7d`)

**Note**: Requires at least ~2 minutes of live data to be available in InfluxDB.

### `POST /api/v1/ml/train/anomaly/:machine_id`
Train anomaly detection model for a specific machine.

**Query params**: `start` (default: `-7d`)

### `POST /api/v1/ml/train/forecast/:machine_id`
Train Prophet forecast model for a specific machine.

**Query params**: `start` (default: `-7d`)

### `GET /api/v1/ml/anomalies`
Anomaly detection results across all machines.

**Query params**: `start` (default: `-1h`)

**Response**
```json
{
  "status": "success",
  "data": [
    {
      "machine_id": "CNC_01",
      "timestamp": "2026-03-04T10:25:00Z",
      "power_kw": 24.1,
      "anomaly_score": -0.32,
      "is_anomaly": true
    }
  ]
}
```

### `GET /api/v1/ml/anomalies/:machine_id`
Anomalies for a specific machine.

**Query params**: `start` (default: `-1h`)

### `GET /api/v1/ml/forecast/:machine_id`
24-hour energy consumption forecast.

**Query params**
| Param | Default | Description |
|---|---|---|
| `horizon` | `24` | Forecast horizon in hours |

**Response**
```json
{
  "status": "success",
  "data": {
    "machine_id": "CNC_01",
    "forecast": [
      { "timestamp": "2026-03-04T11:00:00Z", "predicted_power_kw": 11.8, "lower": 10.2, "upper": 13.4 }
    ]
  }
}
```

### `GET /api/v1/ml/efficiency`
Efficiency scores for all machines.

### `GET /api/v1/ml/efficiency/:machine_id`
Efficiency score for a specific machine.

**Response**
```json
{
  "status": "success",
  "data": {
    "machine_id": "CNC_01",
    "efficiency_score": 0.82,
    "rated_power_kw": 15.0,
    "avg_power_kw": 12.3
  }
}
```

---

## Optimization

> Proxied to `optimization:8083`

### `GET /api/v1/optimization/suggestions`
All optimization recommendations across all strategies.

**Query params**: `range` (default: `-24h`)

**Response**
```json
{
  "status": "success",
  "data": [
    {
      "machine_id": "COMP_01",
      "type": "compressor",
      "title": "Optimize load/unload cycle",
      "description": "COMP_01 is cycling too frequently. Consider adjusting pressure band.",
      "estimated_savings_kwh": 4.2,
      "priority": "high"
    }
  ]
}
```

### `GET /api/v1/optimization/compressor`
Compressor-specific optimization analysis for all compressors.

**Query params**: `range` (default: `-24h`)

### `GET /api/v1/optimization/compressor/:machine_id`
Compressor analysis for a specific machine.

**Query params**: `range` (default: `-24h`)

### `GET /api/v1/optimization/schedule`
Shift schedule optimization suggestions (off-peak operation recommendations).

**Query params**: `range` (default: `-24h`)

### `GET /api/v1/optimization/peak-reduction`
Peak demand reduction strategies using time-of-use tariff analysis.

**Query params**: `range` (default: `-24h`)

---

## Alerts

> Proxied to `alert-service:8084`

### `GET /api/v1/alerts`
All currently active (unresolved) alerts.

**Response**
```json
{
  "status": "success",
  "data": [
    {
      "id": "alert-abc123",
      "rule": "power_factor_drop",
      "machine_id": "CNC_01",
      "severity": "critical",
      "triggered_at": "2026-03-04T10:15:00Z",
      "message": "Power factor 0.81 below threshold 0.85"
    }
  ]
}
```

### `GET /api/v1/alerts/history`
Historical alert log.

**Query params**
| Param | Default | Description |
|---|---|---|
| `limit` | `50` | Max number of records |
| `severity` | *(all)* | Filter by `info`, `warning`, or `critical` |

### `GET /api/v1/alerts/machine/:machine_id`
Alerts (active and recent) for a specific machine.

### `GET /api/v1/alerts/rules`
List all configured alert rules.

**Response**
```json
{
  "status": "success",
  "data": [
    {
      "name": "power_factor_drop",
      "condition": "power_factor_below",
      "threshold": 0.85,
      "duration_seconds": 600,
      "severity": "critical",
      "channels": ["console", "webhook"]
    }
  ]
}
```

### `GET /api/v1/alerts/summary`
Aggregated alert statistics.

---

## Error Responses

| HTTP Status | When |
|---|---|
| 400 | Invalid query parameters or request body |
| 401 | Missing or invalid JWT token |
| 403 | Authenticated but insufficient role |
| 404 | Machine ID or device not found |
| 503 | Upstream service unavailable |
| 500 | Internal server error |

All errors follow the standard envelope:
```json
{
  "status": "error",
  "error": {
    "code": "SERVICE_UNAVAILABLE",
    "message": "analytics service is not responding"
  }
}
```
