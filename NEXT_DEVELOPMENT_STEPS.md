# AquaFlow Next Development Steps

# AquaFlow — Next Development Steps

**Audit date:** 2026-05-10
**Basis:** Real codebase audit (not roadmap assumptions)
**Priority strategy:** Fix broken → complete partials → add missing core → add advanced features

---

## Critical Fixes (Do First — Blocking or Security)

### 🔴 Fix 1: Add JWT Guard to FlowsController

**Why:** All `/api/flows` endpoints are currently public. Any anonymous user can list, create, overwrite, or execute workflows.

**File:** `pfe-backend/src/flows/flows.controller.ts`

**Change:** Add `@UseGuards(JwtGuard)` to the controller class (and optionally `RolesGuard` with `@Roles(UserRole.ADMIN, UserRole.OPERATOR)`).

```typescript
// Before
@Controller('flows')
export class FlowsController { ... }

// After
@UseGuards(JwtGuard)
@Controller('flows')
export class FlowsController { ... }
```

**Effort:** 15 minutes

---

### 🔴 Fix 2: Build the AlertsPage

**Why:** `pfe-frontend/src/modules/alerts/pages/AlertsPage.jsx` is empty (0 bytes). The backend has full CRUD + acknowledge/resolve APIs. Users cannot see or manage alerts.

**What to build:**

- Fetch and display alerts list using `alertsSlice` `fetchAlerts` thunk
- Show columns: severity badge, message, station, sensor, created time, status
- Acknowledge button → `PATCH /api/alerts/:id/acknowledge`
- Resolve button → `PATCH /api/alerts/:id/resolve`
- Filter bar by severity (critical/warning/info) and status (active/acknowledged/resolved)
- Realtime: use `useSocket(true)` to receive new alerts via `alert-created` event

**Effort:** 1 day

---

### 🔴 Fix 3: Emit `station-status` Event from Backend

**Why:** `useSocket.js` already listens for `station-status` events and dispatches `stationStatusReceived` to Redux, but the backend never emits this event. The StationsPage status updates only on full page reload.

**Where to fix:** `pfe-backend/src/stations/stations.service.ts` — inject `RealtimeService` and call `broadcastToAll('station-status', {...})` inside `update()` when status changes.

**Effort:** 1 hour

---

## High Priority — Complete Phase 2 (Core Modules)

### P1: Persist Workflows to PostgreSQL

**Why:** Workflows are lost on every backend restart. The `Workflow` TypeORM entity is already defined.

**Backend changes:**

1. Inject `@InjectRepository(Workflow)` into `FlowsService`
2. Replace `Map<string, FlowRecord>` with TypeORM repository calls
3. Inject `currentUser` from JWT payload into `create()` and `update()`
4. Save `graph` JSON to the `graph` column in `Workflow` entity
5. Inject `@InjectRepository(WorkflowExecution)` to log executions

**File:** `pfe-backend/src/flows/flows.service.ts`

**Effort:** 4 hours

---

### P2: Generate TypeORM Migration Files

**Why:** `database/migrations/` is empty. Without migrations, production deployment is dangerous (no schema tracking).

**Steps:**

```bash
# In pfe-backend/
npx typeorm migration:generate src/database/migrations/InitialSchema -d src/database/database.service.ts
npx typeorm migration:run -d src/database/database.service.ts
```

Ensure `DatabaseModule` datasource config is exported as a standalone `DataSource` for CLI use.

**Effort:** 2 hours (including testing)

---

### P3: Add Create/Edit UI to MaintenancePage

**What to build:**

- "New Work Order" button → modal with fields: title, type, priority, status, station (dropdown), description, scheduledDate, assignedTo
- Edit button per row → same modal pre-filled
- Delete button with confirmation (admin only)
- Dispatch `createMaintenance` and `updateMaintenance` thunks (need to be added to `maintenanceSlice.js`)

**Files:**

- `pfe-frontend/src/modules/maintenance/pages/MaintenancePage.jsx`
- `pfe-frontend/src/store/slices/maintenanceSlice.js`

**Effort:** 1 day

---

### P4: Add Edit/Delete to StationsPage and MonitoringPage

**StationsPage gaps:**

- Delete station button with `confirm()` dialog (admin only)
- Station status filter already has backend support — wire the dropdown to `setStationFilters` and dispatch `fetchStations`

**MonitoringPage gaps:**

- Edit sensor button → modal with current data pre-filled
- Delete sensor button (admin only)
- Link to sensor details (create sensor detail page or expand row)

**Effort:** 1 day

---

### P5: Create Sensor Historical Data Chart

**What to build:**

- `GET /api/sensors/:id/data` returns time-series data — wire this to a chart component
- Use an existing charting library (recharts or Chart.js — add to package.json if needed)
- Create `SensorDetailsPage.jsx` route: `/admin/monitoring/:sensorId`
- Show line chart of last N readings vs time
- Show current reading, status, thresholds as reference lines on chart

**Effort:** 1.5 days

---

### P6: Add `uiSlice` to Redux Store

**What to build:**

```javascript
// store/slices/uiSlice.js
const uiSlice = createSlice({
  name: 'ui',
  initialState: { sidebarOpen: true, theme: 'light', notifications: [] },
  reducers: {
    toggleSidebar, setTheme, pushNotification, dismissNotification
  }
});
```

Register in `store.js`. Use `sidebarOpen` to control `Sidebar.js` toggle state.

**Effort:** 2 hours

---

## Medium Priority — Industrial Workflow Extensions (Phase 3)

### P7: Add Industrial Blocks to `data/blocks.js`

Add the following block definitions to the existing `workflowBlocks` array:


| Block type        | Category    | Description                                 |
| ----------------- | ----------- | ------------------------------------------- |
| `sensor-read`     | Industrial  | Reads current value from a sensor by ID     |
| `threshold-check` | Industrial  | Compares sensor value to min/max thresholds |
| `pump-control`    | Industrial  | Sends start/stop command to a pump via MQTT |
| `alert-trigger`   | Industrial  | Creates an alert in the system              |
| `mqtt-publish`    | Industrial  | Publishes arbitrary payload to MQTT topic   |
| `station-control` | Industrial  | Updates station status                      |
| `delay`           | Logic       | Waits N seconds before continuing           |
| `http-request`    | Integration | Makes external HTTP call                    |

Each block needs: `type`, `title`, `icon`, `category`, `description`, `color`, `inputs`, `outputs`, `properties`.

**Effort:** 4 hours (data only, no handler code yet)

---

### P8: Add Backend Execution Handlers for Industrial Blocks

**New files needed:**


| Handler               | File                                            | Logic                                                                                        |
| --------------------- | ----------------------------------------------- | -------------------------------------------------------------------------------------------- |
| SensorReadHandler     | `execution/handlers/sensor-read.handler.ts`     | Query`SensorRepository` by `sensorId` property, return `{ value, unit, timestamp, status }`  |
| ThresholdCheckHandler | `execution/handlers/threshold-check.handler.ts` | Compare input value to`minThreshold`/`maxThreshold` from block properties; output true/false |
| AlertTriggerHandler   | `execution/handlers/alert-trigger.handler.ts`   | Call`AlertsService.create()` with block-configured severity, message, stationId              |
| MqttPublishHandler    | `execution/handlers/mqtt-publish.handler.ts`    | Call`MqttClient.publish(topic, payload)`                                                     |
| PumpControlHandler    | `execution/handlers/pump-control.handler.ts`    | Publish pump command to MQTT topic`devices/{deviceId}/commands`                              |

Register all handlers in `NodeExecutor.execute()` switch/case.

**Effort:** 2 days

---

## Lower Priority — Advanced Features (Phase 4)

### P9: Analytics Module (Backend)

Create `pfe-backend/src/analytics/analytics.module.ts` with:

- `GET /api/analytics/overview` — aggregate counts: totalStations, activeSensors, openAlerts, maintenancePending
- `GET /api/analytics/sensors/:id/stats?from=&to=` — avg, min, max, count from `SensorData` table
- `GET /api/analytics/stations/:id/history` — sensor readings grouped by hour/day

**Effort:** 2 days

---

### P10: Analytics Dashboard Page (Frontend)

- Create `modules/analytics/pages/AnalyticsPage.jsx`
- Add route `/admin/analytics` to `routes.js`
- Add charts: pressure over time, flow rate trends, alert frequency
- Use recharts (or add it) for line/bar charts

**Effort:** 2 days

---

### P11: Add Redis for Session/Cache

**Use cases:**

- Cache sensor latest readings (avoid repeated DB queries)
- Store JWT refresh token denylist (on logout)
- Rate limiting

**Steps:**

1. Install `@nestjs/cache-manager`, `cache-manager`, `cache-manager-redis-store`
2. Add `CacheModule` to `AppModule` with Redis config from env
3. Inject `CACHE_MANAGER` into `SensorsService` to cache `findAll()` results
4. Inject into `AuthService` to store invalidated refresh tokens

**Effort:** 1 day

---

### P12: Backend Dockerfile + Full Docker Compose

**`pfe-backend/Dockerfile`:**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build
EXPOSE 3001
CMD ["node", "dist/main"]
```

**`pfe-frontend/Dockerfile`:**

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
EXPOSE 80
```

Add `backend` and `frontend` services to `docker-compose.yml` with proper `depends_on`.

**Effort:** 4 hours

---

### P13: Add Swagger/OpenAPI Documentation

In `pfe-backend/src/main.ts`:

```typescript
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
const config = new DocumentBuilder()
  .setTitle('AquaFlow API')
  .setVersion('1.0')
  .addBearerAuth()
  .build();
const document = SwaggerModule.createDocument(app, config);
SwaggerModule.setup('api/docs', app, document);
```

Add `@ApiProperty()` decorators to all DTOs and `@ApiTags()` to controllers.

**Effort:** 1 day

---

### P14: Notifications Module

1. Create `pfe-backend/src/notifications/notifications.module.ts`
2. Use `@nestjs-modules/mailer` for email
3. Trigger from `AlertsService.create()` when severity is `critical`
4. Store delivery status in `Notification` entity
5. Frontend: bell icon in navbar showing unread count via WebSocket

**Effort:** 2 days

---

### P15: Testing Infrastructure

**Backend:**

```bash
npm install --save-dev @nestjs/testing jest supertest
```

- Unit tests for: `AuthService`, `StationsService`, `AlertsService`, `IotService`
- E2E tests for: auth flow, station CRUD

**Frontend:**

```bash
npm install --save-dev @testing-library/react @testing-library/user-event
```

- Tests for: `useSocket`, `authSlice`, `DashboardPage` render

**Effort:** 3 days (initial coverage)

---

## Implementation Order (Recommended)

```
Week 1:  Fix 1 (JWT on flows), Fix 2 (AlertsPage), Fix 3 (station-status event)
         P1 (persist workflows), P2 (migrations)
Week 2:  P3 (Maintenance CRUD UI), P4 (edit/delete in Stations + Monitoring)
         P5 (Sensor history chart), P6 (uiSlice)
Week 3:  P7 (industrial blocks data), P8 (execution handlers)
Week 4:  P9 (analytics backend), P10 (analytics frontend)
Week 5:  P11 (Redis), P12 (Dockerfiles), P13 (Swagger)
Week 6:  P14 (notifications), P15 (tests)
```

---

## Quick Wins (< 2 hours each)

1. Add `@UseGuards(JwtGuard)` to `FlowsController` — **15 min, high security impact**
2. Emit `station-status` from `StationsService.update()` — **1 hour**
3. Create `uiSlice.js` — **2 hours**
4. Add `health` endpoint to `AppController` — **30 min**
5. Add delete button with confirm() to StationsPage — **1 hour**
6. Fix env drift: add `REACT_APP_WORKFLOW_API_URL` fallback to `REACT_APP_API_URL` in `workflowApi.js` — **15 min**

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
