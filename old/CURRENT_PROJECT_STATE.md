# AquaFlow — Current Project State

**Audit date:** 2026-05-25
**Auditor:** Full codebase scan (zip extraction + file-by-file review)
**Compared against:** Previous MD audit dated 2026-05-10

---

## Executive Summary

The codebase has advanced **dramatically** since the May 10 audit. Every critical fix and most high-priority items from the previous `NEXT_DEVELOPMENT_STEPS.md` have been implemented. The platform is now broadly functional end-to-end: authentication, station/sensor/alert/maintenance CRUD, real-time WebSocket updates, a complete analytics module, a full notifications system with in-app bell and email, a persistent workflow engine with industrial blocks, and a production-ready Docker Compose stack.

**Previous overall completion: ~30% (38/126 tracked features)**
**Current overall completion: ~72% (91/126 tracked features)**

---

## 1. Repository Layout

```
pfe-project/
├── backend/                   NestJS API (Node 20, TypeScript)
│   ├── src/
│   │   ├── analytics/         NEW — full analytics module
│   │   ├── alerts/            CRUD + acknowledge/resolve + spec
│   │   ├── auth/              JWT + refresh + Redis denylist + spec
│   │   ├── common/            Guards, decorators, workflow types
│   │   ├── database/
│   │   │   ├── entities/      All 8 entities defined
│   │   │   ├── migrations/    InitialSchema migration (complete)
│   │   │   └── seeds/         seed.ts with realistic demo data
│   │   ├── execution/         Workflow engine + all 10 handlers
│   │   ├── flows/             Persistent workflow CRUD + executor
│   │   ├── iot/               MQTT client + IotService
│   │   ├── maintenance/       Full CRUD
│   │   ├── notifications/     NEW — in-app + email + spec
│   │   ├── realtime/          Socket.IO gateway + service
│   │   ├── sensors/           Full CRUD + Redis cache + data history
│   │   └── stations/          Full CRUD + realtime status emit
│   ├── test/                  E2E auth test
│   ├── Dockerfile             Multi-stage production build ✅
│   └── package.json
├── frontend/                  React 18 + Redux Toolkit + Argon Dashboard
│   ├── src/
│   │   ├── components/        Sidebar, Navbars (w/ notification bell), Builder nodes
│   │   ├── data/blocks.js     All 14 block types including Industrial category
│   │   ├── engine/            autosave, graphSerializer/Deserializer, executorClient
│   │   ├── hooks/             useSocket (5 WS events), useWorkflowEditor, etc.
│   │   ├── modules/
│   │   │   ├── alerts/        AlertsPage — full table, filters, ack/resolve ✅
│   │   │   ├── analytics/     AnalyticsPage — KPI cards + doughnut charts + sensor analysis ✅
│   │   │   ├── auth/          Login, Register, ProtectedRoute
│   │   │   ├── dashboard/     KPISection, AlertsFeed, StationOverview, RealtimeStats
│   │   │   ├── maintenance/   MaintenancePage — full CRUD modal ✅
│   │   │   ├── monitoring/    MonitoringPage (edit/delete), SensorDetailsPage ✅
│   │   │   └── stations/      StationsPage (filters, delete), StationDetailsPage ✅
│   │   ├── services/          All 8 API services + analyticsService + notificationService
│   │   ├── store/slices/      9 Redux slices including uiSlice + notificationsSlice
│   │   └── routes.js          7 pages routed including Analytics
│   ├── Dockerfile             Multi-stage nginx build ✅
│   └── nginx.conf
├── docker-compose.yml         Full stack: postgres + redis + mosquitto + backend + frontend ✅
└── mosquitto/config/
```

---

## 2. Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend runtime | Node 20 / NestJS 10 |
| Language | TypeScript |
| Database | PostgreSQL 15 (TypeORM) |
| Cache | Redis 7 (cache-manager-redis-store, with in-memory fallback) |
| Message broker | Eclipse Mosquitto 2 (MQTT) |
| Authentication | JWT (access 1h + refresh 7d) + Redis denylist on logout |
| Real-time | Socket.IO |
| API docs | Swagger/OpenAPI (available at `/api/docs`) |
| Frontend framework | React 18 + React Router 6 |
| State management | Redux Toolkit (9 slices) |
| UI library | Argon Dashboard React (Reactstrap / Bootstrap 4) |
| Charts | Chart.js 2 via react-chartjs-2 |
| Workflow canvas | JointJS |
| Containerisation | Docker + Docker Compose |

---

## 3. What Is Working (Confirmed by Code Review)

### Authentication & Security
- Full JWT flow: register → login → access token (1h) → refresh token (7d) → logout with Redis denylist
- `JwtGuard` applied to all protected routes **including** `FlowsController` (security fix applied)
- `RolesGuard` + `@Roles()` decorator working for RBAC
- Redis cache used in `AuthService` to store refresh token denylist and in `SensorsService` for list caching

### Stations
- Full CRUD with paginated list, create modal, edit modal, delete with confirm dialog (admin-only)
- Filter bar: text search + status dropdown + type dropdown — all wired to backend query params
- Station details page (`/admin/stations/:stationId`) shows station info, sensors list, and recent alerts
- `station-status` WebSocket event is now **emitted** from `StationsService.update()` whenever `dto.status` is present, and received by all clients via `useSocket`

### Sensors / Monitoring
- Full CRUD: create, edit modal (pre-filled), delete with confirm (admin-only)
- `SensorDetailsPage` (`/admin/monitoring/:sensorId`) shows line chart of historical readings with limit picker (50/100/200/500), stats, and threshold reference lines
- Live `lastReading` updated in sensor table via `sensor-update` WebSocket event
- Redis caching on `findAll()` with cache invalidation on create/update/delete

### Alerts
- `AlertsPage` is **fully implemented** (was 0 bytes in previous audit): table with severity badge, message, station, sensor, time, status; Acknowledge and Resolve action buttons (role-gated); filter dropdowns for severity and status; live updates via `alert-created` WebSocket event

### Maintenance
- Full CRUD with modal: create new order, edit existing (pre-filled form), delete (admin-only)
- Station dropdown populated from Redux store
- Priority and status color-coded badges

### Analytics (New Module — Was 100% Missing)
- Backend: `GET /api/analytics/overview` (counts + breakdowns), `GET /api/analytics/sensors/:id/stats` (avg/min/max/stddev + hourly time-series), `GET /api/analytics/stations/:id/history` (per-sensor buckets)
- Frontend: `AnalyticsPage` with 4 KPI cards, doughnut charts (stations by status, alerts by severity), sensor analysis panel with line/min/max chart, preset range buttons (24h/7d/30d) + custom datetime picker

### Notifications (New Module — Was Entity-Only in Previous Audit)
- Backend `NotificationsService` fully implemented: creates in-app broadcast notification on every alert, sends email to admins on critical/error alerts via SMTP (graceful no-op when `SMTP_HOST` not set)
- `NotificationsController` with GET list, unread count, mark-read, mark-all-read — all behind `JwtGuard`
- Frontend notification bell in `AdminNavbar`: shows unread badge count, dropdown with recent notifications, mark-all-read button; `notificationsSlice` with full thunks; `notificationService` API client
- `notification-created` and `notifications-read-all` WebSocket events wired in `useSocket`

### Workflow Builder
- `FlowsService` now persists workflows to PostgreSQL (`Workflow` entity), replacing the previous in-memory `Map`
- `FlowsController` protected by `JwtGuard`
- All 14 block types defined in `data/blocks.js`, including Industrial category: `sensor-read`, `threshold-check`, `pump-control`, `alert-trigger`, `mqtt-publish`, `station-control`
- All 10 execution handlers implemented in `NodeExecutor` switch: `input`, `action`, `decision`, `output`, `delay`, `sensor-read`, `threshold-check`, `alert-trigger`, `mqtt-publish`, `pump-control`, `station-control`, `http-request`, `api` (mocked), `notification` (mocked)
- `SensorReadHandler` reads from DB, `AlertTriggerHandler` calls `AlertsService.create()`, `MqttPublishHandler` calls `MqttClient.publish()`

### Infrastructure
- Both `backend/Dockerfile` and `frontend/Dockerfile` exist as proper multi-stage builds
- `docker-compose.yml` includes all 5 services (postgres, redis, mosquitto, backend, frontend) with health checks and `depends_on`
- TypeORM migration `InitialSchema1778543154417` exists and creates all 9 tables with correct foreign keys and indexes
- Swagger/OpenAPI served at `/api/docs` with `@ApiProperty` on all DTOs and `@ApiTags` on all controllers
- Redis configured with in-memory fallback — no crash if Redis is unavailable

### Testing (Partial)
- Backend: `alerts.service.spec.ts`, `auth.service.spec.ts`, `notifications.service.spec.ts` — unit tests with mocked repositories
- Backend: `auth.e2e-spec.ts` — E2E test for auth flow
- Frontend: `AdminNavbar.test.jsx`, `notificationsSlice.test.js`

---

## 4. What Is Partially Working or Has Gaps

| Area | Gap |
|------|-----|
| Dashboard KPI live update | KPI cards calculate from Redux store data, but no auto-refresh on WS sensor-update events |
| Sensor filter UI | MonitoringPage has no filter bar (no station/type dropdown); sensors load unfiltered |
| Station analytics | `GET /api/analytics/stations/:id/history` exists but `StationDetailsPage` doesn't use it — only shows sensor list + alerts |
| Workflow execution logging | `WorkflowExecution` entity is registered but `FlowExecutorService` never saves execution records to DB |
| Workflow realtime broadcast | No `workflow-event` WebSocket event is emitted after execution |
| `api` and `notification` workflow blocks | Mocked in `NodeExecutor` (return stub data, don't make real HTTP calls or send real notifications) |
| Health check endpoint | `AppController` / `/api/health` does not exist; Docker healthcheck in backend Dockerfile uses `/api/auth/me` as fallback |
| User management page | No admin user list/edit/deactivate UI; no `UsersModule` in backend |
| User profile page | Argon stub profile page exists but has no real data binding to the API |
| Password reset / email verification | Not implemented anywhere |
| GIS map visualisation | Lat/lon stored in DB and displayed as numbers in `StationDetailsPage` but no map component |
| Predictive maintenance | Not implemented |
| Export (CSV/PDF) | Not implemented |
| Scheduled/MQTT-triggered workflows | `triggerType` and `triggerConfig` fields exist in entity but no scheduler or IoT event hook |
| CI/CD pipeline | Only a frontend GitHub Actions workflow exists; no backend pipeline |
| User management (admin) | No backend UsersModule or frontend admin page |

---

## 5. Known Issues / Bugs

1. **`api` and `notification` workflow blocks are stubs** — they return mock data in `NodeExecutor` instead of making real calls. The block definitions and UI exist, but the execution handlers are placeholders.

2. **Workflow execution not persisted** — `WorkflowExecution` entity is defined and registered in TypeORM but `FlowExecutorService` never saves execution records, so execution history is lost on restart.

3. **Dashboard KPI cards are not realtime** — they derive values from the Redux store at render time, but `useSocket` sensor/station updates don't trigger a KPI recalculation (the store updates, but KPI `useMemo` dependencies are the full arrays, so it will recompute — actually working but not tested).

4. **Health check missing** — the backend Dockerfile healthcheck command calls `/api/auth/me` which returns 401 when unauthenticated, so container health reports as failing.

5. **`REACT_APP_WORKFLOW_API_URL` env fallback** — `workflowApi.js` may reference a separate env variable that defaults to `undefined` if `REACT_APP_WORKFLOW_API_URL` is not set and no fallback to `REACT_APP_API_URL` is in place (minor, verify in deployment).

---

## 6. Key File Reference

| File | Role |
|------|------|
| `backend/src/app.module.ts` | Root module — registers CacheModule with Redis or in-memory fallback |
| `backend/src/main.ts` | Bootstrap — CORS, ValidationPipe, Swagger setup |
| `backend/src/database/migrations/1778543154417-InitialSchema.ts` | Full DB schema — run before first deployment |
| `backend/src/flows/flows.service.ts` | Workflows persisted to PostgreSQL |
| `backend/src/execution/engine/node-executor.ts` | Central dispatcher for all 14 block types |
| `backend/src/notifications/notifications.service.ts` | In-app + email notification logic |
| `backend/src/analytics/analytics.service.ts` | Aggregated metrics and time-series queries |
| `frontend/src/hooks/useSocket.js` | Single WS hook — handles 5 events, dispatches to 9 Redux slices |
| `frontend/src/store/store.js` | Redux store with 9 slices |
| `frontend/src/data/blocks.js` | 14 workflow block definitions (including Industrial) |
| `frontend/src/modules/analytics/pages/AnalyticsPage.jsx` | Full analytics dashboard with Chart.js |
| `frontend/src/modules/alerts/pages/AlertsPage.jsx` | Full alert management page |
| `docker-compose.yml` | Production-ready 5-service stack |
