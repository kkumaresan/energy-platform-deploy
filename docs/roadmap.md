# Roadmap & Future Considerations

## Completed

| Item | Description |
|---|---|
| JWT authentication | JWT auth + RBAC on the API gateway — access tokens (15 min) and refresh tokens (7 days) |
| Device management | Register, configure, health-check, and set per-device alert thresholds via REST API |
| Operations frontend | React SPA at port 3001 — login, dashboards, device management, energy analytics, alerts, admin |

## Post-MVP Priorities

| Item | Description | Phase |
|---|---|---|
| Real sensor integration | Add Modbus TCP/RTU and OPC-UA clients to `energy-data-collector` for live factory hardware | Phase 6 |
| k3s migration | Move from Docker Compose to lightweight Kubernetes (k3s) for multi-factory deployments | Phase 7 |
| Rate limiting | API gateway rate limiting per client | Phase 6 |
| WebSocket alerts | Real-time alert stream via WebSocket (`/ws/alerts`) push to frontend | Phase 6 |
| Persistent alert history | Store alert history in InfluxDB or SQLite instead of in-memory | Phase 6 |

## ML Model Improvements (v2)

| Model | Current (MVP) | Planned (v2) |
|---|---|---|
| Anomaly detection | Isolation Forest | Autoencoder (better on multivariate sensor data) |
| Energy forecasting | Prophet | LSTM (better on complex shift patterns) |
| Efficiency scoring | Statistical heuristic | XGBoost regression vs manufacturer baseline |
| Optimization | Rule engine + heuristics | Google OR-Tools / Pyomo constraint optimization |

## Alert Channel Expansion

The alert service currently supports `console` and `webhook`. Planned channels:
- Email (nodemailer)
- Slack (Slack SDK)
- WhatsApp / SMS

## Longer-Term Vision

- **Multi-tenant SaaS**: Tenant isolation, onboarding flow, usage billing — enabling the platform to serve multiple factories from a single deployment
- **Edge ML inference**: Run anomaly detection on-device using ONNX Runtime, reducing cloud dependency and latency
- **Digital twin**: 3D factory model with live energy consumption overlay
- **Carbon tracking**: CO₂ emissions reporting alongside energy cost, mapped to grid carbon intensity
- **Mobile app**: Operator-facing alerts and dashboards on iOS/Android
- **Energy trading / demand response**: Optimize machine scheduling against time-of-use tariffs and grid demand response signals
- **Data quality monitoring**: Detect sensor drift, missing readings, and calibration drift automatically

## Known Gaps (Current MVP)

- Alert history is in-memory only — lost on container restart
- ML models must be retrained manually via API call — no scheduled retraining
- Single factory only — no tenant/site isolation
- WebSocket `/ws/alerts` endpoint exists but push delivery to the frontend is not yet wired
- `energy-prototype` repo (retired) contained original architecture planning docs — now consolidated into `energy-platform-deploy/docs/`
