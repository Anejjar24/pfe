# AI Continuation Prompt for AquaFlow Development

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

