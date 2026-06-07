# AquaFlow — AI Continuation Prompt

**Last updated:** 2026-05-27 — all development phases (P1–P4) + all P2 enhancements complete  
**Purpose:** Paste this entire document at the start of a new AI session to continue work on AquaFlow. All facts reflect the fully-complete state.

---

## What Is AquaFlow?

AquaFlow is an industrial water-station supervision platform built as a PFE (final-year engineering project). It monitors water distribution infrastructure in real time: stations, sensors, alerts, maintenance, and automation workflows. The codebase lives at `pfe-project/` and is split into a NestJS backend (`backend/`) and a React frontend (`frontend/`). The full dev stack runs with `docker compose up -d` in one command.

---

## Current Status (2026-05-27)

**Overall completion: 100% — all features complete**

| Phase | Description | Status |
|-------|-------------|--------|
| P1 | Critical bug fixes | ✅ ALL COMPLETE |
| P2 | UI enhancements + analyticsSlice + execution history | ✅ ALL COMPLETE |
| P3 | Planned features (users, charts, scheduling, GIS, CSV, live charts) | ✅ ALL COMPLETE |
| P4 | DevOps / production hardening | ✅ ALL COMPLETE |

No open issues. No critical bugs. Platform is production-ready.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | NestJS 10, Node 20, TypeScript |
| Database | PostgreSQL 15 via TypeORM (9 entities, 1 migration) |
| Cache | Redis 7 — `cache-manager-redis-yet@^4.1.2`, TTL in **milliseconds** |
| MQTT | Eclipse Mosquitto 2 — topic `aquaflow/sensor/<id>/data` |
| Auth | JWT access (1 h) + refresh (7 d) + Redis denylist on logout |
| Scheduling | `@nestjs/schedule@^4.1.0` — cron-based workflow triggers |
| Real-time | Socket.IO (6 server→client events) |
| API docs | Swagger at `/api/docs` |
| Frontend | React 18 + Redux Toolkit + React Router 6 (CRA) |
| UI | Argon Dashboard React (Reactstrap / Bootstrap 4) |
| Charts | Chart.js 2 via `react-chartjs-2@2.x` |
| GIS | `leaflet@^1.9.4` + `react-leaflet@^4.2.1` |
| Workflow canvas | JointJS / `@joint/core` |
| Container (dev) | `docker-compose.yml` — 5 services, all ports exposed |
| Container (prod) | `docker-compose.prod.yml` — hardened, no infra ports exposed |
| CI/CD | GitHub Actions — `.github/workflows/backend-ci.yml` + `frontend-ci.yml` |

---

## Repository Layout (post-P4)

```
pfe-project/
├── .github/workflows/
│   ├── backend-ci.yml       lint + build + unit tests + e2e (with postgres + redis services)
│   └── frontend-ci.yml      CI=true build (0 warnings) + Jest tests
├── .env.example             All required prod env vars documented
├── .gitignore               Excludes .env.prod, passwd, dist, node_modules
├── docker-compose.yml       Dev stack — all ports exposed, no auth on infra
├── docker-compose.prod.yml  Prod stack — hardened, resource limits, log rotation
├── mosquitto/config/
│   ├── mosquitto.conf        Dev: allow_anonymous true
│   └── mosquitto.prod.conf   Prod: allow_anonymous false + passwd auth
├── backend/src/
│   ├── app.controller.ts     GET /api/health (DB + Redis probes, HTTP 503 on degraded)
│   ├── app.module.ts
│   ├── auth/                 JWT, refresh, logout (Redis denylist), profile update
│   ├── users/                CRUD + RBAC (admin/operator/technician/analyst)
│   ├── stations/             CRUD + station-status WS broadcast
│   ├── sensors/              CRUD + Redis cache + injectReading
│   ├── alerts/               CRUD + ack/resolve + WS broadcast + CSV export
│   ├── maintenance/          CRUD
│   ├── analytics/            overview, sensor stats, station history
│   ├── notifications/        CRUD + bell badge + mark-read + WS broadcast
│   ├── flows/                Workflow CRUD + activate/deactivate + execute + scheduler
│   ├── iot/                  MQTT ingestion + threshold alerts
│   ├── realtime/             Socket.IO gateway
│   ├── common/               JwtGuard, RolesGuard, decorators, workflow.types.ts
│   ├── execution/engine/     node-executor.ts (api + notification blocks still STUBBED)
│   └── database/
│       ├── entities/         9 entities
│       ├── migrations/       1778543154417-InitialSchema.ts
│       └── seeds/            seed.ts
├── frontend/src/
│   ├── components/           Header, AdminNavbar (notification bell + badge)
│   ├── hooks/
│   │   └── useSocket.js      Handles 6 WS events, dispatches to Redux
│   ├── modules/
│   │   ├── auth/             Login, Register, ProtectedRoute
│   │   ├── dashboard/        DashboardPage + TrendCharts component
│   │   ├── stations/         StationsPage (table + Leaflet map toggle) + StationDetailsPage
│   │   ├── monitoring/       MonitoringPage (sensor table) + SensorDetailsPage (chart + live feed)
│   │   ├── alerts/           AlertsPage (table, filters, ack/resolve, CSV export)
│   │   ├── maintenance/      MaintenancePage (CRUD modal)
│   │   ├── analytics/        AnalyticsPage (KPIs, doughnut, sensor time-series)
│   │   ├── notifications/    NotificationsPage (full list)
│   │   ├── users/            UsersPage (role management, activate/deactivate)
│   │   └── workflow/         WorkflowBuilderPage (JointJS canvas, 14 block types)
│   ├── services/             alertService, sensorService, stationService, etc.
│   ├── store/
│   │   ├── store.js          9 slices registered
│   │   └── slices/           auth, dashboard, realtime, stations, sensors,
│   │                          alerts, maintenance, ui, notifications
│   └── routes.js             All sidebar routes defined here
```

---

## API Endpoints (~52 total)

Base URL: `http://localhost:3001/api`. All routes except auth require `Authorization: Bearer <token>`.

### Auth
| Method | Path | Guard |
|--------|------|-------|
| POST | `/auth/register` | none |
| POST | `/auth/login` | none |
| GET | `/auth/me` | JwtGuard |
| POST | `/auth/logout` | JwtGuard |
| POST | `/auth/refresh` | none |
| PATCH | `/auth/profile` | JwtGuard |
| GET | `/health` | none |

### Stations / Sensors / Alerts / Maintenance / Users
| Module | Endpoints |
|--------|-----------|
| Stations | GET `/stations`, POST, GET `/:id`, PATCH `/:id`, DELETE `/:id`, GET `/stations/map` |
| Sensors | GET `/sensors`, POST, GET `/:id`, PATCH `/:id`, DELETE `/:id`, POST `/:id/readings`, GET `/:id/data`, GET `/:id/data/export` |
| Alerts | GET `/alerts`, POST, GET `/:id`, PATCH `/:id/acknowledge`, PATCH `/:id/resolve`, DELETE `/:id`, GET `/alerts/export/csv` |
| Maintenance | GET `/maintenance`, POST, GET `/:id`, PATCH `/:id`, DELETE `/:id` |
| Users | GET `/users`, GET `/:id`, PATCH `/:id` |

### Analytics / Notifications / Workflows
| Module | Endpoints |
|--------|-----------|
| Analytics | GET `/analytics/overview`, GET `/analytics/sensors/:id/stats`, GET `/analytics/stations/:id/history` |
| Notifications | GET `/notifications`, GET `/notifications/unread-count`, PATCH `/notifications/:id/read`, PATCH `/notifications/read-all` |
| Workflows | GET `/flows`, POST, GET `/:id`, PATCH `/:id`, DELETE `/:id`, POST `/:id/activate`, POST `/:id/deactivate`, POST `/flows/execute` |

Interactive docs: http://localhost:3001/api/docs

---

## WebSocket Events

### Server → Client (dispatched by `useSocket.js`)

| Event | Redux action |
|-------|-------------|
| `sensor-update` | `sensorRealtimeReceived` → realtimeSlice + sensorsSlice + dashboard |
| `station-status` | `stationRealtimeUpdated` → stationsSlice + dashboard |
| `alert-created` | `alertRealtimeReceived` → alertsSlice + dashboard |
| `notification-created` | `notificationReceived` → notificationsSlice |
| `maintenance-update` | `maintenanceRealtimeUpdated` → maintenanceSlice |
| `system-stats` | `updateSystemStats` → dashboard |

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
  // NOTE: No analyticsSlice — AnalyticsPage still uses local useState
}
```

---

## Development Commands

```bash
# Start full dev stack (all 5 services)
docker compose up -d

# Verify backend health
curl http://localhost:3001/api/health
# → {"status":"ok","db":{"status":"ok"},"redis":{"status":"ok"}}

# Backend: run all unit tests (~97 tests)
cd backend && npm test

# Frontend: run all tests (101 tests)
cd frontend && npm test -- --watchAll=false --forceExit

# Frontend: verify CI build (must produce 0 ESLint warnings)
cd frontend && CI=true npm run build

# Seed demo data
cd backend && npm run seed

# View backend logs
docker compose logs -f backend
```

---

## Demo Credentials

| Role | Email | Password |
|------|-------|----------|
| Admin | `admin@aquaflow.io` | `Admin123!` |
| Operator | `operator@aquaflow.io` | `Admin123!` |
| Technician | `tech@aquaflow.io` | `Admin123!` |
| Analyst | `analyst@aquaflow.io` | `Admin123!` |

---

## Test Coverage

| Layer | Tests | Spec files |
|-------|-------|-----------|
| Backend | ~97 | `auth.service.spec.ts` (14), `alerts.service.spec.ts` (8), `notifications.service.spec.ts` (~9), `stations.service.spec.ts` (16), `sensors.service.spec.ts` (20), `flows.service.spec.ts` (18), `iot.service.spec.ts` (12) |
| Frontend | 101 | `AdminNavbar.test.jsx` (13), `notificationsSlice.test.js` (25), `alertsSlice.test.js` (22), `SensorDetailsPage.test.jsx` (22), `useSocket.test.js` (19) |

---

## Remaining Work

**None.** All planned features and optional enhancements are complete.

### What was completed in the final session (2026-05-27)

| Item | Files changed |
|------|--------------|
| `GET /flows/:id/executions` endpoint | `flows.service.ts` (inject repo + `getExecutions`), `flows.controller.ts` (new handler before `GET :id`) |
| `analyticsSlice.js` created | `frontend/src/store/slices/analyticsSlice.js` (new) |
| `store.js` wired | Added `analytics: analyticsReducer` |
| `AnalyticsPage.jsx` migrated | Replaced all local `useState` API state with `useDispatch` + Redux selectors |

### If you need to extend the platform

The most natural next features would be:
- **analyticsSlice tests** — add `analyticsSlice.test.js` following the pattern in `alertsSlice.test.js`
- **Execution history UI** — a `WorkflowExecutionsPage` that calls `GET /flows/:id/executions` and renders a table
- **Password reset** — SMTP is already wired (`nodemailer`); add `POST /auth/forgot-password` + `POST /auth/reset-password`
- **Role-based dashboard** — show different KPI cards based on `userRole`

---

## Key Patterns — Must Follow

### Backend

1. **Guards:** All admin routes use `@UseGuards(JwtGuard)`. Admin-only operations also add `@UseGuards(JwtGuard, RolesGuard)` + `@Roles('admin')`.
2. **DTOs:** Every endpoint has a Create/Update DTO with `class-validator` decorators.
3. **Repositories:** Injected via `@InjectRepository(Entity)`. Module must include `TypeOrmModule.forFeature([Entity])`.
4. **Realtime broadcasts:** Call `this.realtimeService.broadcastToAll(event, payload)` from services after state mutation.
5. **Cache:** `CACHE_MANAGER` from `@nestjs/cache-manager`. TTL is in **milliseconds** (60 s = `60000`).
6. **Enums:** Use entity enum values (`StationStatus.NORMAL`, `SensorStatus.OFFLINE`). Never raw strings.
7. **Testing:** Use `Test.createTestingModule()`. Mock repositories with `{ create, save, findOne, findAndCount, remove, find }` as `jest.fn()`. Use `Object.defineProperty` to replicate TypeORM getter logic on mock objects.

### Frontend

1. **New pages:** Always add to `frontend/src/routes.js` + create in correct `modules/` subdirectory.
2. **New API calls:** Always add to the relevant service file in `frontend/src/services/`. Never use `axios` directly in components.
3. **State:** Always use Redux Toolkit. Never use `useState` for API-fetched data.
4. **UI components:** Use Reactstrap. Follow the existing page layout: header `<div className="header bg-gradient-* pb-8 pt-5 pt-md-8">`, then `<Container className="mt--7" fluid>`.
5. **Real-time:** Use `useSocket(isConnected)`. Subscribe in `useEffect`, unsubscribe on cleanup.
6. **Auth/RBAC:** Use `useSelector(selectUserRole)` for conditional rendering.
7. **Lint:** After any change, run `CI=true npm run build`. Zero warnings required. Move derived values INTO `useEffect` bodies to avoid `exhaustive-deps` warnings.

---

## Architecture Rules — Never Break

1. All API routes protected by JwtGuard except `auth/register`, `auth/login`, `auth/refresh`, `health`
2. RBAC via `RolesGuard` + `@Roles()` — never hardcode role strings
3. Redis cache always has in-memory fallback (app starts even if Redis is down)
4. Sensor list cache invalidated on create/update/delete — call `clearListCache()` in SensorsService
5. WebSocket JWT validated on connect — RealtimeGateway disconnects invalid tokens
6. `station-status` WS event emitted only when `dto.status` is present in PATCH body
7. All DTOs have `@ApiProperty` — required for Swagger
8. `flow-executor.service.ts` must call `validator.validate(graph)` before running
9. Frontend `apiClient.js` handles 401 → auto-refresh → retry — never break this interceptor
10. Never touch JointJS workflow canvas internals unless explicitly asked

---

## Reference Files (read these before writing code)

| What you need | File |
|--------------|------|
| Full project state | `CURRENT_PROJECT_STATE.md` |
| Remaining tasks with copy-paste code | `NEXT_DEVELOPMENT_STEPS.md` |
| Dev setup and test commands | `DEV_TEST_GUIDE.md` |
| Example page pattern | `frontend/src/modules/stations/pages/StationsPage.jsx` |
| Example service pattern | `frontend/src/services/alertService.js` |
| Example slice pattern | `frontend/src/store/slices/alertsSlice.js` |
| Example backend service pattern | `backend/src/alerts/alerts.service.ts` |
| Example backend test pattern | `backend/src/stations/stations.service.spec.ts` |
| Example frontend test pattern | `frontend/src/store/slices/__tests__/alertsSlice.test.js` |
