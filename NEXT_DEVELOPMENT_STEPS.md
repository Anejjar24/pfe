# AquaFlow Next Development Steps

This roadmap is based on the real implementation state found in `C:\Users\Grous info\Downloads\pfe`, not only on the original planning documents.

## Guiding Principles

- Preserve the existing workflow builder/editor and extend it incrementally.
- Keep the current NestJS module structure and React/Redux/Argon frontend structure unless a change directly supports a missing feature.
- Prioritize integration gaps before adding advanced modules.
- Do not mark a roadmap item complete until backend, frontend, API/UI, and realtime behavior have been verified.

## Priority 0: Stabilize Existing Integration

1. Fix workflow builder route wiring.
   - In `pfe-frontend/src/routes.js`, `/admin/builder` currently imports `BuilderPage` but renders `Test`.
   - Route should render the real builder unless `Test` is intentionally a diagnostic screen.

2. Make API/port documentation consistent.
   - Actual backend defaults to `3001` and `/api`.
   - Frontend `apiClient` defaults to `http://localhost:3001/api`.
   - Update documentation and env examples to match the current application, or intentionally change code and docs together.

3. Add `.env.example` files.
   - Backend: database, JWT, MQTT, frontend URL, port.
   - Frontend: `REACT_APP_API_URL`, `REACT_APP_WS_URL`, optional workflow API URL.

4. Remove or ignore generated artifacts.
   - Confirm `pfe-backend/dist` and `pfe-frontend/build` are ignored if this is not intended source.
   - Avoid using generated files as source of truth.

5. Add a simple health endpoint.
   - Example: `GET /api/health`.
   - Include database and MQTT connection state later.

## Priority 1: Complete Phase 1 Properly

### Backend Priorities

1. Implement refresh token support or remove client references.
   - Add `POST /api/auth/refresh`, refresh token DTO, persistence/rotation strategy.
   - Or simplify frontend to access-token-only deliberately.

2. Validate Socket.IO authentication.
   - Backend currently accepts `auth.token` but does not validate it in the gateway.
   - Add token validation during `handleConnection`.
   - Reject invalid sockets and attach user identity server-side.

3. Wire MQTT ingestion into `IotService`.
   - Parse `sensors/+/data` topics.
   - Extract sensor ID and numeric value.
   - Call `IotService.processSensorData(sensorId, value)`.
   - Ensure invalid payloads are logged and ignored safely.

4. Persist threshold alerts.
   - When `IotService` detects a threshold violation, create an `Alert` record through `AlertsService`.
   - Broadcast the same event name the frontend handles, preferably `alert-created`.

5. Introduce TypeORM migrations.
   - Stop relying on `synchronize` for serious environments.
   - Create an initial migration for current entities.

### Frontend Priorities

1. Normalize realtime event names.
   - Listen for backend-emitted `threshold-alert` or change backend to emit `alert-created`.
   - Ensure dashboard, alerts, and monitoring slices update consistently.

2. Add missing UI loading/error patterns to current pages.
   - Stations already has relatively strong UI state.
   - Monitoring, alerts, and maintenance should gain consistent retry/empty/error handling.

3. Add role-aware UI behavior.
   - Hide destructive/admin actions from users who do not have the required role.
   - Keep server-side RBAC as source of truth.

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

## Recommended Immediate Sprint

1. Fix `/admin/builder` route and remove unused route import warning.
2. Add `.env.example` files matching real ports.
3. Implement socket JWT validation.
4. Wire MQTT messages to `IotService.processSensorData`.
5. Persist threshold alerts and emit `alert-created`.
6. Replace in-memory flow storage with TypeORM workflow persistence.
7. Add minimal tests around MQTT-to-sensor-to-alert flow.

