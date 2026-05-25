# AquaFlow — Next Development Steps

**Audit date:** 2026-05-25
**Basis:** Real codebase audit — all previous critical fixes and P1–P8 tasks are now complete
**Priority strategy:** Fix remaining broken → complete partials → add high-value missing → advanced features

---

## Critical Fixes (Do First — Bugs or Security)

### 🔴 Fix 1: Add Health Check Endpoint

**Why:** The backend `Dockerfile` healthcheck calls `/api/auth/me` which returns `401` when unauthenticated — so Docker Compose reports the backend as unhealthy even when it is running. This also means `depends_on: condition: service_healthy` in the frontend service will never pass.

**File:** Create `pfe-backend/src/app.controller.ts`

```typescript
import { Controller, Get } from '@nestjs/common';
import { ApiTags, ApiOperation } from '@nestjs/swagger';

@ApiTags('health')
@Controller()
export class AppController {
  @Get('health')
  @ApiOperation({ summary: 'Health check' })
  health() {
    return { status: 'ok', timestamp: new Date().toISOString() };
  }
}
```

Register in `app.module.ts` → `controllers: [AppController]`. Update the Dockerfile healthcheck:
```
CMD wget -qO- http://localhost:3001/api/health | grep -q ok && exit 0 || exit 1
```

**Effort:** 20 minutes

---

### 🔴 Fix 2: Fix `api` and `notification` Workflow Block Stubs

**Why:** In `NodeExecutor`, the `api` and `notification` cases return stub data instead of doing real work. Users who drag these blocks into a workflow get silently incorrect results.

**For `api` block** — it has no dedicated handler file. The `http-request` handler already exists and does real HTTP calls. Wire the `api` case to the same `HttpRequestHandler`:
```typescript
// In node-executor.ts
case 'api':         return this.httpRequestHandler.execute(node, input);
// Remove: case 'api': return { request: node.data, input, mocked: true };
```

**For `notification` block** — inject `NotificationsService` into `NodeExecutor` and create a minimal inline handler or a `NotificationHandler` that calls `notificationsService.notifyAlertCreated()` or a new `broadcastMessage()` method.

**Files:**
- `pfe-backend/src/execution/engine/node-executor.ts`
- `pfe-backend/src/execution/handlers/notification.handler.ts` (new)

**Effort:** 1.5 hours

---

## High Priority — Complete Partials

### P1: Persist Workflow Execution History to DB

**Why:** `WorkflowExecution` entity is fully defined and registered in TypeORM but `FlowExecutorService.execute()` never saves execution records. There is no execution history, duration tracking, or error log.

**File:** `pfe-backend/src/flows/flow-executor.service.ts`

**Changes:**
1. Inject `@InjectRepository(WorkflowExecution)` and optional `triggeredBy: User`
2. Before `runner.run()`: create and save a `WorkflowExecution` record with `status: 'running'`
3. After `runner.run()`: update the record with `status: 'completed'`, `output`, `duration`, `nodeStates`
4. In catch block: update with `status: 'failed'`, `errorMessage`, `stackTrace`
5. Also increment `workflow.executionCount` and set `workflow.lastExecutedAt`

Add `GET /flows/:id/executions` to `FlowsController` to list past runs.

**Effort:** 3 hours

---

### P2: Add Sensor Filter Bar to MonitoringPage

**Why:** Backend `GET /sensors` supports `stationId` and `type` query params, but the `MonitoringPage` loads all sensors with no filter controls. On any real deployment with many sensors, this is unusable.

**File:** `pfe-frontend/src/modules/monitoring/pages/MonitoringPage.jsx`

**Changes:**
1. Add local state: `stationFilter = ''`, `typeFilter = ''`
2. Add filter bar row (matching the pattern used in `StationsPage`):
   - Station dropdown populated from `selectStations`
   - Sensor type dropdown: pressure/flow/temperature/quality/level/ph/turbidity/chlorine
   - Clear button when filters are active
3. Pass filters to `dispatch(fetchSensors({ stationId: stationFilter, type: typeFilter }))`
4. Re-fetch on filter change via `useEffect`

**Effort:** 1.5 hours

---

### P3: Wire Station History Chart into StationDetailsPage

**Why:** `GET /api/analytics/stations/:id/history` returns a full multi-sensor time-series dataset but `StationDetailsPage` doesn't call it — it only shows a static sensors list and alerts table. The analytics API for stations is wasted.

**File:** `pfe-frontend/src/modules/stations/pages/StationDetailsPage.jsx`

**Changes:**
1. Import `analyticsService` and `Line` from `react-chartjs-2`
2. On mount, call `analyticsService.getStationHistory(stationId, { granularity: 'hour' })`
3. Build a multi-line chart: one line per sensor, using `timeSeries.buckets` → avg values
4. Add a range preset bar (24h / 7d / 30d) to re-trigger the fetch
5. Show a "No data" empty state when `sensors` array is empty

**Effort:** 2.5 hours

---

### P4: Add Maintenance Filter Bar and Technician Assignment

**Why:** The maintenance list can grow long with no way to filter. The `assignedTo` field is in the DB entity and API but there is no UI control to set it.

**File:** `pfe-frontend/src/modules/maintenance/pages/MaintenancePage.jsx`

**Changes:**
1. Add filter bar: status dropdown + priority dropdown + clear button (same pattern as stations)
2. Dispatch `fetchMaintenance({ status: statusFilter, priority: priorityFilter })` on change
3. In the create/edit modal, add an "Assigned To" text input (user UUID or name — or a future user dropdown once UsersModule exists)
4. Add the `assignedTo` field to `createMaintenance` and `updateMaintenance` thunks' payload

**Effort:** 2 hours

---

### P5: Fix Backend Healthcheck in Dockerfile

**Already detailed in Fix 1 above.** This is also needed as a standalone task since it affects local `docker-compose up` reliability.

---

### P6: Add Alert Detail Modal / Drawer

**Why:** `AlertsPage` shows a table but there is no way to view the full alert `description`, `data` JSON, or `acknowledgedAt`/`resolvedAt` timestamps. The API `GET /alerts/:id` exists.

**File:** `pfe-frontend/src/modules/alerts/pages/AlertsPage.jsx`

**Changes:**
1. Add a "View" button in each row's action column
2. On click, fetch `alertService.getAlert(id)` and open a Modal
3. Modal shows: type, severity, status, full message, description, data JSON (formatted), station name, sensor name, createdAt, acknowledgedAt, resolvedAt, acknowledgedBy

**Effort:** 2 hours

---

## Medium Priority — New Features

### P7: UsersModule (Backend) + User Management Page (Frontend)

**Why:** Admins cannot manage users from the app. There is no way to deactivate a user, change their role, or list all users.

**Backend — new module `pfe-backend/src/users/`:**
- `GET /users` — paginated list, admin only
- `GET /users/:id` — user detail
- `PATCH /users/:id` — update role, isActive
- `DELETE /users/:id` — soft delete (set isActive: false)
- `PATCH /auth/profile` — let the current user update their own name/password

**Frontend — new page `pfe-frontend/src/modules/users/pages/UsersPage.jsx`:**
- Table: email, name, role, status, joined date
- Role change dropdown (admin only)
- Deactivate/reactivate button
- Add route `/admin/users` in `routes.js` (only visible to admin role)

**Effort:** 2 days

---

### P8: Dashboard Time-Series Chart Section

**Why:** The dashboard is the first screen users see. It currently has KPI cards and tables but no charts. A 24h pressure / flow trend chart would make the dashboard much more informative.

**Files:**
- `pfe-frontend/src/modules/dashboard/pages/DashboardPage.jsx`
- `pfe-frontend/src/modules/dashboard/components/TrendCharts.jsx` (new)

**Changes:**
1. Create `TrendCharts` component: on mount, call `analyticsService.getOverview()` to get sensor IDs, then call `analyticsService.getSensorStats(id, { from: -24h })` for the first available pressure sensor and flow sensor
2. Render two line charts side by side: "24h Pressure Trend" and "24h Flow Rate"
3. Add a refresh button with last-updated timestamp
4. Insert `<TrendCharts />` in `DashboardPage` below the KPI section

**Effort:** 1.5 days

---

### P9: Workflow Execution History UI

**After P1 is done.** Add a read-only execution history panel to the Workflow Builder page.

**File:** `pfe-frontend/src/views/builder/BuilderPage.jsx`

**Changes:**
1. After saving a workflow, show a collapsible "Execution History" sidebar or bottom panel
2. Fetch `GET /flows/:id/executions` and display: run time, duration, status (completed/failed), input summary
3. Click a row to expand node-by-node output (from `nodeStates`)

**Effort:** 1.5 days

---

### P10: GIS Map for Stations

**Why:** All stations have `latitude` and `longitude` in the DB. A map view is far more useful than a table for infrastructure spread across a city or region.

**Dependencies:** Add `leaflet` + `react-leaflet` to frontend.

**Implementation:**
1. `npm install leaflet react-leaflet`
2. Create `pfe-frontend/src/modules/stations/components/StationsMap.jsx`
3. Render a Leaflet map with one marker per station, colored by status (normal=green, warning=yellow, critical=red, offline=gray)
4. Clicking a marker navigates to `StationDetailsPage`
5. Add a "Map / Table" toggle button to `StationsPage`

**Effort:** 1 day

---

### P11: Workflow Scheduling / MQTT-Triggered Execution

**Why:** `Workflow` entity already has `triggerType` and `triggerConfig` JSON fields. The engine is ready to execute any graph. Adding cron and MQTT triggers unlocks the real industrial automation use case.

**Backend changes:**
1. Install `@nestjs/schedule` and add `ScheduleModule.forRoot()` to `AppModule`
2. Create `WorkflowSchedulerService`: on startup, query all `Workflow` where `triggerType = 'scheduled'` and `isActive = true`, schedule cron jobs from `triggerConfig.cron`
3. In `IotService.handleMessage()`, after processing a reading, check if any active workflow has `triggerType = 'sensor_threshold'` with matching `sensorId` → execute it
4. Emit a `workflow-event` WebSocket message after each scheduled/triggered execution with status + nodeStates summary

**Frontend changes:**
- In the workflow properties panel, add a "Trigger" section: dropdown for manual/scheduled/sensor_threshold, and config inputs for cron expression or sensor selector

**Effort:** 3 days

---

### P12: CSV Export for Alerts and Sensor Data

**Why:** Operations teams always want to export data. The analytics backend has all the queries needed.

**Backend:**
1. `GET /alerts/export?format=csv&from=&to=` — stream CSV rows using `fast-csv` or string-building
2. `GET /sensors/:id/data/export?format=csv&from=&to=` — stream sensor readings

**Frontend:**
1. Add "Export CSV" button to `AlertsPage` toolbar
2. Add "Export CSV" button to `SensorDetailsPage`
3. Trigger a file download using `window.URL.createObjectURL(blob)`

**Effort:** 1.5 days

---

## Lower Priority — Advanced Features

### P13: Password Reset Flow

**Backend:** `POST /auth/forgot-password` (sends reset email), `POST /auth/reset-password` (validates token from email link, updates password). Store time-limited token in Redis.

**Frontend:** "Forgot password?" link on Login page → form → success screen.

**Effort:** 1 day

---

### P14: Real-time Live Chart (Streaming Gauge / Sparkline)

**Why:** The sensor table shows the latest value but there is no chart that scrolls in real time as WebSocket `sensor-update` events arrive.

**Implementation:** In `SensorDetailsPage` or `MonitoringPage`, maintain a rolling buffer (last 50 readings) in React state, updated on each `sensor-update` event via `useSocket`. Render as a sparkline chart using Chart.js.

**Effort:** 1 day

---

### P15: Backend CI/CD Pipeline

**Why:** Only a frontend GitHub Actions workflow exists (`frontend/.github/workflows/main.yml`). The backend has no automated pipeline.

**Create `pfe-backend/.github/workflows/ci.yml`:**
```yaml
name: Backend CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run test
      - run: npm run test:e2e
```

**Effort:** 2 hours

---

### P16: Expand Test Coverage

**Backend — missing test files:**
- `stations.service.spec.ts` — CRUD + realtime broadcast
- `sensors.service.spec.ts` — CRUD + cache invalidation
- `flows.service.spec.ts` — persistence + validation
- `iot.service.spec.ts` — MQTT message processing + threshold alerts

**Frontend — missing test files:**
- `alertsSlice.test.js` — acknowledge/resolve thunks
- `AlertsPage.test.jsx` — render, filter, button clicks
- `useSocket.test.js` — event dispatching
- `SensorDetailsPage.test.jsx` — chart render, data fetch

**Effort:** 3 days

---

## Implementation Order (Recommended)

```
Week 1:  Fix 1 (health endpoint), Fix 2 (stub blocks)
         P1 (execution history DB), P2 (sensor filter bar), P3 (station history chart)

Week 2:  P4 (maintenance filters + assignedTo), P6 (alert detail modal)
         P7 (UsersModule backend + frontend page)

Week 3:  P8 (dashboard trend charts), P9 (execution history UI)
         P10 (GIS map for stations)

Week 4:  P11 (workflow scheduling + MQTT triggers)
         P12 (CSV export)

Week 5:  P13 (password reset), P14 (live chart widget)
         P15 (backend CI), P16 (test coverage)
```

---

## Quick Wins (< 2 hours each)

| Task | Effort | Impact |
|------|--------|--------|
| Add `/api/health` endpoint (Fix 1) | 20 min | Fixes Docker healthcheck |
| Wire `api` block to `HttpRequestHandler` (Fix 2 partial) | 30 min | Removes silent mock |
| Add sensor type + station filter bar to MonitoringPage (P2) | 1.5 h | High UX improvement |
| Add alert detail modal (P6) | 2 h | Completes alert management |
| Add `assignedTo` field to maintenance modal (P4 partial) | 1 h | Completes the maintenance form |
| Add backend CI GitHub Actions file (P15) | 2 h | Automated test gate |
