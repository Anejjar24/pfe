# AquaFlow — AI Continuation Prompt

**Last updated:** 2026-05-25
**Purpose:** Paste this entire document at the start of a new Claude session to resume AquaFlow development instantly.

---

## What Is AquaFlow?

AquaFlow is an industrial water-station supervision platform built as a PFE (final-year engineering project). It monitors water distribution infrastructure in real time: stations, sensors, alerts, maintenance, and automation workflows.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | NestJS 10 + TypeScript, Node 20 |
| Database | PostgreSQL 15 via TypeORM |
| Cache | Redis 7 (cache-manager-redis-store, in-memory fallback) |
| MQTT | Eclipse Mosquitto 2 via mqtt npm package |
| Auth | JWT access (1h) + refresh (7d) + Redis denylist |
| Real-time | Socket.IO |
| API docs | Swagger at `/api/docs` |
| Frontend | React 18, Redux Toolkit, React Router 6 |
| UI | Argon Dashboard React (Reactstrap / Bootstrap 4) |
| Charts | Chart.js 2 via react-chartjs-2 |
| Workflow canvas | JointJS |
| Container | Docker + Docker Compose |

---

## Repository Layout

```
pfe-project/
├── backend/src/
│   ├── alerts/             Full CRUD + acknowledge + resolve
│   ├── analytics/          Overview, sensor stats, station history
│   ├── auth/               JWT, refresh, Redis denylist
│   ├── common/             JwtGuard, RolesGuard, decorators
│   ├── database/
│   │   ├── entities/       User, Station, Sensor, SensorData, Alert,
│   │   │                   Maintenance, Notification, Workflow, WorkflowExecution
│   │   ├── migrations/     InitialSchema1778543154417 (complete)
│   │   └── seeds/          seed.ts with demo data
│   ├── execution/
│   │   ├── engine/         workflow-runner, node-executor, execution-context
│   │   └── handlers/       action, alert-trigger, decision, http-request,
│   │                       input, mqtt-publish, output, pump-control,
│   │                       sensor-read, station-control, threshold-check
│   ├── flows/              FlowsService (DB-persisted), FlowExecutorService, FlowValidatorService
│   ├── iot/                IotService (MQTT), MqttClient
│   ├── maintenance/        Full CRUD
│   ├── notifications/      In-app + email (nodemailer), WS broadcast
│   ├── realtime/           RealtimeGateway, RealtimeService
│   ├── sensors/            Full CRUD + Redis cache + data history
│   └── stations/           Full CRUD + station-status WS emit
│   app.module.ts           CacheModule(Redis), all modules
│   main.ts                 CORS, ValidationPipe, Swagger setup
├── frontend/src/
│   ├── components/         Sidebar, AdminNavbar (notification bell), builder nodes
│   ├── data/blocks.js      14 block types: generic + Industrial category
│   ├── engine/             autosaveManager, graphSerializer/Deserializer, workflowExecutorClient
│   ├── hooks/              useSocket (5 WS events), useWorkflowEditor, useAutosave, useLogout
│   ├── modules/
│   │   ├── alerts/         AlertsPage (table, filters, ack, resolve)
│   │   ├── analytics/      AnalyticsPage (KPI cards, doughnut charts, sensor line chart)
│   │   ├── auth/           Login, Register, ProtectedRoute
│   │   ├── dashboard/      DashboardPage, KPISection, AlertsFeed, StationOverview, RealtimeStats
│   │   ├── maintenance/    MaintenancePage (full CRUD modal)
│   │   ├── monitoring/     MonitoringPage (edit/delete sensors), SensorDetailsPage (line chart)
│   │   └── stations/       StationsPage (filters, delete), StationDetailsPage
│   ├── services/           apiClient, authService, stationService, sensorService,
│   │                       alertService, maintenanceService, analyticsService,
│   │                       notificationService, workflowApi
│   ├── store/slices/       auth, dashboard, realtime, stations, sensors, alerts,
│   │                       maintenance, ui, notifications
│   └── routes.js           7 pages: dashboard, builder, stations, monitoring,
│                           alerts, maintenance, analytics
├── docker-compose.yml      postgres + redis + mosquitto + backend + frontend
├── backend/Dockerfile      Multi-stage Node 20 build
└── frontend/Dockerfile     Multi-stage nginx build
```

---

## Current Implementation Status

**Overall: ~67% complete (91/135 tracked features)**

### ✅ Fully Working
- Auth: register, login, JWT, refresh, logout with Redis denylist, RBAC
- Stations: full CRUD + status/type filters + delete + details page + realtime status WS
- Sensors: full CRUD + edit/delete + SensorDetailsPage with Chart.js line chart + realtime updates
- Alerts: AlertsPage (full table, severity/status filters, acknowledge, resolve, realtime via WS)
- Maintenance: full CRUD modal (create, edit, delete, station dropdown)
- Analytics: backend + frontend — overview KPIs, doughnut charts, sensor stats, sensor time-series
- Notifications: in-app bell (AdminNavbar), email to admins on critical alerts, WS push, mark-read
- Workflow Builder: DB-persisted workflows, JWT-protected, autosave, 14 block types, all 10 execution handlers (sensor-read, threshold-check, alert-trigger, mqtt-publish, pump-control, station-control, http-request all functional)
- Realtime: 5 WS events handled (sensor-update, alert-created, station-status, notification-created, notifications-read-all)
- Infrastructure: both Dockerfiles, full docker-compose, TypeORM migration, Redis cache, Swagger

### 🔶 Partially Working (Needs Completion)
- Sensor filter bar: backend supports stationId/type filters but MonitoringPage has no filter controls
- Station analytics in StationDetailsPage: `GET /analytics/stations/:id/history` exists but UI doesn't use it
- Maintenance: missing assignedTo field in form; no filter bar
- Alert detail view: no click-through modal to see full alert data
- Dashboard: no trend charts (data available via analytics API but not rendered on dashboard)
- Workflow `api` block: returns stub data instead of real HTTP
- Workflow `notification` block: returns stub data
- Workflow execution history: entity defined but never written

### ❌ Not Started
- Health check endpoint `/api/health` (Dockerfile uses /api/auth/me fallback which returns 401)
- User management: no UsersModule, no admin user list/edit page
- User profile update (PATCH /auth/profile)
- Password reset flow
- GIS map for stations (lat/lon in DB, no map component)
- Workflow execution scheduling (triggerType field exists but no cron runner)
- MQTT-triggered workflows
- CSV/PDF export
- Live streaming sensor gauge chart

---

## API Endpoints Reference

### Auth
- `POST /api/auth/register` — `{ email, password, firstname, lastname }`
- `POST /api/auth/login` — `{ email, password }` → `{ access_token, refresh_token, user }`
- `GET  /api/auth/me` — current user (JwtGuard)
- `POST /api/auth/refresh` — `{ refresh_token }`
- `POST /api/auth/logout` — `{ refresh_token }`

### Stations
- `GET  /api/stations?page=&limit=&search=&status=&type=`
- `POST /api/stations`
- `GET  /api/stations/:id`
- `PATCH /api/stations/:id`
- `DELETE /api/stations/:id`

### Sensors
- `GET  /api/sensors?page=&limit=&stationId=&type=`
- `POST /api/sensors`
- `GET  /api/sensors/:id`
- `PATCH /api/sensors/:id`
- `DELETE /api/sensors/:id`
- `GET  /api/sensors/:id/data?limit=100`

### Alerts
- `GET  /api/alerts?page=&limit=&severity=&status=`
- `POST /api/alerts`
- `GET  /api/alerts/:id`
- `PATCH /api/alerts/:id/acknowledge`
- `PATCH /api/alerts/:id/resolve`

### Maintenance
- `GET  /api/maintenance?page=&limit=&status=&priority=`
- `POST /api/maintenance`
- `GET  /api/maintenance/:id`
- `PATCH /api/maintenance/:id`
- `DELETE /api/maintenance/:id`

### Flows (Workflows)
- `GET  /api/flows` — list all (JwtGuard)
- `POST /api/flows` — `{ name, graph }` (JwtGuard)
- `GET  /api/flows/:id`
- `PUT  /api/flows/:id` — replace graph
- `DELETE /api/flows/:id`
- `POST /api/flows/execute` — `{ graph, input }` — ad-hoc execution

### Analytics
- `GET  /api/analytics/overview`
- `GET  /api/analytics/sensors/:id/stats?from=&to=`
- `GET  /api/analytics/stations/:id/history?from=&to=&granularity=hour|day`

### Notifications
- `GET  /api/notifications?page=&limit=`
- `GET  /api/notifications/unread-count`
- `PATCH /api/notifications/:id/read`
- `PATCH /api/notifications/read-all`

---

## WebSocket Events

**Server → Client:**
| Event | Payload | Redux dispatch |
|-------|---------|---------------|
| `sensor-update` | `{ sensorId, value, unit, timestamp, status }` | sensorUpdateReceived, sensorRealtimeUpdated, applySensorUpdate |
| `alert-created` | alert object | alertReceived, alertRealtimeReceived, addDashboardAlert |
| `station-status` | `{ stationId, status, name, timestamp }` | stationStatusReceived, updateStationStatus, stationRealtimeUpdated |
| `notification-created` | notification object | notificationReceived |
| `notifications-read-all` | — | allNotificationsCleared |

**Client → Server:**
| Event | Payload |
|-------|---------|
| `subscribe` | `{ channel: 'dashboard' \| 'alerts' \| 'stations' \| 'sensors' }` |
| `unsubscribe` | `{ channel }` |

---

## Redux Store Shape

```javascript
store = {
  auth: { user, accessToken, refreshToken, loading, error },
  dashboard: { stations, sensors, alerts, realtimeStats },
  realtime: { connected, sensorUpdates, alertsReceived, stationStatuses },
  stations: { items, meta, filters, loading, saving, error },
  sensors: { items, meta, loading, saving, error },
  alerts: { items, meta, loading, saving, error },
  maintenance: { items, meta, loading, saving, error },
  ui: { sidebarMini, theme, notifications },
  notifications: { items, unreadCount, loading, meta }
}
```

---

## Known Issues to Be Aware Of

1. **Health endpoint missing** — `/api/health` returns 404. The Docker healthcheck fails. Add `AppController` with a `@Get('health')` route before deploying.

2. **`api` and `notification` workflow blocks are stubs** — `NodeExecutor` returns mock data for these two block types. They show in the builder UI but don't execute real logic.

3. **Workflow execution not logged** — `WorkflowExecution` entity exists but `FlowExecutorService` never writes to it. No execution history.

4. **No sensor filter bar in MonitoringPage** — sensors load without any station/type filter controls even though the backend supports them.

5. **StationDetailsPage does not show station history chart** — `GET /analytics/stations/:id/history` is implemented but the detail page doesn't call it.

---

## How to Run Locally

```bash
# Full stack (requires Docker)
cd pfe-project
docker-compose up --build

# Backend only (dev)
cd backend
cp .env.example .env   # or create .env with DATABASE_HOST=localhost etc.
npm install
npm run migration:run
npm run seed           # optional demo data
npm run start:dev      # http://localhost:3001

# Frontend only (dev)
cd frontend
npm install
npm start              # http://localhost:3000

# Default login after seeding
# Email: admin@aquaflow.io   Password: Admin123!
```

---

## Suggested Next Task (Start Here)

The highest-value next tasks in order are:

1. **Add `/api/health` endpoint** — 20 minutes, fixes Docker healthcheck
2. **Add sensor filter bar to MonitoringPage** — 1.5 hours, high UX value
3. **Wire station history chart into StationDetailsPage** — 2.5 hours, uses existing API
4. **Persist workflow execution records to DB** — 3 hours, completes the execution engine
5. **Add alert detail modal** — 2 hours, completes alert management

When you start, state which task you are working on and I will provide the relevant code context.
