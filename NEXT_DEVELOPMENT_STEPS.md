# AquaFlow Next Development Steps

This roadmap is based on the real implementation state found in `C:\Users\Grous info\Downloads\pfe`, not only on the original planning documents.

## Guiding Principles

- Preserve the existing workflow builder/editor and extend it incrementally.
- Keep the current NestJS module structure and React/Redux/Argon frontend structure unless a change directly supports a missing feature.
- Prioritize integration gaps before adding advanced modules.
- Do not mark a roadmap item complete until backend, frontend, API/UI, and realtime behavior have been verified.

## Priority 0: Stabilize Existing Integration ✅ COMPLETE

**Completed 2026-05-10:**

1. ✅ Fixed workflow builder route wiring.
   - `pfe-frontend/src/routes.js` now renders `<BuilderPage />` on `/admin/builder`.

2. ✅ API/port documentation consistent.
   - Backend defaults to port `3001` with `/api` global prefix.
   - Frontend `apiClient` defaults to `http://localhost:3001/api`.
   - Documentation aligns with implementation.

3. ✅ Added `.env.example` files.
   - `pfe-backend/.env.example` with database, JWT, MQTT, FRONTEND_URL, and PORT.
   - `pfe-frontend/.env.example` with REACT_APP_API_URL, REACT_APP_WS_URL.

4. ✅ Build verification complete.
   - Backend: `npx.cmd tsc --noEmit` passes cleanly.
   - Frontend: `npm.cmd run build` succeeds.

**Not addressed (defer or reconsider):**

5. Health endpoint.
   - `GET /api/health` would be useful for deployment checks.
   - Defer to Priority 6 (deployment hardening) unless critical for current iteration.

## Priority 1: Complete Phase 1 Properly ✅ MOSTLY COMPLETE

### Backend Priorities

1. ✅ Validate Socket.IO authentication. **COMPLETE 2026-05-10**
   - `RealtimeGateway.handleConnection` now validates JWT token from handshake auth.
   - Invalid/missing tokens are rejected with clean disconnect.
   - See `src/realtime/realtime.gateway.ts`.

2. ✅ Wire MQTT ingestion into `IotService`. **COMPLETE 2026-05-10**
   - `MqttClient.handleMessage` parses `sensors/{sensorId}/data` topics.
   - Extracts numeric values and delegates to `IotService.processSensorData`.
   - Invalid payloads logged safely; see `src/iot/mqtt/mqtt.client.ts`.

3. ✅ Persist threshold alerts. **COMPLETE 2026-05-10**
   - `IotService` now creates persistent `Alert` records when thresholds violated.
   - `AlertsService.create` automatically broadcasts `alert-created` event.
   - Alert includes sensor/station/threshold context; see `src/iot/iot.service.ts`.

4. ✅ Implement refresh token support. **COMPLETE 2026-05-10**
   - Added `POST /api/auth/refresh`.
   - Frontend `apiClient` retries expired requests through the refresh endpoint.
   - Remaining hardening: persist/rotate refresh tokens server-side instead of treating them as stateless JWTs.

5. ⏳ Introduce TypeORM migrations.
   - Backend still uses `synchronize: true` for non-production.
   - Migration generation status is documented in `pfe-backend/src/database/migrations/README.md`.
   - **Recommend:** Generate a real initial migration from a clean database snapshot for all 9 entities.
   - Add migration runner to deployment process.

### Frontend Priorities

1. ✅ Normalize realtime event names. **IMPLICIT COMPLETE**
   - Backend now emits `alert-created` (not `threshold-alert`).
   - Frontend already listens for `alert-created`.
   - Alert slices (`alertsSlice`, `dashboardSlice`, `realtimeSlice`) handle updates consistently.

2. ✅ Improve current data display patterns. **PARTIAL COMPLETE 2026-05-10**
   - Dashboard now derives KPIs, station overview, and active alert feed from real stations/sensors/alerts data.
   - Stations and monitoring format numeric readings/capacity more cleanly.
   - Further retry buttons and richer error recovery remain useful.

3. ✅ Add role-aware UI behavior. **PARTIAL COMPLETE 2026-05-10**
   - Station create/edit hidden unless role is admin/operator.
   - Sensor create hidden unless role is admin/operator.
   - Alert acknowledge/resolve hidden unless role is admin/operator/technician.
   - Keep expanding this as new privileged actions are added.

## Priority 2: Finish Core CRUD Feature Modules

### Stations

1. Add Station Details page.
2. Add delete action in UI for admins.
3. Add station sensor/equipment summary.
4. Add `GET /api/stations/:id/analytics` or defer explicitly until analytics module.

### Sensors / Monitoring

1. Add Sensor Details page.
2. Add live sensor history chart.
3. Add edit/delete sensor UI.
4. Add filtering by station/type/status.
5. Display realtime update timestamps and threshold state.

### Alerts

1. Add alert details page.
2. Add filters for severity/status/station/date.
3. Add clear/delete endpoint and UI if required by business rules.
4. Connect threshold-generated alerts to persistent alert list.

### Maintenance

1. Add create maintenance form.
2. Add intervention details page.
3. Add technician assignment endpoint and UI.
4. Add status transition controls.
5. Add realtime event for maintenance status changes.

## Priority 3: Make Workflow Automation Durable and Industrial

1. Decide API naming and migrate intentionally.
   - Either keep `/api/flows` and update docs, or add `/api/workflows` as planned.

2. Replace in-memory `FlowsService` storage.
   - Use `Workflow` and `WorkflowExecution` entities.
   - Store graph JSON, status, createdBy, execution logs.

3. Add industrial frontend blocks.
   - `sensor-trigger`
   - `threshold-checker`
   - `alert-sender`
   - `maintenance-request`
   - `mqtt-publisher`
   - `email-notification`
   - `sms-notification`
   - `pump-control`
   - `analytics-processor`
   - `timer/scheduler`
   - `station-monitor`

4. Add backend execution handlers.
   - Implement one handler at a time, with tests.
   - Start with `threshold-checker`, `alert-sender`, `maintenance-request`, and `mqtt-publisher`.

5. Add workflow list/details/log pages.
   - Reuse the existing builder.
   - Add run history and manual execute controls.

6. Integrate workflows with realtime/MQTT.
   - Sensor update can trigger eligible workflows.
   - Workflow actions can create alerts, maintenance records, and MQTT publishes.

## Priority 4: Add Missing Roadmap Modules

### GIS Map

1. Add `modules/map`.
2. Use station latitude/longitude already present in backend entities.
3. Add live marker status updates via Socket.IO.
4. Add filters by status/type.

### Analytics

1. Add backend `analytics` module.
2. Start with pragmatic endpoints:
   - `GET /api/analytics/kpis`
   - `GET /api/analytics/trends`
   - `GET /api/analytics/anomalies`
3. Add frontend analytics page with trend charts and KPI panels.
4. Defer predictive maintenance until enough data exists.

### Reports

1. Add backend `reports` module.
2. Start with CSV/Excel export before PDF if speed matters.
3. Add frontend reports list/builder.
4. Add templates for station summary, alerts, maintenance, and sensor trends.

### Notifications

1. Add notification service using existing `Notification` entity.
2. Implement in-app notifications first.
3. Add email/SMS channels later behind provider interfaces.
4. Add notification preferences UI.

### IoT Device Management

1. Add device registry entity/module if device identity is distinct from sensors.
2. Add UI for device connection status.
3. Track MQTT heartbeats from `devices/+/heartbeat`.
4. Add last-seen and offline detection.

## Priority 5: Testing and Quality

1. Backend tests.
   - Unit tests for auth, CRUD services, IoT parsing, workflow handlers.
   - Integration tests for protected endpoints.

2. Frontend tests.
   - Component tests for core pages.
   - Redux slice tests for realtime updates.

3. E2E smoke tests.
   - Login.
   - Create station.
   - Create sensor.
   - Simulate sensor update.
   - Verify alert/dashboard update.

4. Build/lint gates.
   - Backend: `npx.cmd tsc --noEmit`.
   - Frontend: `npm.cmd run build`.
   - Fix current frontend warnings:
     - unused imports in `src/components/Headers/Header.js`
     - unused `BuilderPage` import in `src/routes.js`

## Priority 6: Deployment Hardening

1. Add backend and frontend Dockerfiles.
2. Add production compose or deployment manifests.
3. Add migration workflow for deployment.
4. Add Redis-backed Socket.IO adapter before horizontal scaling.
5. Add indexes and retention strategy for `SensorData`.
6. Add structured logging and request correlation.
7. Add health/ready endpoints.
8. Add basic metrics for API latency, MQTT messages, socket clients, and alert creation.

## Recommended Immediate Sprint (Next Sessions)

**Phase 1A: Stabilization** ✅ COMPLETE (2026-05-10)
1. ✅ Fix `/admin/builder` route rendering.
2. ✅ Add `.env.example` files matching real ports.
3. ✅ Implement socket JWT validation.
4. ✅ Wire MQTT messages to `IotService.processSensorData`.
5. ✅ Persist threshold alerts and emit `alert-created`.

**Phase 1B: Complete Phase 1 Cleanup (Recommended Next)**
1. ✅ Implement `POST /api/auth/refresh` endpoint.
   - Added refresh token DTO and service method.
   - Updated frontend `apiClient` interceptor to call refresh on 401.
   - Verified refresh flow through authenticated API call.

2. Create initial TypeORM migration.
   - Migration generation is documented as blocked until it is generated from a clean database snapshot.
   - Next: add TypeORM migration scripts and generate the initial migration.

3. ✅ Add role-aware UI to current critical actions.
   - Station create/edit: admin/operator.
   - Sensor create: admin/operator.
   - Alert acknowledge/resolve: admin/operator/technician.
   - Next: apply the same pattern to future delete/assignment actions.

4. ✅ Improve seeded-data display.
   - Dashboard now uses real API data instead of hardcoded placeholder data.
   - Next: add retry buttons and richer page-level error recovery.

**Phase 2: Core CRUD Feature Completeness (After Phase 1B)**
1. Add Station Details page (`/admin/stations/:id`).
   - Display station metadata, sensors, recent alerts.
   - Add edit/delete actions.

2. Add Sensor Details page with live chart.
   - Show sensor readings over time (last 24h).
   - Display threshold violations.
   - Allow threshold adjustment (for admins).

3. Add Alert Details page.
   - Show full alert context, linked sensor/station.
   - Timeline of acknowledgments/resolutions.
   - Manual resolution form.

4. Add Maintenance create/assignment UI.
   - Create intervention form.
   - Assign to technician dropdown.
   - Track status transitions.
