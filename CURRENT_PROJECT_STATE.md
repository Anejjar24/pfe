# AquaFlow — Current Project State

**Audit date:** 2026-05-25
**Auditor:** Full file-by-file codebase scan (every controller, service, page, slice, Dockerfile)
**Baseline:** Compared against `/old/CURRENT_PROJECT_STATE.md` (dated 2026-05-10, ~30%) and `/old/AI_CONTINUATION_PROMPT.md` (dated 2026-05-25, ~67%)

---

## Executive Summary

AquaFlow is an industrial water-station supervision platform. The core platform is broadly functional end-to-end: authentication, full CRUD for stations/sensors/alerts/maintenance, real-time WebSocket updates, analytics, notifications, and a workflow builder with 14 block types. The previously identified critical fixes (security, Dockerfiles, migration, analytics, notifications, workflow blocks) have all been applied. Several partial features remain open.

**Overall completion: ~67% (~91/135 tracked features)**

The platform is demo-ready for happy-path scenarios. It is not production-ready due to missing health endpoint, incomplete workflow execution logging, missing filter bars, a dead "View all notifications" route, and two stubbed workflow block types.

---

## 1. Repository Layout (Actual — verified by file read)

```
pfe-project/
├── backend/
│   ├── src/
│   │   ├── alerts/
│   │   │   ├── alerts.controller.ts      GET list, GET :id, POST, PATCH :id/ack, PATCH :id/resolve
│   │   │   ├── alerts.service.ts
│   │   │   ├── alerts.service.spec.ts    ✅ unit tests
│   │   │   └── dto/
│   │   ├── analytics/
│   │   │   ├── analytics.controller.ts   GET overview, GET sensors/:id/stats, GET stations/:id/history
│   │   │   ├── analytics.service.ts
│   │   │   └── dto/analytics-query.dto.ts
│   │   ├── auth/
│   │   │   ├── auth.controller.ts        POST register/login/logout/refresh, GET me
│   │   │   ├── auth.service.ts
│   │   │   ├── auth.service.spec.ts      ✅ unit tests
│   │   │   ├── strategies/jwt.strategy.ts
│   │   │   └── dto/
│   │   ├── common/
│   │   │   ├── guards/jwt.guard.ts
│   │   │   ├── guards/roles.guard.ts
│   │   │   ├── decorators/roles.decorator.ts
│   │   │   └── types/workflow.types.ts
│   │   ├── database/
│   │   │   ├── entities/
│   │   │   │   ├── User.entity.ts              4 roles: admin/operator/technician/analyst
│   │   │   │   ├── Station.entity.ts
│   │   │   │   ├── Sensor.entity.ts
│   │   │   │   ├── SensorData.entity.ts
│   │   │   │   ├── Alert.entity.ts
│   │   │   │   ├── Maintenance.entity.ts
│   │   │   │   ├── Notification.entity.ts
│   │   │   │   ├── Workflow.entity.ts          triggerType enum, executionCount, executions[]
│   │   │   │   └── WorkflowExecution.entity.ts DEFINED but NEVER written by any service
│   │   │   ├── migrations/
│   │   │   │   └── 1778543154417-InitialSchema.ts   ✅ full schema, all 9 tables
│   │   │   └── seeds/seed.ts                   5 stations, 15 sensors, 5 alerts, 4 maint. orders
│   │   ├── execution/
│   │   │   ├── engine/
│   │   │   │   ├── node-executor.ts            14 block types; 'api' and 'notification' STUBBED
│   │   │   │   ├── workflow-runner.ts
│   │   │   │   └── execution-context.ts
│   │   │   └── handlers/                       10 real handlers
│   │   ├── flows/
│   │   │   ├── flows.controller.ts             JwtGuard, 6 endpoints
│   │   │   ├── flows.service.ts                DB-persisted via TypeORM
│   │   │   ├── flow-executor.service.ts        NO DB writes — execution never logged
│   │   │   └── flow-validator.service.ts
│   │   ├── iot/
│   │   │   ├── iot.service.ts                  MQTT ingestion + threshold checks
│   │   │   └── mqtt/mqtt.client.ts
│   │   ├── maintenance/                        Full CRUD + spec
│   │   ├── notifications/                      In-app + email + WS broadcast + spec
│   │   ├── realtime/                           Socket.IO gateway + service
│   │   ├── sensors/                            Full CRUD + Redis cache + inject reading endpoint
│   │   ├── stations/                           Full CRUD + WS status emit on update
│   │   ├── app.module.ts                       NO AppController registered — NO /api/health
│   │   └── main.ts                             CORS, ValidationPipe, Swagger at /api/docs
│   ├── test/auth.e2e-spec.ts
│   └── Dockerfile                              Multi-stage; healthcheck tries /api/health then /api/auth/me
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── Blocksidebar/               BlockSidebar, BlockCategory, BlockSearch
│   │   │   ├── canvas/                     FlowCanvas, JointPaper, CanvasToolbar
│   │   │   ├── Navbars/AdminNavbar.js       bell + notifications dropdown; "View all notifications"
│   │   │   │                               links to /admin/notifications (DEAD ROUTE)
│   │   │   ├── nodes/                      4 node components
│   │   │   └── properties/                 NodeEditorModal, PropertiesPanel, PropertyField
│   │   ├── data/blocks.js                  14 block types (6 Generic + 8 Industrial/Integration)
│   │   ├── engine/                         autosaveManager, graphSerializer/Deserializer, executorClient
│   │   ├── hooks/
│   │   │   ├── useSocket.js                5 WS events → dispatches to 9 slices
│   │   │   ├── useWorkflowEditor.js
│   │   │   ├── useAutosave.js
│   │   │   └── useLogout.js
│   │   ├── layouts/
│   │   │   ├── Admin.js                    routes.js routes + 2 detail routes
│   │   │   └── Auth.js
│   │   ├── modules/
│   │   │   ├── alerts/pages/AlertsPage.jsx          ✅ Full — severity/status filters, ack, resolve
│   │   │   ├── analytics/pages/AnalyticsPage.jsx    ✅ Full — KPIs, doughnuts, sensor stats+chart
│   │   │   ├── auth/pages/Login.jsx, Register.jsx   ✅
│   │   │   ├── auth/components/ProtectedRoute.jsx   ✅
│   │   │   ├── dashboard/pages/DashboardPage.jsx    🔶 No trend charts; "Operational Focus" placeholder
│   │   │   ├── maintenance/pages/MaintenancePage.jsx 🔶 No filter bar; no assignedTo field
│   │   │   ├── monitoring/pages/MonitoringPage.jsx   🔶 No filter bar; loads all sensors
│   │   │   ├── monitoring/pages/SensorDetailsPage.jsx ✅ Line chart + limit picker + stats
│   │   │   ├── stations/pages/StationsPage.jsx       ✅ Full — search/status/type filters, CRUD, delete
│   │   │   └── stations/pages/StationDetailsPage.jsx 🔶 No analytics chart; uses local state not Redux
│   │   ├── pages/BuilderPage.jsx           Actual workflow builder implementation
│   │   ├── views/builder/BuilderPage.jsx   Re-export wrapper of pages/BuilderPage.jsx
│   │   ├── views/examples/Profile.js       Argon stub — no real data binding to auth API
│   │   ├── views/test.js                   "Diagnostics" — renders BuilderPage inside a Card
│   │   ├── routes.js                       8 sidebar routes + hidden /test route + auth routes
│   │   ├── services/                       9 API service files + apiClient.js + authSession.js
│   │   └── store/
│   │       ├── store.js                    9 slices registered
│   │       └── slices/                     9 slice files + 2 test files
│   ├── Dockerfile                          Multi-stage nginx, REACT_APP_API_URL baked at build time
│   └── nginx.conf
├── docker-compose.yml                      5 services with healthchecks
├── mosquitto/config/mosquitto.conf
└── old/                                    Superseded documentation — do not edit
```

---

## 2. Tech Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| Backend | NestJS 10, Node 20, TypeScript | |
| Database | PostgreSQL 15 via TypeORM | 9 entities, 1 migration |
| Cache | Redis 7 (cache-manager-redis-store) | In-memory fallback when Redis unavailable |
| MQTT | Eclipse Mosquitto 2 | Topic `sensor/+/data`; pump/station control publish |
| Auth | JWT access (1h) + refresh (7d) | Redis denylist on logout |
| Real-time | Socket.IO | 5 server→client events |
| API docs | Swagger/OpenAPI | `/api/docs`, all DTOs annotated |
| Frontend | React 18 + React Router 6 | CRA build |
| State | Redux Toolkit | 9 slices |
| UI | Argon Dashboard React 1.2.4 | Reactstrap / Bootstrap 4 |
| Charts | Chart.js 2 via react-chartjs-2 | Line, Doughnut, Bar |
| Workflow canvas | JointJS | Drag-and-drop |
| Container | Docker + Docker Compose | 5-service stack |

---

## 3. Full Backend API Surface

### Auth
| Method | Path | Guard | Body / Returns |
|--------|------|-------|---------------|
| POST | `/api/auth/register` | none | `{ email, password, firstname, lastname }` |
| POST | `/api/auth/login` | none | `{ email, password }` → `{ access_token, refresh_token, user }` |
| GET | `/api/auth/me` | JwtGuard | → current user |
| POST | `/api/auth/logout` | JwtGuard | `{ refresh_token }` |
| POST | `/api/auth/refresh` | none | `{ refresh_token }` → `{ access_token, refresh_token }` |

### Stations (JwtGuard + RolesGuard)
| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| GET | `/api/stations` | any | query: page, limit, search, status, type |
| GET | `/api/stations/:id` | any | returns sensors + alerts + maintenances |
| POST | `/api/stations` | operator | |
| PATCH | `/api/stations/:id` | operator | emits `station-status` WS if status in body |
| DELETE | `/api/stations/:id` | admin | 204 |

### Sensors (JwtGuard + RolesGuard)
| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| GET | `/api/sensors` | any | query: page, limit, stationId, type, status, search; Redis-cached 60 s |
| GET | `/api/sensors/:id` | any | with station + recent alerts |
| GET | `/api/sensors/:id/data` | any | query: limit (default 100) |
| POST | `/api/sensors` | operator | |
| PATCH | `/api/sensors/:id` | operator | invalidates list cache |
| DELETE | `/api/sensors/:id` | admin | 204, invalidates list cache |
| POST | `/api/sensors/:id/reading` | operator | **Manual inject** — same as MQTT message; updates lastReading, saves SensorData |

### Alerts (JwtGuard + RolesGuard)
| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| GET | `/api/alerts` | any | query: page, limit, severity, status |
| GET | `/api/alerts/:id` | any | |
| POST | `/api/alerts` | operator | broadcasts `alert-created` WS |
| PATCH | `/api/alerts/:id/acknowledge` | technician | |
| PATCH | `/api/alerts/:id/resolve` | technician | |

### Maintenance (JwtGuard + RolesGuard)
| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| GET | `/api/maintenance` | any | query: page, limit, status, priority |
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
| POST | `/api/flows/execute` | `{ graph, input }` ad-hoc run |

### Analytics (JwtGuard only)
| Method | Path | Query Params |
|--------|------|-------------|
| GET | `/api/analytics/overview` | — |
| GET | `/api/analytics/sensors/:id/stats` | from, to (ISO 8601) |
| GET | `/api/analytics/stations/:id/history` | from, to, granularity=hour\|day |

### Notifications (JwtGuard only)
| Method | Path | Notes |
|--------|------|-------|
| GET | `/api/notifications` | query: page, limit |
| GET | `/api/notifications/unread-count` | → `{ count }` |
| PATCH | `/api/notifications/read-all` | → `{ updated }` |
| PATCH | `/api/notifications/:id/read` | |

**No `/api/health` endpoint exists** — `app.module.ts` has no `controllers:` array and no `AppController`.

---

## 4. Known Bugs

### 🔴 Bug 1: `api` and `notification` workflow blocks return stub data
**File:** `backend/src/execution/engine/node-executor.ts` lines 60–61
```typescript
case 'api':          return { request: node.data, input, mocked: true };
case 'notification': return { notified: true, channel: node.data?.channel, input };
```
These blocks appear functional in the UI but silently return fake responses. No real HTTP call or notification is sent.

### 🔴 Bug 2: Workflow execution never persisted to DB
**File:** `backend/src/flows/flow-executor.service.ts`
The entire service is 17 lines: `validator.validate(graph)` then `runner.run(graph, input)`. The `WorkflowExecution` entity is never written, so execution history, duration tracking, and error logs are lost.

### 🔴 Bug 3: `/api/health` endpoint missing
**File:** `backend/src/app.module.ts` (no AppController)
The Docker healthcheck calls `/api/health` (→ 404), then falls back to `/api/auth/me` (→ 401 JSON body). The fallback grep matches non-empty output so the container is reported healthy, but the check is misleading and fragile.

### 🔴 Bug 4: "View all notifications" is a dead link
**File:** `frontend/src/components/Navbars/AdminNavbar.js` line 228
Links to `/admin/notifications` which is not registered in `routes.js` or `Admin.js`. Clicking it redirects to `/admin/dashboard`.

### 🔶 Bug 5: MonitoringPage has no sensor filter bar
**File:** `frontend/src/modules/monitoring/pages/MonitoringPage.jsx` line 78
Calls `dispatch(fetchSensors())` with no filter params. Backend supports `stationId`, `type`, `status`, `search`.

### 🔶 Bug 6: StationDetailsPage uses local state and has no analytics chart
**File:** `frontend/src/modules/stations/pages/StationDetailsPage.jsx`
Uses `useState` + direct service calls instead of Redux. Does not call `analyticsService.getStationHistory()` even though the API exists.

### 🔶 Bug 7: MaintenancePage missing filter bar and `assignedTo` field
**File:** `frontend/src/modules/maintenance/pages/MaintenancePage.jsx`
`initialForm` has no `assignedTo` field; no status/priority filter controls.

### 🔶 Bug 8: Demo credentials use `.local` domain, not `.io`
**File:** `backend/src/database/seeds/seed.ts`
Actual: `admin@aquaflow.local / Admin123!` — old docs incorrectly stated `admin@aquaflow.io`.

---

## 5. Infrastructure Status

| Item | Status | Notes |
|------|--------|-------|
| docker-compose.yml | ✅ | 5 services, healthchecks, volumes, shared network |
| backend/Dockerfile | ✅ | Multi-stage, non-root user `aquaflow` |
| frontend/Dockerfile | ✅ | Multi-stage nginx, ARG build-time env vars |
| TypeORM migration | ✅ | `1778543154417-InitialSchema.ts` |
| Seed script | ✅ | `npm run seed` in backend package.json |
| Redis cache | ✅ | Auth denylist + sensor list cache; in-memory fallback |
| Swagger | ✅ | `/api/docs` with all DTOs annotated |
| MQTT | ✅ | Mosquitto 2; subscribe + publish working |
| `/api/health` | ❌ | No endpoint; healthcheck uses 401 fallback |
| Backend CI/CD | ❌ | No pipeline; only frontend `.github/workflows` exists |
| Backend unit tests | 🔶 | 3 spec files: alerts, auth, notifications |
| Backend E2E tests | 🔶 | auth.e2e-spec.ts only |
| Frontend unit tests | 🔶 | AdminNavbar.test.jsx, notificationsSlice.test.js |

---

## 6. Overall Completion by Domain

| Domain | ✅ Complete | 🔶 Partial | ❌/🔴 Missing/Broken | % |
|--------|------------|-----------|---------------------|---|
| Auth & Security | 8 | 1 | 3 | ~67% |
| Station Management | 7 | 2 | 3 | ~58% |
| Sensor Monitoring | 9 | 3 | 3 | ~60% |
| Alerts | 7 | 2 | 3 | ~58% |
| Maintenance | 5 | 3 | 3 | ~45% |
| Dashboard | 4 | 1 | 3 | ~50% |
| Analytics | 6 | 1 | 3 | ~60% |
| Workflow Builder | 11 | 0 | 5 | ~69% |
| Real-time | 8 | 1 | 1 | ~80% |
| Notifications | 6 | 0 | 2 | ~75% |
| Infrastructure/DevOps | 10 | 4 | 3 | ~59% |
| **TOTAL** | **~91** | **~18** | **~32** | **~67%** |
