# Roadmap & Future Considerations

## Post-MVP Priorities

| Item | Description | Phase |
|---|---|---|
| Real sensor integration | Add Modbus TCP/RTU and OPC-UA clients to `energy-data-collector` for live factory hardware | Phase 5 |
| k3s migration | Move from Docker Compose to lightweight Kubernetes (k3s) for multi-factory deployments | Phase 6 |
| JWT authentication | Add JWT-based auth to the API gateway (middleware scaffold exists at `src/middleware/auth.ts`) | Phase 6 |
| Rate limiting | API gateway rate limiting per client | Phase 6 |
| WebSocket alerts | Real-time alert stream via WebSocket (`/ws/alerts`) — gateway has `@fastify/websocket` installed | Phase 6 |

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

- No authentication on any endpoint — all APIs are open
- Alert history is in-memory only — lost on container restart
- ML models must be retrained manually via API call — no scheduled retraining
- Single factory only — no tenant/site isolation
- `energy-prototype` repo (retired) contained original architecture planning docs — now consolidated into `energy-platform-deploy/docs/`
