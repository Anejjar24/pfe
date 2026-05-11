# AquaFlow Current Project State Audit

# AquaFlow — Current Project State (Full Audit)

**Audit date:** 2026-05-10
**Audited by:** Senior Software Architect (AI-assisted code scan)
**Workspace:** `pfe/` (extracted from `pfe1.rar`)
**Application code:** `pfe-backend` (NestJS 10), `pfe-frontend` (React 18, CRA, Argon Dashboard + JointJS workflow builder)

---

## Executive Summary

AquaFlow is **~55% implemented** relative to the four-phase roadmap defined in `IMPLEMENTATION_ROADMAP.md`.

**Phase 1 (Core Infrastructure):** ✅ **Largely complete** — PostgreSQL/TypeORM, JWT auth with refresh tokens, RBAC guards, Socket.IO gateway with JWT handshake, MQTT client, Redux Toolkit with 7 slices, Axios API client with refresh interceptor.

**Phase 2 (CRUD Modules & Pages):** ⚠️ **Partially complete** — Dashboard, Stations, Monitoring, Maintenance pages exist with REST integration and real-time hooks. Alerts page is **empty** (0 bytes). Create/Edit forms exist for Stations and Sensors only. No delete UI. No station-details or sensor-details page.

**Phase 3 (Workflow Extensions):** ❌ **Mostly missing** — The original workflow builder (JointJS, block registry, execution engine) is preserved and functional. However, industrial blocks (sensor-read, threshold-check, pump-control, etc.) defined in the roadmap are **not added** to `blocks.js`. Workflow persistence is **in-memory only** (`Map`); the `Workflow` TypeORM entity exists but is never used by `FlowsService`. `WorkflowExecution` entity is also unused.

**Phase 4 (Analytics, GIS, Reporting):** ❌ **Absent** — No analytics dashboard, no GIS map view, no export system, no reporting module, no email/SMS notifications, no predictive maintenance.

---

## Architecture Summary (As Implemented)

### Backend (`pfe-backend`)


| Aspect      | Detail                                                                                               |
| ----------- | ---------------------------------------------------------------------------------------------------- |
| Framework   | NestJS 10, TypeScript, strict mode                                                                   |
| Entry point | `main.ts` — global prefix `api`, port 3001, CORS from `FRONTEND_URL`, global `ValidationPipe`       |
| Database    | TypeORM + PostgreSQL 15 (Docker),`synchronize: false` in config                                      |
| Auth        | JWT access token (1h) + refresh token (7d), bcrypt hashing                                           |
| Realtime    | Socket.IO gateway with JWT validation on connect                                                     |
| IoT         | MQTT client (mqtt package), subscribes to`sensors/+/data`, `sensors/+/status`, `devices/+/heartbeat` |
| API prefix  | All routes under`/api/`                                                                              |

**Registered NestJS Modules:**


| Module            | File                                    | Status                            |
| ----------------- | --------------------------------------- | --------------------------------- |
| DatabaseModule    | `src/database/database.module.ts`       | ✅ Implemented                    |
| AuthModule        | `src/auth/auth.module.ts`               | ✅ Implemented                    |
| RealtimeModule    | `src/realtime/realtime.module.ts`       | ✅ Implemented                    |
| IotModule         | `src/iot/iot.module.ts`                 | ✅ Implemented                    |
| StationsModule    | `src/stations/stations.module.ts`       | ✅ Implemented                    |
| SensorsModule     | `src/sensors/sensors.module.ts`         | ✅ Implemented                    |
| AlertsModule      | `src/alerts/alerts.module.ts`           | ✅ Implemented                    |
| MaintenanceModule | `src/maintenance/maintenance.module.ts` | ✅ Implemented                    |
| FlowsModule       | `src/flows/flows.module.ts`             | ⚠️ Implemented (in-memory only) |

**Missing modules per roadmap:**

- `AnalyticsModule` — absent
- `ReportsModule` — absent
- `NotificationsModule` — absent (entity exists, module does not)
- `UsersModule` (admin user management) — absent

---

### Backend — Implemented APIs

**Auth (`/api/auth`)**


| Method | Route                | Guard    | Status |
| ------ | -------------------- | -------- | ------ |
| POST   | `/api/auth/register` | Public   | ✅     |
| POST   | `/api/auth/login`    | Public   | ✅     |
| POST   | `/api/auth/refresh`  | Public   | ✅     |
| POST   | `/api/auth/logout`   | JwtGuard | ✅     |
| GET    | `/api/auth/me`       | JwtGuard | ✅     |

**Stations (`/api/stations`)**


| Method | Route               | Guard                     | Status |
| ------ | ------------------- | ------------------------- | ------ |
| GET    | `/api/stations`     | JwtGuard                  | ✅     |
| POST   | `/api/stations`     | JwtGuard + Admin/Operator | ✅     |
| GET    | `/api/stations/:id` | JwtGuard                  | ✅     |
| PATCH  | `/api/stations/:id` | JwtGuard + Admin/Operator | ✅     |
| DELETE | `/api/stations/:id` | JwtGuard + Admin          | ✅     |

**Sensors (`/api/sensors`)**


| Method | Route                   | Guard                     | Status |
| ------ | ----------------------- | ------------------------- | ------ |
| GET    | `/api/sensors`          | JwtGuard                  | ✅     |
| POST   | `/api/sensors`          | JwtGuard + Admin/Operator | ✅     |
| GET    | `/api/sensors/:id`      | JwtGuard                  | ✅     |
| GET    | `/api/sensors/:id/data` | JwtGuard                  | ✅     |
| PATCH  | `/api/sensors/:id`      | JwtGuard + Admin/Operator | ✅     |
| DELETE | `/api/sensors/:id`      | JwtGuard + Admin          | ✅     |

**Alerts (`/api/alerts`)**


| Method | Route                         | Guard    | Status |
| ------ | ----------------------------- | -------- | ------ |
| GET    | `/api/alerts`                 | JwtGuard | ✅     |
| POST   | `/api/alerts`                 | JwtGuard | ✅     |
| GET    | `/api/alerts/:id`             | JwtGuard | ✅     |
| PATCH  | `/api/alerts/:id/acknowledge` | JwtGuard | ✅     |
| PATCH  | `/api/alerts/:id/resolve`     | JwtGuard | ✅     |

**Maintenance (`/api/maintenance`)**


| Method | Route                  | Guard                                | Status |
| ------ | ---------------------- | ------------------------------------ | ------ |
| GET    | `/api/maintenance`     | JwtGuard                             | ✅     |
| POST   | `/api/maintenance`     | JwtGuard + Admin/Operator/Technician | ✅     |
| GET    | `/api/maintenance/:id` | JwtGuard                             | ✅     |
| PATCH  | `/api/maintenance/:id` | JwtGuard                             | ✅     |
| DELETE | `/api/maintenance/:id` | JwtGuard + Admin                     | ✅     |

**Flows (`/api/flows`)** ⚠️


| Method | Route                    | Guard             | Status             |
| ------ | ------------------------ | ----------------- | ------------------ |
| GET    | `/api/flows`             | **None (public)** | ⚠️ No auth guard |
| POST   | `/api/flows`             | **None (public)** | ⚠️ No auth guard |
| GET    | `/api/flows/:id`         | **None (public)** | ⚠️ No auth guard |
| PUT    | `/api/flows/:id`         | **None (public)** | ⚠️ No auth guard |
| DELETE | `/api/flows/:id`         | **None (public)** | ⚠️ No auth guard |
| POST   | `/api/flows/:id/execute` | **None (public)** | ⚠️ No auth guard |

**Missing APIs per roadmap:**

- `GET /api/analytics/overview` — absent
- `GET /api/analytics/stations/:id` — absent
- `GET /api/sensors/:id/statistics` — absent
- `GET /api/reports` — absent
- `GET /api/users` (admin management) — absent
- `POST /api/notifications` — absent

---

### Backend — Database Entities


| Entity            | File                                            | DB Table              | Status                                            |
| ----------------- | ----------------------------------------------- | --------------------- | ------------------------------------------------- |
| User              | `database/entities/User.entity.ts`              | `users`               | ✅ Full                                           |
| Station           | `database/entities/Station.entity.ts`           | `stations`            | ✅ Full                                           |
| Sensor            | `database/entities/Sensor.entity.ts`            | `sensors`             | ✅ Full                                           |
| SensorData        | `database/entities/SensorData.entity.ts`        | `sensor_data`         | ✅ Full                                           |
| Alert             | `database/entities/Alert.entity.ts`             | `alerts`              | ✅ Full                                           |
| Maintenance       | `database/entities/Maintenance.entity.ts`       | `maintenance`         | ✅ Full                                           |
| Workflow          | `database/entities/Workflow.entity.ts`          | `workflows`           | ⚠️ Entity defined, never used by FlowsService   |
| WorkflowExecution | `database/entities/WorkflowExecution.entity.ts` | `workflow_executions` | ⚠️ Entity defined, never used                   |
| Notification      | `database/entities/Notification.entity.ts`      | `notifications`       | ⚠️ Entity defined, no module/service/controller |

**Migrations:** `database/migrations/` directory exists but contains only a `README.md`. No actual migration files authored. Production deployment requires TypeORM migrations to be generated and run.

---

### Backend — Realtime & IoT

**Socket.IO Gateway (`RealtimeGateway`):**

- Validates JWT from `handshake.auth.token` on connection
- Disconnects unauthenticated clients
- Supports `subscribe` / `unsubscribe` / `ping` messages
- Broadcasts via `RealtimeService.broadcastToAll()`

**Events emitted by backend:**


| Event            | Source          | Payload                                                              |
| ---------------- | --------------- | -------------------------------------------------------------------- |
| `sensor-update`  | `IotService`    | `{sensorId, stationId, value, timestamp, thresholdViolated, status}` |
| `alert-created`  | `AlertsService` | `{id, severity, message, stationId, station, sensorId, timestamp}`   |
| `station-status` | Not yet emitted | — (planned)                                                         |

**MQTT Client:**

- Connects to `MQTT_BROKER_URL` (default `mqtt://localhost:1883`)
- Subscribes to: `sensors/+/data`, `sensors/+/status`, `devices/+/heartbeat`
- Parses sensor data and calls `IotService.processSensorData(sensorId, value)`
- Publishes via `MqttClient.publish()` — no consumers yet call this for outbound commands

---

### Frontend (`pfe-frontend`)


| Aspect           | Detail                                                      |
| ---------------- | ----------------------------------------------------------- |
| Framework        | React 18, Create React App                                  |
| Router           | React Router 6,**HashRouter** (`#/`)                        |
| UI Kit           | Reactstrap + Argon Dashboard Bootstrap theme                |
| Workflow Editor  | JointJS (`@joint/core`), existing implementation preserved  |
| State Management | Redux Toolkit                                               |
| HTTP Client      | Axios via`services/apiClient.js` (with refresh interceptor) |
| WebSocket        | Socket.IO client via`hooks/useSocket.js`                    |

---

### Frontend — Implemented Pages & Routes


| Route                 | Component             | Real-time     | API Connected                                  | Status                         |
| --------------------- | --------------------- | ------------- | ---------------------------------------------- | ------------------------------ |
| `/admin/dashboard`    | `DashboardPage.jsx`   | ✅`useSocket` | ✅ fetchStations, fetchSensors, fetchAlerts    | ✅                             |
| `/admin/builder`      | `BuilderPage.jsx`     | ❌            | ✅ workflowApi (fetch-based)                   | ✅ (in-memory)                 |
| `/admin/stations`     | `StationsPage.jsx`    | ❌            | ✅ fetchStations, createStation, updateStation | ⚠️ No delete UI              |
| `/admin/monitoring`   | `MonitoringPage.jsx`  | ✅`useSocket` | ✅ fetchSensors                                | ⚠️ No charts/live graph      |
| `/admin/alerts`       | `AlertsPage.jsx`      | ❌            | ❌                                             | ❌**Empty file (0 bytes)**     |
| `/admin/maintenance`  | `MaintenancePage.jsx` | ❌            | ✅ fetchMaintenance                            | ⚠️ Read-only, no create/edit |
| `/admin/user-profile` | `Profile.js`          | ❌            | ❌                                             | ⚠️ Argon template stub       |
| `/auth/login`         | `Login.jsx`           | ❌            | ✅ loginUser thunk                             | ✅                             |
| `/auth/register`      | `Register.jsx`        | ❌            | ✅ registerUser thunk                          | ✅                             |

**Missing pages per roadmap:**

- Station Details page (`/admin/stations/:id`)
- Create Station page (exists as modal, not dedicated route)
- Sensor Details / Live Chart page
- Analytics Dashboard
- GIS Map view
- Reports page
- User Management page (admin)

---

### Frontend — Redux Store


| Slice         | File                               | State Shape                                                        | Status                             |
| ------------- | ---------------------------------- | ------------------------------------------------------------------ | ---------------------------------- |
| `auth`        | `store/slices/authSlice.js`        | user, accessToken, refreshToken, isAuthenticated, isLoading, error | ✅ Full                            |
| `stations`    | `store/slices/stationsSlice.js`    | items, meta, filters, isLoading, isSaving, error                   | ✅ Full                            |
| `sensors`     | `store/slices/sensorsSlice.js`     | items, isLoading, error                                            | ✅ Full                            |
| `alerts`      | `store/slices/alertsSlice.js`      | items, isLoading, error                                            | ✅ Full                            |
| `maintenance` | `store/slices/maintenanceSlice.js` | items, isLoading, error                                            | ✅ Full                            |
| `realtime`    | `store/slices/realtimeSlice.js`    | isConnected, sensorUpdates, alertReceived, stationStatus           | ✅ Full                            |
| `dashboard`   | `store/slices/dashboardSlice.js`   | derived/computed dashboard state                                   | ✅ Full                            |
| `ui`          | —                                 | theme, sidebar state                                               | ❌**Missing** (planned in roadmap) |

---

### Frontend — Services & Hooks

**API Services:**


| Service              | File                             | Backend Endpoint                               | Status              |
| -------------------- | -------------------------------- | ---------------------------------------------- | ------------------- |
| `authService`        | `services/authService.js`        | `/api/auth/*`                                  | ✅                  |
| `stationService`     | `services/stationService.js`     | `/api/stations/*`                              | ✅                  |
| `sensorService`      | `services/sensorService.js`      | `/api/sensors/*`                               | ✅                  |
| `alertService`       | `services/alertService.js`       | `/api/alerts/*`                                | ✅                  |
| `maintenanceService` | `services/maintenanceService.js` | `/api/maintenance/*`                           | ✅                  |
| `workflowApi`        | `services/workflowApi.js`        | `/api/flows/*` (fetch-based, separate env var) | ⚠️ Env drift risk |

**Custom Hooks:**


| Hook                | File                         | Status                                             |
| ------------------- | ---------------------------- | -------------------------------------------------- |
| `useSocket`         | `hooks/useSocket.js`         | ✅ Connects to Socket.IO, dispatches Redux actions |
| `useWorkflowEditor` | `hooks/useWorkflowEditor.js` | ✅ JointJS graph management                        |
| `useJointGraph`     | `hooks/useJointGraph.js`     | ✅                                                 |
| `useAutosave`       | `hooks/useAutosave.js`       | ✅                                                 |
| `useLogout`         | `hooks/useLogout.js`         | ✅                                                 |
| `useAuth`           | —                           | ❌ Missing (referenced in roadmap)                 |
| `useFetch`          | —                           | ❌ Missing                                         |
| `useLocalStorage`   | —                           | ❌ Missing                                         |
| `useTheme`          | —                           | ❌ Missing                                         |

---

### Frontend — Workflow Builder

The original workflow builder is **fully preserved** and functional:


| Component                                                     | Status |
| ------------------------------------------------------------- | ------ |
| `BlockSidebar` — drag & drop panel                           | ✅     |
| `FlowCanvas` / `JointPaper` — JointJS canvas                 | ✅     |
| `CanvasToolbar` — run/save/clear                             | ✅     |
| `NodeEditorModal` / `PropertiesPanel` — property editing     | ✅     |
| `graphSerializer` / `graphDeserializer` — JSON ↔ graph      | ✅     |
| `workflowExecutorClient` — client-side execution trigger     | ✅     |
| `blockRegistry` / `blockFactory` — block registration system | ✅     |
| `autosaveManager` — debounced save                           | ✅     |

**Block types in `data/blocks.js`:**


| Block               | Type              | Status                       |
| ------------------- | ----------------- | ---------------------------- |
| Input               | `input`           | ✅ (generic)                 |
| Action              | `action`          | ✅ (generic math/string ops) |
| Decision            | `decision`        | ✅ (generic comparator)      |
| Output              | `output`          | ✅ (generic)                 |
| **Sensor Read**     | `sensor-read`     | ❌ Missing                   |
| **Threshold Check** | `threshold-check` | ❌ Missing                   |
| **Pump Control**    | `pump-control`    | ❌ Missing                   |
| **Alert Trigger**   | `alert-trigger`   | ❌ Missing                   |
| **MQTT Publish**    | `mqtt-publish`    | ❌ Missing                   |
| **Station Control** | `station-control` | ❌ Missing                   |

---

### Infrastructure & Deployment


| Component          | Config                                             | Status                                    |
| ------------------ | -------------------------------------------------- | ----------------------------------------- |
| PostgreSQL 15      | `docker-compose.yml`, port 5432                    | ✅ Configured                             |
| Redis 7            | `docker-compose.yml`, port 6379                    | ⚠️ Configured but**unused** in app code |
| Mosquitto MQTT     | `docker-compose.yml`, ports 1883/9001              | ✅ Configured                             |
| Backend container  | Not in docker-compose                              | ❌ No backend Dockerfile                  |
| Frontend container | Not in docker-compose                              | ❌ No frontend Dockerfile                 |
| Frontend build     | `build/` directory present                         | ✅ Built artifact exists                  |
| `.env` files       | `pfe-backend/.env` and `pfe-frontend/.env` present | ✅                                        |

---

## Completed Phases

### Phase 1 — Core Infrastructure ✅ (Largely Complete)

- [X]  TypeORM + PostgreSQL database module
- [X]  9 database entities (User, Station, Sensor, SensorData, Alert, Maintenance, Workflow, WorkflowExecution, Notification)
- [X]  JWT authentication (register, login, refresh, logout, me)
- [X]  JWT strategy + JwtGuard + RolesGuard
- [X]  RBAC decorators (Roles, CurrentUser)
- [X]  Socket.IO gateway with JWT authentication
- [X]  RealtimeService with broadcast methods
- [X]  MQTT client subscribing to sensor topics
- [X]  IotService processing sensor data → DB → WebSocket → Alerts
- [X]  Redux Toolkit store with 7 slices (auth, stations, sensors, alerts, maintenance, realtime, dashboard)
- [X]  Axios API client with JWT bearer + refresh interceptor
- [X]  Docker Compose (PostgreSQL, Redis, Mosquitto)

**Phase 1 Gaps:**

- [ ]  No database migration files generated
- [ ]  Redis unused
- [ ]  `uiSlice` not created

### Phase 2 — Core Modules ⚠️ (Partially Complete)

- [X]  Dashboard page with KPI cards, station overview, alerts feed, realtime stats
- [X]  Stations CRUD page with create/edit modals and filtering
- [X]  Monitoring page with sensor table, create sensor modal, realtime updates
- [X]  Maintenance page (read-only list view)
- [X]  Auth pages (Login, Register) with Redux integration
- [X]  Protected route component
- [ ]  AlertsPage — **EMPTY** (0 bytes, not implemented)
- [ ]  Station Details page
- [ ]  Sensor Details / history chart page
- [ ]  Maintenance create/edit forms
- [ ]  Alert acknowledge/resolve UI
- [ ]  Live sensor chart components (`LiveChart`, `RealtimeGauge`)
- [ ]  Common UI components library (`Common/`, `DataDisplay/`, `Forms/`, `Layout/`)

---

## Missing Phases

### Phase 3 — Workflow Extensions ❌

- [ ]  Industrial workflow blocks (SensorRead, ThresholdCheck, PumpControl, AlertTrigger, MqttPublish)
- [ ]  Workflow persistence to PostgreSQL via `Workflow` entity
- [ ]  Workflow execution history saved to `WorkflowExecution` entity
- [ ]  Workflow execution triggers from IoT events
- [ ]  `FlowsService` refactored from in-memory Map to TypeORM repository
- [ ]  JWT guards added to `/api/flows` routes

### Phase 4 — Advanced Features ❌

- [ ]  Analytics dashboard (consumption trends, pressure analytics, quality metrics)
- [ ]  GIS/map station visualization
- [ ]  Export system (CSV, PDF)
- [ ]  Reports module
- [ ]  Email/SMS notification channels (`NotificationsModule`)
- [ ]  Predictive maintenance algorithms
- [ ]  User management admin panel
- [ ]  Sensor calibration/configuration UI
- [ ]  Dark mode / theme system

---

## Technical Debt & Risks


| Risk                               | Severity | Description                                                                                                                                                               |
| ---------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/api/flows` has no auth           | HIGH     | All flow CRUD + execution endpoints are public. Any anonymous user can create, read, or execute workflows.                                                                |
| FlowsService is in-memory          | HIGH     | Workflows are lost on every server restart.`Workflow` TypeORM entity exists but is unused.                                                                                |
| No DB migrations                   | HIGH     | No migration files in`database/migrations/`. Production deployment requires `synchronize: true` (dangerous) or manual migration authoring.                                |
| AlertsPage is empty                | HIGH     | The`/admin/alerts` page renders nothing. Alerts functionality has no UI despite full backend implementation.                                                              |
| Redis unused                       | MEDIUM   | Redis is declared in Docker Compose but never connected to the NestJS app. Planned for session/cache use.                                                                 |
| Env drift on workflows             | MEDIUM   | `workflowApi.js` uses `REACT_APP_WORKFLOW_API_URL` (fetch-based) while all other services use `REACT_APP_API_URL` (Axios). If env vars diverge, workflows hit wrong host. |
| No uiSlice                         | LOW      | Planned in roadmap for theme/sidebar control; not created.                                                                                                                |
| No test files                      | MEDIUM   | Zero test files found (`*.spec.ts`, `*.test.ts`, `*.test.jsx`).                                                                                                           |
| No backend Dockerfile              | MEDIUM   | Backend cannot be containerized without authoring a Dockerfile.                                                                                                           |
| `station-status` event not emitted | LOW      | `useSocket.js` listens for `station-status` but backend never emits it.                                                                                                   |

Generated from the real codebase in `C:\Users\Grous info\Downloads\pfe` on 2026-05-10. The requested path `C:\Users\DELL\Downloads\pfe-project` was not present in this environment; this repository contains the AquaFlow documents and the `pfe-backend` / `pfe-frontend` applications.

## Executive Summary

AquaFlow is currently an Argon Dashboard React frontend plus a NestJS backend that has been extended with a meaningful subset of the planned AquaFlow Phase 1 and early Phase 2 work. The workflow-builder/editor is still present and functional as a distinct subsystem, but the industrial workflow extension described in the roadmap has not been implemented.

The project is not a blank scaffold. It contains real backend modules for database persistence, JWT auth, RBAC guards, realtime Socket.IO infrastructure, MQTT client setup, station/sensor/alert/maintenance CRUD, and an in-memory flow execution module. The frontend contains Redux Toolkit state, auth screens, protected admin routes, station management, monitoring, alerts, maintenance, dashboard widgets, API services, and Socket.IO client integration.

The biggest gap is that the roadmap's later modules are absent: analytics, reports, GIS map, IoT device management UI, notifications, industrial workflow blocks/handlers, persistent workflow management, testing, and production deployment hardening. Several implemented pieces are also still partial, especially workflow persistence and production migration/deployment hardening.

## Documentation Compared

The following root documentation files were read and compared against code:

- `AQUAFLOW_ARCHITECTURE.md`
- `IMPLEMENTATION_ROADMAP.md`
- `QUICK_START.md`
- `README_AQUAFLOW.md`
- `PROJECT_SETUP.md`
- `PHASE_1_CHECKLIST.md`
- `DELIVERABLES.md`

The documents define a four-phase roadmap:

- Phase 1: database, auth/RBAC, realtime, MQTT, Redux.
- Phase 2: frontend feature modules and CRUD APIs.
- Phase 3: industrial workflow blocks and execution integration.
- Phase 4: analytics, reports, IoT management, notifications, performance, testing, polish.

Current implementation aligns most closely with Phase 1 plus selected Phase 2 CRUD/UI work.

## Current Architecture Summary

### Backend

Backend path: `pfe-backend`.

The backend is a NestJS 10 application with:

- Global API prefix: `/api`
- Default port: `3001`
- Global validation pipe with whitelist and implicit conversion
- CORS configured from `FRONTEND_URL`, defaulting to `http://localhost:3000`
- TypeORM/PostgreSQL database configuration
- JWT authentication and RBAC guards
- Socket.IO realtime service/gateway
- MQTT client wrapper
- Feature modules for stations, sensors, alerts, maintenance, and flows

`AppModule` imports:

- `ConfigModule`
- `DatabaseModule`
- `AuthModule`
- `RealtimeModule`
- `IotModule`
- `StationsModule`
- `SensorsModule`
- `AlertsModule`
- `MaintenanceModule`
- `FlowsModule`

### Frontend

Frontend path: `pfe-frontend`.

The frontend is a Create React App application based on Creative Tim Argon Dashboard React. It uses:

- React 18
- React Router v6
- Redux Toolkit and React Redux
- Reactstrap / Bootstrap / Argon SCSS
- Socket.IO client
- Axios
- JointJS / `@joint/core` for workflow editing

The AquaFlow feature modules that currently exist are:

- `auth`
- `dashboard`
- `stations`
- `monitoring`
- `alerts`
- `maintenance`

Missing planned modules:

- `map`
- `analytics`
- `reports`
- `iot`
- `automation` as a feature module
- `notifications`

## Backend Implementation State

### Implemented Backend Modules


| Module          | State                  | Notes                                                                                                                                                                       |
| --------------- | ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Database        | Partial/strong Phase 1 | TypeORM configured for PostgreSQL with core entities and synchronize enabled outside production. Migration status is documented, but no initial migration is generated yet. |
| Auth            | Partial/strong Phase 1 | Register, login, me, logout, and refresh token endpoint exist. JWT strategy and password hashing exist. Refresh tokens are JWT-based and not yet persisted/rotated.         |
| RBAC            | Partial                | `JwtGuard`, `RolesGuard`, and `Roles` decorator exist and are applied to CRUD modules. Frontend now hides several privileged station/sensor/alert actions by role.          |
| Realtime        | Partial                | Socket.IO gateway and service exist. Subscribe/unsubscribe/ping events exist. Broadcast methods exist. Socket connections now validate JWT tokens from handshake auth.      |
| IoT/MQTT        | Partial                | MQTT client connects, subscribes, parses`sensors/{sensorId}/data`, delegates valid numeric readings to `IotService`, and threshold violations create persistent alerts.     |
| Stations        | Partial/usable         | CRUD controller/service/DTOs exist, protected with JWT/RBAC. Includes pagination/filtering.                                                                                 |
| Sensors         | Partial/usable         | CRUD plus`GET /sensors/:id/data` exists. Sensor data model exists.                                                                                                          |
| Alerts          | Partial/usable         | List, detail, create, acknowledge, resolve exist. Delete/clear endpoint from docs is absent.                                                                                |
| Maintenance     | Partial/usable         | List, detail, create, update, delete exist. Dedicated assignment endpoint is absent.                                                                                        |
| Flows/workflows | Partial                | Existing workflow execution engine and`/flows` endpoints exist, but flow storage is in-memory and not using `Workflow` entities.                                            |
| Analytics       | Missing                | No backend analytics module.                                                                                                                                                |
| Reports         | Missing                | No backend reports module.                                                                                                                                                  |
| Notifications   | Missing                | Notification entity exists, but no notification service/controller/channels.                                                                                                |
| GIS/map         | Missing                | No backend map/geospatial endpoint beyond station latitude/longitude fields.                                                                                                |

### Implemented APIs

Because `main.ts` sets `app.setGlobalPrefix('api')`, actual routes are under `/api`.

Implemented:

- `POST /api/auth/register`
- `POST /api/auth/login`
- `GET /api/auth/me`
- `POST /api/auth/logout`
- `GET /api/stations`
- `GET /api/stations/:id`
- `POST /api/stations`
- `PATCH /api/stations/:id`
- `DELETE /api/stations/:id`
- `GET /api/sensors`
- `GET /api/sensors/:id`
- `GET /api/sensors/:id/data`
- `POST /api/sensors`
- `PATCH /api/sensors/:id`
- `DELETE /api/sensors/:id`
- `GET /api/alerts`
- `GET /api/alerts/:id`
- `POST /api/alerts`
- `PATCH /api/alerts/:id/acknowledge`
- `PATCH /api/alerts/:id/resolve`
- `GET /api/maintenance`
- `GET /api/maintenance/:id`
- `POST /api/maintenance`
- `PATCH /api/maintenance/:id`
- `DELETE /api/maintenance/:id`
- `POST /api/flows`
- `GET /api/flows`
- `GET /api/flows/:id`
- `PUT /api/flows/:id`
- `DELETE /api/flows/:id`
- `POST /api/flows/execute`

Not implemented from roadmap:

- `POST /api/auth/refresh`
- `GET /api/stations/:id/analytics`
- `PUT /api/stations/:id` (PATCH exists instead)
- `PUT /api/sensors/:id` (PATCH exists instead)
- `DELETE /api/alerts/:id`
- `PATCH /api/maintenance/:id/assign`
- `/api/workflows/*` endpoints as documented
- `/api/analytics/*`
- `/api/reports/*`
- `/api/notifications/*`
- `/api/iot/*`

### Implemented Entities

The database entity layer contains:

- `User`
- `Station`
- `Sensor`
- `SensorData`
- `Alert`
- `Maintenance`
- `Workflow`
- `WorkflowExecution`
- `Notification`

This goes beyond the original eight-entity Phase 1 list by including `WorkflowExecution`.

Important caveat: workflow/notification entities exist, but their application modules are not fully implemented. The current flow module stores flows in memory rather than TypeORM.

### Realtime State

Backend:

- `RealtimeGateway` handles Socket.IO connections.
- `RealtimeService` tracks socket connections and user-to-client mappings.
- Server supports `subscribe`, `unsubscribe`, and `ping`.
- Broadcast helpers exist for all clients, rooms, users, and individual clients.

Frontend:

- `useSocket` connects to `REACT_APP_WS_URL` or `http://localhost:3001`.
- It sends token in Socket.IO `auth` and userId in query.
- It subscribes to `dashboard`, `alerts`, `stations`, and `sensors`.
- It handles `sensor-update`, `alert-created`, and `station-status`.

Gaps:

- Backend does not validate the socket token.
- Backend room naming is `channel:${channel}`, but `broadcastToRoom` expects a raw room string. Callers must know to include the `channel:` prefix.
- Some backend emissions use `threshold-alert`, while frontend listens for `alert-created` and not `threshold-alert`.
- Realtime is present, but not consistently connected to all CRUD and workflow events.

### MQTT State

Implemented:

- MQTT dependency installed.
- `MqttClient` connects on module init.
- Subscribes to:
  - `sensors/+/data`
  - `sensors/+/status`
  - `devices/+/heartbeat`
- Supports publishing JSON payloads.

Gaps:

- `handleMessage` parses and logs messages but does not call `IotService.processSensorData`.
- `subscribe(topic, callback)` ignores the callback.
- No MQTT controller/admin API.
- No device registry, payload validation module, dead-letter handling, or reconnect observability.
- Threshold violations broadcast websocket events but do not currently create persistent `Alert` records from MQTT data.

### Workflow Builder / Execution State

Frontend:

- Workflow builder/editor components exist under `components/Blocksidebar`, `components/canvas`, `components/nodes`, `components/properties`, `components/builder`, `registry`, `engine`, and `views/builder`.
- `data/blocks.js` defines generic blocks:
  - input
  - action
  - decision
  - delay
  - api
  - notification
  - output
- `workflowApi.js` can save and execute flows through `/flows` and `/flows/execute`.

Backend:

- Execution engine exists with `WorkflowRunner`, `NodeExecutor`, and handlers for:
  - input
  - action
  - decision
  - output
- `FlowsController` exposes CRUD and execute endpoints.
- `FlowsService` stores flow records in a process-local `Map`.

Gaps:

- Industrial blocks from the roadmap are missing:
  - sensor-trigger
  - threshold-checker
  - alert-sender
  - maintenance-request
  - mqtt-publisher
  - email-notification
  - sms-notification
  - pump-control
  - analytics-processor
  - timer/scheduler
  - station-monitor
- Corresponding backend execution handlers are missing.
- Flow persistence is not durable.
- The planned `/api/workflows` contract is not implemented; current API is `/api/flows`.
- Route `/admin/builder` imports `BuilderPage` but renders `Test`, which is inconsistent.

### Redux / Store State

Implemented slices:

- `auth`
- `dashboard`
- `realtime`
- `stations`
- `sensors`
- `alerts`
- `maintenance`

Not implemented from roadmap:

- `uiSlice`
- explicit workflow/automation slice
- analytics slice
- reports slice
- notifications slice
- map slice
- IoT/device slice

The existing Redux implementation is enough for current auth, dashboard, station, sensor, alert, maintenance, and realtime screens.

### Frontend Pages and Routes

Implemented admin routes:

- `/admin/dashboard`
- `/admin/builder`
- `/admin/stations`
- `/admin/monitoring`
- `/admin/alerts`
- `/admin/maintenance`
- `/admin/test`
- `/admin/user-profile`

Implemented auth routes:

- `/auth/login`
- `/auth/register`

Missing planned pages:

- Station details page
- Create station page as standalone route
- Station analytics page
- Sensor details page
- Alert details page
- Intervention details page
- Technician assignment page
- Map page
- Analytics page
- Trend analysis page
- Anomaly detection page
- Reports page
- Report builder page
- IoT devices page
- Sensor config page
- Notifications page
- Workflow list/details/logs pages

### Deployment and Docker State

Implemented:

- Root `docker-compose.yml` defines:
  - PostgreSQL 15
  - Redis 7
  - Eclipse Mosquitto 2
- Mosquitto config exists under `mosquitto/config/mosquitto.conf`.
- Backend and frontend package scripts exist.

Gaps:

- No Dockerfiles for backend/frontend.
- No production compose file.
- No CI/CD.
- No database migration scripts in source.
- No `.env.example` files were observed in the scanned file list.
- Redis is provisioned but not used by application code.

## Completed Phases

### Phase 1: Core Infrastructure

Mostly complete but not production-complete.

Completed:

- TypeORM/PostgreSQL setup.
- Core entities.
- JWT login/register/me/logout.
- JWT guard and roles guard.
- Socket.IO gateway/service.
- MQTT client wrapper.
- Redux store and core slices.
- Docker Compose infrastructure.

Partial:

- Refresh token endpoint exists, but refresh-token persistence/rotation is still missing.
- Initial TypeORM migration is not generated; migration status is documented in `pfe-backend/src/database/migrations/README.md`.
- MQTT ingestion is wired for `sensors/{sensorId}/data`.
- Socket token validation is implemented.
- No formal automated tests.

### Phase 2: Feature Modules

Partially complete.

Completed/usable:

- Dashboard page/widgets now derive KPI, station overview, and active alert feed from real station/sensor/alert API data.
- Station management list/create/edit UI.
- Monitoring sensor list/create UI.
- Alerts list/acknowledge/resolve UI.
- Maintenance list UI.
- API services for auth/stations/sensors/alerts/maintenance.

Missing/partial:

- Many detailed subpages and reusable component library from the roadmap.
- GIS map module.
- Station analytics.
- Live charts/gauges are minimal or missing.
- Maintenance create/assignment/detail UI.
- CRUD support exists backend-side, but UI is uneven.

### Phase 3: Workflow Extension

Mostly missing.

Present:

- Original/generic workflow builder and generic execution engine.

Missing:

- Industrial blocks.
- Industrial handlers.
- Realtime event-triggered workflow execution.
- Workflow-triggered alert/maintenance/notification/MQTT integrations.
- Durable workflow persistence.

### Phase 4: Advanced Features

Missing.

No analytics module, reports module, notifications module, IoT management UI, caching integration, test suite, or production observability stack is implemented.

## Technical Debt and Inconsistencies

- API naming mismatch: documents describe `/workflows`, code uses `/flows`.
- HTTP method mismatch: documents describe some `PUT` routes; code mostly uses `PATCH`.
- Auth refresh endpoint now exists at `POST /api/auth/refresh`, but refresh-token rotation/persistence remains future hardening.
- Frontend API default is `http://localhost:3001/api`, while some docs mention backend `3000`.
- Socket URL defaults to `http://localhost:3001`, matching backend but not the docs' separate Socket port model.
- MQTT client and IoT service are not fully connected.
- `FlowsService` uses in-memory storage despite workflow entities existing.
- `routes.js` imports `BuilderPage` but the builder route renders `Test`.
- Frontend is still partly Argon template code, with unused examples and template routes/assets.
- `dist/` and frontend `build/` artifacts exist in the workspace; they should generally be ignored/generated, not treated as source of truth.
- Redis is included in infrastructure but unused.
- No tests were found in the scanned source tree.

## Scalability Concerns

- Sensor time-series data is stored via TypeORM without visible partitioning, indexes, retention policy, or aggregation strategy.
- Realtime service stores socket state in memory, so horizontal scaling would require a Socket.IO adapter such as Redis.
- Workflow storage is in memory and will be lost on restart.
- MQTT ingestion lacks backpressure, validation, retry/dead-letter handling, and persistent alert creation.
- TypeORM `synchronize` is enabled outside production; migrations should replace this before serious deployment.
- No caching layer is wired despite Redis being available.
- No health endpoints, metrics, structured logging, or tracing are implemented.

## Verification Performed

- Backend type-check: `npx.cmd tsc --noEmit` succeeded.
- Frontend production build: `npm.cmd run build` succeeded with warnings:
  - unused imports in `src/components/Headers/Header.js`
- Authenticated API verification succeeded for login, refresh, stations, sensors, alerts, and maintenance seeded data.
- Browser automation opened the login page, but could not type into the `type="email"` field due to the in-app browser automation layer; UI data verification was completed through authenticated API/build checks.
