# AquaFlow — AI Continuation Prompt

**Last updated:** 2026-05-25
**Purpose:** Paste this entire document at the start of a new Claude Code session to resume AquaFlow development without re-scanning the codebase. All facts are derived from a full file-by-file code audit.

---

## What Is AquaFlow?

AquaFlow is an industrial water-station supervision platform built as a PFE (final-year engineering project). It monitors water distribution infrastructure in real time: stations, sensors, alerts, maintenance, and automation workflows. The codebase lives at `pfe-project/` and is split into a NestJS backend (`backend/`) and a React frontend (`frontend/`). The full stack runs with `docker-compose up --build` in one command.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | NestJS 10, Node 20, TypeScript |
| Database | PostgreSQL 15 via TypeORM (9 entities, 1 migration) |
| Cache | Redis 7 (sensor list cache + auth denylist; in-memory fallback) |
| MQTT | Eclipse Mosquitto 2 — topic `sensor/+/data` ingestion |
| Auth | JWT access (1 h) + refresh (7 d) + Redis denylist on logout |
| Real-time | Socket.IO — 5 server→client events |
| API docs | Swagger at `/api/docs` |
| Frontend | React 18 + Redux Toolkit + React Router 6 |
| UI | Argon Dashboard React (Reactstrap / Bootstrap 4) |
| Charts | Chart.js 2 via react-chartjs-2 |
| Workflow canvas | JointJS |
| Container | Docker + Docker Compose (5-service stack) |

---

## Repository Layout (Key Paths)

```
pfe-project/
├── backend/src/
│   ├── alerts/           CRUD + ack/resolve + spec
│   ├── analytics/        Overview, sensor stats, station history
│   ├── auth/             JWT, refresh, Redis denylist + spec
│   ├── common/           JwtGuard, RolesGuard, decorators, workflow.types.ts
│   ├── database/
│   │   ├── entities/     9 entities (User, Station, Sensor, SensorData, Alert,
│   │   │                  Maintenance, Notification, Workflow, WorkflowExecution)
│   │   ├── migrations/   1778543154417-InitialSchema.ts
│   │   └── seeds/        seed.ts — 5 stations, 15 sensors, 5 alerts, 4 maint orders
│   ├── execution/
│   │   ├── engine/       node-executor.ts, workflow-runner.ts, execution-context.ts
│   │   └── handlers/     10 real handlers + 2 STUBS (api, notification)
│   ├── flows/            FlowsService (DB-persisted), FlowExecutorService (no DB write),
│   │                     FlowValidatorService, FlowsController
│   ├── iot/              IotService (MQTT ingestion), MqttClient
│   ├── maintenance/      Full CRUD
│   ├── notifications/    In-app + email (nodemailer) + WS broadcast + spec
│   ├── realtime/         RealtimeGateway, RealtimeService
│   ├── sensors/          Full CRUD + Redis cache + inject reading endpoint
│   ├── stations/         Full CRUD + station-status WS emit
│   ├── app.module.ts     NO AppController (no /api/health route)
│   └── main.ts           CORS, ValidationPipe, Swagger setup, port 3001
├── frontend/src/
│   ├── components/       Sidebar, AdminNavbar (notification bell), builder nodes
│   ├── data/blocks.js    14 block types (6 generic + 8 industrial/integration)
│   ├── engine/           autosaveManager, graphSerializer/Deserializer, workflowExecutorClient
│   ├── hooks/            useSocket (5 WS events), useWorkflowEditor, useAutosave, useLogout
│   ├── layouts/Admin.js  routes + /monitoring/:sensorId + /stations/:stationId
│   ├── modules/
│   │   ├── alerts/       AlertsPage (table, severity/status filters, ack, resolve) ✅
│   │   ├── analytics/    AnalyticsPage (KPI cards, doughnut charts, sensor stats) ✅
│   │   ├── auth/         Login, Register, ProtectedRoute ✅
│   │   ├── dashboard/    DashboardPage (KPIs, feeds, realtime; no trend charts) 🔶
│   │   ├── maintenance/  MaintenancePage (CRUD modal; no filter bar, no assignedTo) 🔶
│   │   ├── monitoring/   MonitoringPage (table; no filter bar) 🔶
│   │   │                 SensorDetailsPage (line chart, limit picker) ✅
│   │   └── stations/     StationsPage (filters, CRUD, delete) ✅
│   │                     StationDetailsPage (local state; no analytics chart) 🔶
│   ├── pages/BuilderPage.jsx         Actual workflow builder implementation
│   ├── views/builder/BuilderPage.jsx Re-export of pages/BuilderPage.jsx
│   ├── views/examples/Profile.js     Argon stub — no real data
│   ├── views/test.js                 Diagnostics page (embeds BuilderPage)
│   ├── routes.js         8 sidebar routes + hidden /test + auth routes
│   ├── services/         apiClient.js, authSession.js + 8 domain services
│   └── store/
│       ├── store.js      9 slices: auth, dashboard, realtime, stations, sensors,
│       │                  alerts, maintenance, ui, notifications
│       └── slices/       9 slice files; each has selectors exported
├── docker-compose.yml    5 services with healthchecks
└── mosquitto/config/
```

---

## Complete API Endpoint Table

### Auth
| Method | Path | Auth | Body |
|--------|------|------|------|
| POST | `/api/auth/register` | none | `{ email, password, firstname, lastname }` |
| POST | `/api/auth/login` | none | `{ email, password }` → `{ access_token, refresh_token, user }` |
| GET | `/api/auth/me` | JwtGuard | → current user |
| POST | `/api/auth/logout` | JwtGuard | `{ refresh_token }` |
| POST | `/api/auth/refresh` | none | `{ refresh_token }` → new token pair |

### Stations (JwtGuard + RolesGuard)
| Method | Path | Min Role | Query/Notes |
|--------|------|----------|------------|
| GET | `/api/stations` | any | `page, limit, search, status, type` |
| GET | `/api/stations/:id` | any | with sensors + alerts + maintenances |
| POST | `/api/stations` | operator | |
| PATCH | `/api/stations/:id` | operator | emits `station-status` WS if status in body |
| DELETE | `/api/stations/:id` | admin | 204 |

### Sensors (JwtGuard + RolesGuard)
| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| GET | `/api/sensors` | any | `page, limit, stationId, type, status, search`; Redis-cached |
| GET | `/api/sensors/:id` | any | with station + recent alerts |
| GET | `/api/sensors/:id/data` | any | `limit` (default 100); newest first |
| POST | `/api/sensors` | operator | |
| PATCH | `/api/sensors/:id` | operator | invalidates list cache |
| DELETE | `/api/sensors/:id` | admin | 204 |
| POST | `/api/sensors/:id/reading` | operator | Manual inject: `{ value }` → updates lastReading, saves SensorData |

### Alerts (JwtGuard + RolesGuard)
| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| GET | `/api/alerts` | any | `page, limit, severity, status` |
| GET | `/api/alerts/:id` | any | |
| POST | `/api/alerts` | operator | broadcasts `alert-created` WS |
| PATCH | `/api/alerts/:id/acknowledge` | technician | |
| PATCH | `/api/alerts/:id/resolve` | technician | |

### Maintenance (JwtGuard + RolesGuard)
| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| GET | `/api/maintenance` | any | `page, limit, status, priority` |
| GET | `/api/maintenance/:id` | any | |
| POST | `/api/maintenance` | technician | |
| PATCH | `/api/maintenance/:id` | technician | |
| DELETE | `/api/maintenance/:id` | admin | 204 |

### Flows (JwtGuard only)
| Method | Path | Notes |
|--------|------|-------|
| GET | `/api/flows` | list all |
| GET | `/api/flows/:id` | |
| POST | `/api/flows` | `{ name, graph }` |
| PUT | `/api/flows/:id` | replace graph |
| DELETE | `/api/flows/:id` | |
| POST | `/api/flows/execute` | `{ graph, input }` — ad-hoc run |

### Analytics (JwtGuard only)
| Method | Path | Query Params |
|--------|------|-------------|
| GET | `/api/analytics/overview` | — |
| GET | `/api/analytics/sensors/:id/stats` | `from, to` (ISO 8601; default 24h ago to now) |
| GET | `/api/analytics/stations/:id/history` | `from, to, granularity=hour\|day` |

### Notifications (JwtGuard only)
| Method | Path | Notes |
|--------|------|-------|
| GET | `/api/notifications` | `page, limit` |
| GET | `/api/notifications/unread-count` | → `{ count }` |
| PATCH | `/api/notifications/read-all` | → `{ updated }` |
| PATCH | `/api/notifications/:id/read` | |

**No `/api/health` endpoint exists** — `app.module.ts` has no AppController.

---

## WebSocket Events Table

### Server → Client (emitted by backend, received by `useSocket.js`)

| Event | Payload | Redux actions dispatched |
|-------|---------|------------------------|
| `sensor-update` | `{ sensorId, value, unit, timestamp, status }` | `sensorUpdateReceived` (realtime), `sensorRealtimeUpdated` (sensors), `applySensorUpdate` (dashboard) |
| `alert-created` | alert object | `alertReceived` (realtime), `alertRealtimeReceived` (alerts), `addDashboardAlert` (dashboard) |
| `station-status` | `{ stationId, status, name, timestamp }` | `stationStatusReceived` (realtime), `updateStationStatus` (dashboard), `stationRealtimeUpdated` (stations) |
| `notification-created` | notification object | `notificationReceived` (notifications) |
| `notifications-read-all` | — | `allNotificationsCleared` (notifications) |

### Client → Server

| Event | Payload | Notes |
|-------|---------|-------|
| `subscribe` | `{ channel: 'dashboard'\|'alerts'\|'stations'\|'sensors' }` | sent automatically on connect |
| `unsubscribe` | `{ channel }` | |
| `ping` | — | server responds with `{ pong: Date.now() }` |

---

## Redux Store Shape

```javascript
store = {
  auth:          { user, accessToken, refreshToken, loading, error },
  dashboard:     { stations, sensors, alerts, realtimeStats },
  realtime:      { connected, sensorUpdates, alertsReceived, stationStatuses },
  stations:      { items, selectedStation, meta, filters, isLoading, isSaving, error },
  sensors:       { items, meta, isLoading, isSaving, error },
  alerts:        { items, meta, isLoading, error },
  maintenance:   { items, meta, isLoading, isSaving, error },
  ui:            { sidebarMini, theme, notifications },
  notifications: { items, unreadCount, meta, isLoading, error },
}
```

---

## Demo Credentials (from seed.ts — verified)

| Email | Password | Role |
|-------|----------|------|
| `admin@aquaflow.local` | `Admin123!` | admin |
| `operator@aquaflow.local` | `Operator123!` | operator |
| `technician@aquaflow.local` | `Tech123!` | technician |
| `analyst@aquaflow.local` | `Analyst123!` | analyst |

> **Note:** The old docs said `admin@aquaflow.io` — this is wrong. Domain is `.local`.

---

## Current Bugs (with exact file + fix)

### 🔴 Bug 1: No `/api/health` endpoint
- **File:** `backend/src/app.module.ts`
- **Fix:** Create `backend/src/app.controller.ts` with `@Get('health')` returning `{ status: 'ok' }`. Add `controllers: [AppController]` to the `@Module` decorator.

### 🔴 Bug 2: `api` workflow block returns mock data
- **File:** `backend/src/execution/engine/node-executor.ts` line 60
- **Fix:** Change `case 'api': return { request: node.data, input, mocked: true };` to `case 'api': return this.httpRequestHandler.execute(node, input);`

### 🔴 Bug 3: `notification` workflow block returns mock data
- **File:** `backend/src/execution/engine/node-executor.ts` line 61
- **Fix:** Create `NotificationHandler` that calls `NotificationsService.createBroadcast()`. Wire it in `NodeExecutor` constructor.

### 🔴 Bug 4: "View all notifications" is a dead link
- **File:** `frontend/src/components/Navbars/AdminNavbar.js` line 228
- **Fix:** Create `frontend/src/modules/notifications/pages/NotificationsPage.jsx` and register `/admin/notifications` in `routes.js`.

### 🔶 Bug 5: MonitoringPage has no sensor filter bar
- **File:** `frontend/src/modules/monitoring/pages/MonitoringPage.jsx` line 78
- **Fix:** Add `stationFilter`, `typeFilter` state; pass them to `dispatch(fetchSensors({ stationId, type }))`. Add filter bar UI in `<CardHeader>`.

### 🔶 Bug 6: StationDetailsPage has no analytics chart
- **File:** `frontend/src/modules/stations/pages/StationDetailsPage.jsx`
- **Fix:** Add `useEffect` that calls `analyticsService.getStationHistory(stationId, { granularity: 'hour' })`. Render a `<Line>` chart from the response.

### 🔶 Bug 7: MaintenancePage missing filter bar and assignedTo field
- **File:** `frontend/src/modules/maintenance/pages/MaintenancePage.jsx`
- **Fix:** Add `assignedTo` to `initialForm`. Add status/priority filter selects. Pass filters to `dispatch(fetchMaintenance(params))`.

### 🔶 Bug 8: Workflow execution never persisted
- **File:** `backend/src/flows/flow-executor.service.ts`
- **Fix:** Inject `WorkflowExecution` repository. Before `runner.run()` create execution record; after, update with status/output/duration.

---

## Features Not Yet Built (with effort estimates)

| Feature | Estimated Effort |
|---------|-----------------|
| Add `/api/health` (Fix 1) | 15 min |
| Fix `api` block (Fix 2) | 5 min |
| Fix `notification` block (Fix 3) | 1.5 h |
| Add notifications page (Fix 4) | 30–45 min |
| Sensor filter bar (MonitoringPage) | 1.5 h |
| Station history chart | 2.5 h |
| Maintenance filters + assignedTo | 2 h |
| Workflow execution logging | 3 h |
| Alert detail modal | 2 h |
| UsersModule + user management page | 2 d |
| Dashboard trend charts | 1.5 d |
| GIS station map (react-leaflet) | 1 d |
| Workflow scheduling (cron/MQTT triggers) | 3 d |
| CSV export (alerts + sensor data) | 1.5 d |
| Live streaming chart (rolling buffer) | 1 d |
| Password reset flow | 1 d |
| Backend CI pipeline | 2 h |
| Expanded test coverage | 3 d |

---

## Architecture Principles — Never Break

1. **All API routes protected by JwtGuard** — no endpoint should be left public except `auth/register`, `auth/login`, `auth/refresh`
2. **RBAC via RolesGuard + `@Roles()` decorator** — use the `UserRole` enum; never hardcode role strings
3. **Redis cache always has in-memory fallback** — `app.module.ts` pattern ensures the app starts even if Redis is down
4. **Sensor list cache must be invalidated on create/update/delete** — see `SensorsService.clearListCache()`
5. **WebSocket JWT validation on connect** — `RealtimeGateway.handleConnection()` disconnects sockets with invalid tokens
6. **`station-status` WS event emitted only when `dto.status` is in PATCH body** — do not emit on every update
7. **All DTOs have `@ApiProperty`** — required for Swagger docs
8. **`flow-executor.service.ts` must call `validator.validate(graph)` before running** — prevents malformed graphs from crashing the runner
9. **Frontend apiClient.js handles 401 → auto refresh → retry** — never break this interceptor chain
10. **`deleteStation` thunk expects the service to return the station `id`** — `stationService.deleteStation(id)` returns `id`, not the object

---

## How to Start the Project

```bash
# Full stack (requires Docker)
cd pfe-project
docker-compose up --build

# Backend only (dev hot reload)
cd backend
cp .env.example .env   # set DATABASE_HOST=localhost, etc.
npm install
npm run migration:run
npm run seed            # optional — loads demo data
npm run start:dev       # http://localhost:3001

# Frontend only (dev)
cd frontend
npm install
npm start               # http://localhost:3000

# Swagger UI
open http://localhost:3001/api/docs

# Run backend tests
cd backend
npm run test            # unit tests
npm run test:e2e        # E2E tests
```

---

## Recommended First Session Task Order

Start with the quickest wins that unblock the most:

1. **Fix 1** (15 min) — Add `/api/health` endpoint → fixes Docker healthcheck
2. **Fix 2** (5 min) — Wire `api` block to `HttpRequestHandler` → removes silent mock
3. **Fix 4** (30–45 min) — Create notifications page + route → fixes dead navbar link
4. **P2-A** (1.5 h) — Add sensor filter bar to `MonitoringPage`
5. **P2-C** (2 h) — Add maintenance filters + `assignedTo` to `MaintenancePage`
6. **P2-E** (2 h) — Add alert detail modal to `AlertsPage`
7. **Fix 3** (1.5 h) — Wire `notification` block to `NotificationsService`
8. **P2-D** (3 h) — Persist workflow execution to DB

After completing these 8 tasks, the platform will be ~80% complete with all broken items fixed and the core partial features done.
