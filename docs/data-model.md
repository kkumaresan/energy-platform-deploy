# Data Model

## MQTT Message Schema

Published by `energy-data-collector` to `factory/{factory_id}/machines/{machine_id}/power`.

```json
{
  "factory_id": "factory_01",
  "machine_id": "CNC_01",
  "machine_type": "cnc",
  "timestamp": "2026-03-04T10:30:00Z",
  "metrics": {
    "power_kw": 12.5,
    "voltage": 415.2,
    "current": 17.5,
    "power_factor": 0.92,
    "temperature": 45.3,
    "pressure": null,
    "flow_rate": null
  },
  "status": "running"
}
```

### Field Definitions

| Field | Type | Required | Description |
|---|---|---|---|
| `factory_id` | string | Yes | Factory identifier |
| `machine_id` | string | Yes | Unique machine identifier |
| `machine_type` | string | No | One of: `cnc`, `compressor`, `hvac`, `pump`, `conveyor`, `lighting`, `other` |
| `timestamp` | ISO 8601 string | Yes | UTC timestamp of reading |
| `metrics.power_kw` | float ≥ 0 | Yes | Active power in kilowatts |
| `metrics.voltage` | float | No | RMS voltage (volts) |
| `metrics.current` | float | No | RMS current (amps) |
| `metrics.power_factor` | float 0–1 | No | Power factor |
| `metrics.temperature` | float | No | Machine temperature (°C) |
| `metrics.pressure` | float | No | Compressor pressure (bar) — compressors only |
| `metrics.flow_rate` | float | No | Compressor flow rate (m³/min) — compressors only |
| `status` | string | No | One of: `running`, `idle`, `off`, `fault`, `maintenance` |

---

## InfluxDB Measurements

**Database**: InfluxDB 2.x
**Org**: `energy-platform`
**Bucket**: `energy_data`
**Retention**: 30 days
**Query language**: Flux

### Measurement: `machine_power`

Written by `energy-ingestion-service` from validated MQTT messages.

| Field | Type | Description |
|---|---|---|
| **Tags** | | |
| `factory_id` | string | Factory identifier |
| `machine_id` | string | Machine identifier |
| `machine_type` | string | Machine category |
| **Fields** | | |
| `power_kw` | float | Active power (kW) |
| `voltage` | float | RMS voltage (V) |
| `current` | float | RMS current (A) |
| `power_factor` | float | Power factor |
| `temperature` | float | Temperature (°C) |
| `pressure` | float | Pressure (bar) |
| `flow_rate` | float | Flow rate (m³/min) |

### Measurement: `machine_status`

| Field | Type | Description |
|---|---|---|
| **Tags** | | |
| `factory_id` | string | Factory identifier |
| `machine_id` | string | Machine identifier |
| `machine_type` | string | Machine category |
| **Fields** | | |
| `state` | string | Machine state (running/idle/off/fault/maintenance) |

---

## Example Flux Queries

### Last 1 hour of power readings for all machines
```flux
from(bucket: "energy_data")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "machine_power")
  |> filter(fn: (r) => r._field == "power_kw")
```

### Mean power per machine over last 24 hours
```flux
from(bucket: "energy_data")
  |> range(start: -24h)
  |> filter(fn: (r) => r._measurement == "machine_power" and r._field == "power_kw")
  |> group(columns: ["machine_id"])
  |> mean()
```

### Latest status for each machine
```flux
from(bucket: "energy_data")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "machine_status")
  |> last()
```

---

## API Response Envelope

All gateway responses use this envelope:

```json
{
  "status": "success",
  "data": { },
  "meta": {
    "timestamp": "2026-03-04T10:30:00Z"
  }
}
```

Error responses:

```json
{
  "status": "error",
  "error": {
    "code": "NOT_FOUND",
    "message": "Machine CNC_99 not found"
  }
}
```

### `status` values
- `success` — Request completed successfully
- `error` — Request failed

### Optional `meta` fields
- `page`, `per_page`, `total` — Pagination (when applicable)
- `timestamp` — Response generation time

---

## Machine Specifications

Used by alert-service for threshold ratio calculations.

| Machine ID | Type | Rated Power (kW) |
|---|---|---|
| CNC_01 | cnc | 15.0 |
| CNC_02 | cnc | 15.0 |
| COMP_01 | compressor | 22.0 |
| COMP_02 | compressor | 22.0 |
| HVAC_01 | hvac | 10.0 |
| PUMP_01 | pump | 5.5 |
| CONV_01 | conveyor | 3.0 |
| LIGHT_01 | lighting | 2.0 |

> **Note**: Simulator uses slightly different rated values (CNC_02 = 18 kW, HVAC_01 = 30 kW) for simulation realism. Alert thresholds use the conservative values above.
