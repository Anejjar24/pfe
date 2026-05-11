# AI Continuation Prompt for AquaFlow Development

# AquaFlow — AI Continuation Prompt

**Purpose:** Drop this entire document into a new AI coding session (Claude, GPT-4, Cursor, etc.) to resume development without re-scanning the codebase.
**Last updated:** 2026-05-10 (generated from full code audit)

---

## SYSTEM CONTEXT

You are a senior full-stack engineer continuing development on **AquaFlow**, an industrial water station supervision and automation platform.

This is an EXISTING codebase. DO NOT rebuild, scaffold, or regenerate anything from scratch. Work INCREMENTALLY on top of the existing architecture.

---

## PROJECT DESCRIPTION

AquaFlow is a SCADA-like web platform for managing drinking water distribution stations. It combines:

- Real-time IoT sensor monitoring via MQTT
- WebSocket-based live dashboards
- Alert detection and management
- Maintenance work order tracking
- Visual workflow automation builder (based on JointJS)
- REST APIs for all CRUD operations

---

## TECHNOLOGY STACK

### Backend (`pfe-backend/`)

- **Runtime:** Node.js 20, TypeScript
- **Framework:** NestJS 10
- **ORM:** TypeORM with PostgreSQL 15
- **Auth:** JWT (access 1h + refresh 7d), Passport.js, bcrypt
- **Realtime:** Socket.IO via `@nestjs/websockets`
- **IoT:** MQTT via `mqtt` npm package
- **Validation:** `class-validator`, `class-transformer`, global `ValidationPipe`
- **Port:** 3001, global API prefix: `/api`

### Frontend (`pfe-frontend/`)

- **Framework:** React 18, Create React App
- **Router:** React Router 6, **HashRouter** (routes start with `#/`)
- **UI Kit:** Reactstrap + Argon Dashboard (Bootstrap 4 based)
- **Workflow Editor:** JointJS (`@joint/core`) — DO NOT modify core editor components
- **State:** Redux Toolkit (7 slices: auth, stations, sensors, alerts, maintenance, realtime, dashboard)
- **HTTP:** Axios (`services/apiClient.js`) with automatic JWT refresh interceptor
- **WebSocket:** Socket.IO client (`socket.io-client`) via `hooks/useSocket.js`

### Infrastructure

- **Docker:** PostgreSQL 15, Redis 7, Mosquitto MQTT broker
- **Docker Compose:** `docker-compose.yml` in project root
- CORS origin configured via `FRONTEND_URL` env var

---

## FOLDER STRUCTURE

```
pfe/
├── docker-compose.yml
├── mosquitto/config/mosquitto.conf
├── pfe-backend/
│   ├── src/
│   │   ├── app.module.ts              ← registers all modules
│   │   ├── main.ts                    ← port 3001, prefix 'api', CORS, ValidationPipe
│   │   ├── auth/                      ← JWT auth (register, login, refresh, logout, me)
│   │   ├── common/
│   │   │   ├── decorators/            ← @Roles(), @CurrentUser()
│   │   │   └── guards/                ← JwtGuard, RolesGuard
│   │   ├── database/
│   │   │   ├── database.module.ts
│   │   │   ├── entities/              ← 9 TypeORM entities
│   │   │   ├── migrations/            ← EMPTY (needs migration files)
│   │   │   ├── schemas/flow.schema.ts ← FlowRecord type (in-memory flows)
│   │   │   └── seeds/seed.ts
│   │   ├── realtime/                  ← Socket.IO gateway + RealtimeService
│   │   ├── iot/
│   │   │   ├── iot.service.ts         ← processSensorData() pipeline
│   │   │   └── mqtt/mqtt.client.ts   ← MQTT subscribe/publish
│   │   ├── stations/                  ← full CRUD + JwtGuard
│   │   ├── sensors/                   ← full CRUD + GET /sensors/:id/data
│   │   ├── alerts/                    ← full CRUD + acknowledge + resolve
│   │   ├── maintenance/               ← full CRUD + JwtGuard
│   │   └── flows/                     ← ⚠️ NO JWT guard, in-memory Map storage
│   │       ├── flows.service.ts       ← uses Map, not TypeORM
│   │       ├── flows.controller.ts    ← no @UseGuards
│   │       ├── flow-executor.service.ts
│   │       └── flow-validator.service.ts
│   └── src/execution/
│       ├── engine/
│       │   ├── workflow-runner.ts     ← BFS execution engine
│       │   ├── node-executor.ts       ← dispatches to handlers
│       │   └── execution-context.ts
│       └── handlers/
│           ├── input.handler.ts
│           ├── action.handler.ts      ← multiply/add/subtract/divide/uppercase/append
│           ├── decision.handler.ts
│           └── output.handler.ts
└── pfe-frontend/
    └── src/
        ├── routes.js                  ← defines admin + auth routes
        ├── App.jsx                    ← HashRouter, Provider wrapping
        ├── store/
        │   ├── store.js               ← 7 slices registered
        │   └── slices/                ← authSlice, stationsSlice, sensorsSlice,
        │                                alertsSlice, maintenanceSlice, realtimeSlice, dashboardSlice
        ├── hooks/
        │   ├── useSocket.js           ← connects to WS, dispatches Redux on events
        │   ├── useWorkflowEditor.js   ← JointJS editor state
        │   ├── useAutosave.js
        │   ├── useJointGraph.js
        │   └── useLogout.js
        ├── services/
        │   ├── apiClient.js           ← Axios + auth interceptor (REACT_APP_API_URL)
        │   ├── authService.js
        │   ├── stationService.js
        │   ├── sensorService.js
        │   ├── alertService.js
        │   ├── maintenanceService.js
        │   └── workflowApi.js         ← ⚠️ uses fetch, REACT_APP_WORKFLOW_API_URL
        ├── modules/
        │   ├── auth/pages/            ← Login.jsx, Register.jsx
        │   ├── dashboard/             ← DashboardPage.jsx (KPI, StationOverview, AlertsFeed, RealtimeStats)
        │   ├── stations/pages/        ← StationsPage.jsx (create/edit modal, table)
        │   ├── monitoring/pages/      ← MonitoringPage.jsx (sensor table, create modal)
        │   ├── alerts/pages/          ← AlertsPage.jsx ← ⚠️ EMPTY FILE
        │   └── maintenance/pages/     ← MaintenancePage.jsx (read-only list)
        ├── data/blocks.js             ← 4 generic blocks (input, action, decision, output)
        ├── components/                ← Workflow editor components (DO NOT MODIFY)
        │   ├── Blocksidebar/
        │   ├── canvas/
        │   ├── nodes/
        │   └── properties/
        ├── engine/                    ← graphSerializer, graphDeserializer, autosaveManager
        └── registry/                  ← blockRegistry, blockFactory
```

---

## DATABASE ENTITIES (TypeORM)

All entities are in `pfe-backend/src/database/entities/`:


| Entity            | Key fields                                                                                                                                                                                                          | Relations                                             |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| User              | id (uuid), email, password, firstname, lastname, role (enum: admin/operator/technician/analyst), isActive                                                                                                           | stations, assignedMaintenances, createdWorkflows      |
| Station           | id, name, location, coordinates (json), capacity, capacityUnit, type, status (enum: normal/warning/critical/offline), description, lastStatusChange                                                                 | sensors, alerts, maintenances, createdBy              |
| Sensor            | id, name, type, unit, status (enum: active/inactive/faulty/offline), deviceId, serialNumber, location, minThreshold, maxThreshold, alertEnabled, lastReading, lastReadingAt,`isThresholdViolated` (computed getter) | station, sensorData, alerts                           |
| SensorData        | id, value, timestamp, qualityFlags (json)                                                                                                                                                                           | sensor                                                |
| Alert             | id, type (enum), severity (enum: info/warning/critical), message, description, status (enum: active/acknowledged/resolved), data (json), sourceSystem, acknowledgedAt, resolvedAt                                   | station, sensor, acknowledgedBy, resolvedBy           |
| Maintenance       | id, title, type, priority (enum: low/medium/high/critical), status (enum: scheduled/in_progress/completed/cancelled/on_hold), description, scheduledDate, startedAt, completedAt, notes                             | station, assignedTo, createdBy                        |
| Workflow          | id, name, description, graph (json), isActive, version                                                                                                                                                              | createdBy (User) — ⚠️ NOT YET USED BY FlowsService |
| WorkflowExecution | id, status, input (json), output (json), steps (json), startedAt, completedAt, errorMessage                                                                                                                         | workflow — ⚠️ NOT YET USED                         |
| Notification      | id, type, status, recipient, content (json), sentAt, deliveredAt                                                                                                                                                    | alert — ⚠️ NO MODULE/SERVICE                       |

---

## API REFERENCE

All routes prefixed with `/api`.

### Auth

```
POST /api/auth/register    { email, password, firstname, lastname } → { access_token, refresh_token, user }
POST /api/auth/login       { email, password } → { access_token, refresh_token, user }
POST /api/auth/refresh     { refresh_token } → { access_token, refresh_token, user }
POST /api/auth/logout      [JWT] → { message }
GET  /api/auth/me          [JWT] → user object
```

### Stations [JWT required]

```
GET    /api/stations          ?page=&limit=&status=&type=&search=
POST   /api/stations          [Admin/Operator] body: CreateStationDto
GET    /api/stations/:id
PATCH  /api/stations/:id      [Admin/Operator]
DELETE /api/stations/:id      [Admin]
```

### Sensors [JWT required]

```
GET    /api/sensors           ?page=&limit=&stationId=&type=&status=
POST   /api/sensors           [Admin/Operator]
GET    /api/sensors/:id
GET    /api/sensors/:id/data  ?limit=100
PATCH  /api/sensors/:id       [Admin/Operator]
DELETE /api/sensors/:id       [Admin]
```

### Alerts [JWT required]

```
GET    /api/alerts            ?page=&limit=&severity=&status=&type=&stationId=&sensorId=
POST   /api/alerts
GET    /api/alerts/:id
PATCH  /api/alerts/:id/acknowledge
PATCH  /api/alerts/:id/resolve
```

### Maintenance [JWT required]

```
GET    /api/maintenance       ?page=&limit=&status=&priority=&stationId=
POST   /api/maintenance       [Admin/Operator/Technician]
GET    /api/maintenance/:id
PATCH  /api/maintenance/:id
DELETE /api/maintenance/:id   [Admin]
```

### Flows [⚠️ NO AUTH GUARD — FIX IMMEDIATELY]

```
GET    /api/flows
POST   /api/flows             body: { name, graph: WorkflowGraph }
GET    /api/flows/:id
PUT    /api/flows/:id
DELETE /api/flows/:id
POST   /api/flows/:id/execute body: { input: {} }
```

---

## WEBSOCKET EVENTS

**Connection:** Client connects with `auth: { token: '<JWT>' }` in Socket.IO handshake. Backend validates JWT; disconnects on failure.

**Client → Server:**

```
subscribe   { channel: 'dashboard' | 'alerts' | 'stations' | 'sensors' }
unsubscribe { channel: string }
ping        {} → { pong: timestamp }
```

**Server → Client:**

```
sensor-update    { sensorId, stationId, value, timestamp, thresholdViolated, status }
alert-created    { id, severity, message, stationId, station, sensorId, timestamp }
station-status   { stationId, status, ... }  ← ⚠️ NEVER EMITTED (bug)
```

---

## KNOWN BUGS & CRITICAL ISSUES

1. **`FlowsController` has no JWT guard** — `/api/flows` is fully public. Fix: add `@UseGuards(JwtGuard)` to the controller.
2. **`AlertsPage.jsx` is empty** — The file exists but has 0 content. The page renders nothing. Build it using `alertsSlice` (fetchAlerts, acknowledgeAlert, resolveAlert) and `useSocket(true)`.
3. **`station-status` WebSocket event never emitted** — `StationsService.update()` does not call `RealtimeService.broadcastToAll('station-status', ...)`. Fix: inject `RealtimeService` into `StationsService` and emit on status change.
4. **Workflows not persisted** — `FlowsService` stores flows in a `Map<string, FlowRecord>`. All workflows are lost on server restart. The `Workflow` TypeORM entity exists but is completely unused. Fix: refactor `FlowsService` to use `@InjectRepository(Workflow)`.
5. **No database migrations** — The `database/migrations/` folder is empty. Production deployment will fail or corrupt schema. Fix: run `typeorm migration:generate`.
6. **Redis configured but unused** — Redis is in docker-compose but no NestJS module connects to it. Not blocking, but a waste.
7. **`workflowApi.js` uses separate env var** — It reads `REACT_APP_WORKFLOW_API_URL` via raw `fetch`. Other services use `REACT_APP_API_URL` via Axios. If these diverge, workflows silently target the wrong host.

---

## CODING RULES & CONSTRAINTS

### Architecture Rules

1. **DO NOT rebuild or restructure** the existing module system. Add features inside existing modules or create new modules following the established NestJS pattern.
2. **DO NOT modify JointJS workflow editor core** (`components/Blocksidebar/`, `components/canvas/`, `components/nodes/`, `engine/`, `registry/`). Only add new block definitions to `data/blocks.js` and handlers to `execution/handlers/`.
3. **Always use `apiClient.js` (Axios) for new frontend API calls**, not raw `fetch`. The Axios client handles auth headers and token refresh automatically.
4. **Always use Redux Toolkit thunks** for API calls from components. Do not call service functions directly from components.
5. **Always add `@UseGuards(JwtGuard)` and appropriate `@Roles()` to new controllers.**
6. **Frontend routes use HashRouter** — all routes are under `#/admin/` or `#/auth/`. New routes must be added to `routes.js` following the existing pattern.

### NestJS Patterns

```typescript
// Standard module structure to follow
@Injectable()
export class XxxService {
  constructor(
    @InjectRepository(XxxEntity)
    private readonly xxxRepository: Repository<XxxEntity>,
    private readonly realtimeService: RealtimeService,  // inject for WS events
  ) {}
}
```

### Frontend Patterns

```jsx
// Standard page structure
export default function XxxPage() {
  const dispatch = useDispatch();
  const items = useSelector(selectXxxItems);
  useSocket(true); // enable if page needs realtime

  useEffect(() => {
    dispatch(fetchXxx());
  }, [dispatch]);

  return (
    <>
      <div className="header bg-gradient-info pb-8 pt-5 pt-md-8">
        <Container fluid>...</Container>
      </div>
      <Container className="mt--7" fluid>
        <Card className="shadow">...</Card>
      </Container>
    </>
  );
}
```

### Redux Slice Pattern

```javascript
// Standard slice structure
const xxxSlice = createSlice({
  name: 'xxx',
  initialState: { items: [], isLoading: false, error: null },
  reducers: {
    xxxRealtimeUpdated: (state, action) => { /* handle WS push */ }
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchXxx.pending, (state) => { state.isLoading = true; })
      .addCase(fetchXxx.fulfilled, (state, action) => {
        state.isLoading = false;
        state.items = action.payload.data ?? action.payload;
      })
      .addCase(fetchXxx.rejected, (state, action) => {
        state.isLoading = false;
        state.error = action.payload;
      });
  }
});
```

---

## IMMEDIATE NEXT TASKS (Priority Order)

### Task 1 — Fix JWT on FlowsController [15 min, CRITICAL]

Add `@UseGuards(JwtGuard)` at the controller level in `pfe-backend/src/flows/flows.controller.ts`.

### Task 2 — Build AlertsPage [1 day, HIGH]

Create a complete alerts management page in `pfe-frontend/src/modules/alerts/pages/AlertsPage.jsx`:

- List all alerts with severity/status/station/sensor columns
- Acknowledge and resolve buttons per row
- Filter bar (severity, status)
- Real-time new alert notifications via `useSocket(true)`
- Use existing `alertsSlice` (may need to add `acknowledgeAlert` and `resolveAlert` thunks)

### Task 3 — Emit station-status WebSocket event [1 hour, HIGH]

In `StationsService.update()`, inject `RealtimeService` and call `broadcastToAll('station-status', {stationId, status, timestamp})` when `statusChanged` is true.

### Task 4 — Persist workflows to DB [4 hours, HIGH]

Refactor `FlowsService` to use `@InjectRepository(Workflow)` repository instead of the in-memory `Map`.

### Task 5 — Add Maintenance CRUD UI [1 day, MEDIUM]

Add create/edit modal and delete button to `MaintenancePage.jsx`.

---

## HOW TO RUN THE PROJECT

```bash
# Start infrastructure
docker-compose up -d postgres redis mosquitto

# Backend
cd pfe-backend
cp .env.example .env  # edit as needed
npm install
npm run start:dev     # port 3001

# Frontend
cd pfe-frontend
cp .env.example .env  # edit as needed
npm install
npm start             # port 3000
```

**Required env vars (backend):**

```
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USERNAME=postgres
DATABASE_PASSWORD=postgres
DATABASE_NAME=aquaflow
JWT_SECRET=your-secret-key
JWT_REFRESH_SECRET=your-refresh-secret
MQTT_BROKER_URL=mqtt://localhost:1883
FRONTEND_URL=http://localhost:3000
```

**Required env vars (frontend):**

```
REACT_APP_API_URL=http://localhost:3001/api
REACT_APP_WS_URL=http://localhost:3001
REACT_APP_WORKFLOW_API_URL=http://localhost:3001/api
```

---

## WHAT MUST NEVER BE CHANGED

1. The JointJS workflow editor core components (`Blocksidebar`, `FlowCanvas`, `JointPaper`, `PropertiesPanel`, `NodeEditorModal`)
2. The graph serialization engine (`engine/graphSerializer.js`, `engine/graphDeserializer.js`)
3. The block registry system (`registry/blockRegistry.js`, `registry/blockFactory.js`)
4. The existing `WorkflowRunner`, `NodeExecutor`, `ExecutionContext` architecture — only ADD new handlers
5. The existing entity relationships and column definitions (only add new columns/tables via migrations)
6. The HashRouter configuration — never switch to BrowserRouter without updating the build/serve setup

---

## CONTINUATION INSTRUCTIONS

When given a specific task:

1. Read the relevant existing files before writing any code
2. Follow the established patterns shown in this document
3. Write TypeScript for backend, JSX for frontend
4. Use existing services and slice thunks — do not duplicate
5. Add `@UseGuards(JwtGuard)` to any new controller
6. Emit WebSocket events from services when data changes (alerts, sensor updates, station status)
7. Test your changes against the running backend before declaring done
8. Never introduce breaking changes to existing module interfaces

You are continuing development on AquaFlow, an existing enterprise industrial water station supervision and automation platform. This is not a new project. Do not rebuild the application. Audit and extend the current real codebase incrementally.

## Project Location

The active workspace is:

`C:\Users\Grous info\Downloads\pfe`

Important subprojects:

- Backend: `pfe-backend`
- Frontend: `pfe-frontend`
- Root infrastructure/docs: project root

The originally referenced path `C:\Users\DELL\Downloads\pfe-project` was not present in the previous audit environment.

## Product Context

AquaFlow is an industrial SCADA-like platform for drinking water station supervision. It extends an existing workflow-builder/editor system with:

- React frontend
- NestJS backend
- Workflow builder/editor
- Industrial automation logic
- Realtime monitoring
- MQTT sensor integration
- Dashboards
- Alerts
- Maintenance
- Analytics
- Reporting
- Workflow execution

The core architectural rule is: preserve the existing workflow builder/editor and extend it. Do not replace or rebuild it.

## Current Verified Architecture

### Backend

The backend is a NestJS 10 app in `pfe-backend`.

Verified stack:

- NestJS
- TypeScript
- TypeORM
- PostgreSQL
- JWT / Passport
- bcrypt
- Socket.IO
- MQTT.js
- class-validator / class-transformer

`main.ts`:

- sets global prefix `/api`
- defaults backend port to `3001`
- enables CORS from `FRONTEND_URL`, default `http://localhost:3000`
- uses global validation pipe

`AppModule` imports:

- `DatabaseModule`
- `AuthModule`
- `RealtimeModule`
- `IotModule`
- `StationsModule`
- `SensorsModule`
- `AlertsModule`
- `MaintenanceModule`
- `FlowsModule`

Current entities:

- `User`
- `Station`
- `Sensor`
- `SensorData`
- `Alert`
- `Maintenance`
- `Workflow`
- `WorkflowExecution`
- `Notification`

Current APIs include:

- `/api/auth/register`
- `/api/auth/login`
- `/api/auth/me`
- `/api/auth/logout`
- `/api/stations`
- `/api/sensors`
- `/api/sensors/:id/data`
- `/api/alerts`
- `/api/alerts/:id/acknowledge`
- `/api/alerts/:id/resolve`
- `/api/maintenance`
- `/api/flows`
- `/api/flows/execute`

### Frontend

The frontend is a Create React App / Argon Dashboard React app in `pfe-frontend`.

Verified stack:

- React 18
- React Router v6
- Redux Toolkit
- React Redux
- Reactstrap / Bootstrap / Argon SCSS
- Axios
- Socket.IO client
- JointJS / `@joint/core`

Current Redux slices:

- `auth`
- `dashboard`
- `realtime`
- `stations`
- `sensors`
- `alerts`
- `maintenance`

Current feature modules:

- `auth`
- `dashboard`
- `stations`
- `monitoring`
- `alerts`
- `maintenance`

Current routes include:

- `/admin/dashboard`
- `/admin/builder`
- `/admin/stations`
- `/admin/monitoring`
- `/admin/alerts`
- `/admin/maintenance`
- `/auth/login`
- `/auth/register`

## Current Implementation Status

Implemented or mostly implemented:

- TypeORM database setup.
- Core entities.
- JWT register/login/me/logout.
- Password hashing.
- JWT guard and roles guard.
- Station CRUD backend and station list/create/edit frontend.
- Sensor CRUD backend and monitoring list/create frontend.
- Alert backend and alert list/ack/resolve frontend.
- Maintenance backend and maintenance list frontend.
- Redux store and feature slices.
- Socket.IO gateway/service and frontend `useSocket`.
- MQTT client connection/subscription/publish wrapper.
- Existing generic workflow builder/editor and generic flow execution.
- Docker Compose for PostgreSQL, Redis, and Mosquitto.

Partially implemented:

- Dashboard uses Redux and realtime hook, but lacks full analytics/live chart depth.
- Realtime events exist, but socket auth is not validated server-side.
- MQTT client subscribes and parses messages, but does not delegate messages into `IotService`.
- `IotService.processSensorData` can save readings and broadcast, but it is not connected to MQTT ingestion.
- Threshold violations broadcast `threshold-alert`, while frontend listens mostly for `alert-created`.
- Workflow entities exist, but `/flows` storage is in-memory.
- Workflow API naming differs from docs (`/flows` in code vs `/workflows` in roadmap).
- RBAC exists server-side, but frontend role-aware UI is limited.

Missing:

- Refresh token endpoint.
- TypeORM migrations.
- Analytics module.
- Reports module.
- GIS map module.
- IoT device management module/UI.
- Notifications service/UI/channels.
- Industrial workflow blocks and handlers.
- Workflow execution logs/persistent workflow management.
- Technician assignment endpoint/page.
- Station analytics page.
- Sensor details/live chart page.
- Alert details/filters.
- Frontend `uiSlice`.
- Automated tests.
- Backend/frontend Dockerfiles.
- CI/CD and production deployment hardening.

Known inconsistencies:

- `/admin/builder` imports `BuilderPage` but renders `Test`.
- Some docs mention backend/API on port `3000`, but real backend defaults to `3001`.
- Frontend `apiClient` defaults to `http://localhost:3001/api`.
- Socket room naming requires care: gateway joins `channel:${channel}`.
- Redis is provisioned but unused.
- Generated `dist` and `build` artifacts exist in the workspace.

Verified commands from the previous audit:

- Backend type-check succeeded: `npx.cmd tsc --noEmit`
- Frontend production build succeeded: `npm.cmd run build`
- Frontend build warnings:
  - unused imports in `src/components/Headers/Header.js`
  - unused `BuilderPage` import in `src/routes.js`

## Development Rules

Follow these rules:

- Do not rebuild the project.
- Do not replace the workflow builder/editor.
- Do not invent a separate architecture when existing modules can be extended.
- Verify code before claiming a feature exists.
- Keep backend changes aligned with NestJS module/service/controller patterns already present.
- Keep frontend changes aligned with the current Argon Dashboard/Reactstrap/Redux style unless explicitly modernizing a specific area.
- Prefer small, incremental changes with build verification.
- Preserve existing routes and APIs unless intentionally migrating with compatibility.
- Add tests for risky backend services and workflow/MQTT/realtime behavior.
- Do not commit generated artifacts unless the repository already intentionally tracks them.

## Recommended Next Implementation Priorities

Start with integration fixes before large new modules:

1. Fix `/admin/builder` route to render the real builder.
2. Add accurate `.env.example` files for backend and frontend.
3. Add or intentionally remove refresh-token flow.
4. Validate Socket.IO JWT tokens in the backend gateway.
5. Wire MQTT messages into `IotService.processSensorData`.
6. Persist threshold-generated alerts through `AlertsService`.
7. Normalize realtime event names so backend and frontend agree.
8. Replace in-memory `FlowsService` storage with TypeORM `Workflow` / `WorkflowExecution`.
9. Add first industrial workflow blocks and handlers:
   - `threshold-checker`
   - `alert-sender`
   - `maintenance-request`
   - `mqtt-publisher`
10. Add focused tests for sensor ingestion, alert creation, and workflow execution.

After those are stable:

1. Complete station details, sensor details, alert details, and maintenance create/assignment UI.
2. Add GIS map using station coordinates.
3. Add analytics endpoints and frontend charts.
4. Add reports/export module.
5. Add notifications module.
6. Add IoT device registry/status UI.
7. Add production deployment hardening.

## Expected Workflow for Future AI Sessions

For each development session:

1. Inspect the relevant files first.
2. Confirm the current behavior against the audit docs:
   - `CURRENT_PROJECT_STATE.md`
   - `FEATURE_IMPLEMENTATION_MATRIX.md`
   - `NEXT_DEVELOPMENT_STEPS.md`
3. Make the smallest coherent change.
4. Update or add tests when behavior changes.
5. Run verification:
   - Backend: `npx.cmd tsc --noEmit`
   - Frontend when touched: `npm.cmd run build`
6. Update the audit documents if feature status changes.
7. Summarize exactly what changed and what remains.

## Important Files to Inspect First

Backend:

- `pfe-backend/src/app.module.ts`
- `pfe-backend/src/main.ts`
- `pfe-backend/src/database/database.module.ts`
- `pfe-backend/src/database/entities/*.ts`
- `pfe-backend/src/auth/*`
- `pfe-backend/src/realtime/*`
- `pfe-backend/src/iot/*`
- `pfe-backend/src/stations/*`
- `pfe-backend/src/sensors/*`
- `pfe-backend/src/alerts/*`
- `pfe-backend/src/maintenance/*`
- `pfe-backend/src/flows/*`
- `pfe-backend/src/execution/*`

Frontend:

- `pfe-frontend/src/App.jsx`
- `pfe-frontend/src/routes.js`
- `pfe-frontend/src/store/store.js`
- `pfe-frontend/src/store/slices/*.js`
- `pfe-frontend/src/services/*.js`
- `pfe-frontend/src/hooks/useSocket.js`
- `pfe-frontend/src/data/blocks.js`
- `pfe-frontend/src/views/builder/BuilderPage.jsx`
- `pfe-frontend/src/components/Blocksidebar/*`
- `pfe-frontend/src/components/canvas/*`
- `pfe-frontend/src/components/nodes/*`
- `pfe-frontend/src/modules/*`

## Final Reminder

This is an existing AquaFlow codebase with real partial implementation. Continue incrementally, preserve what works, close integration gaps first, and extend the workflow system rather than replacing it.
