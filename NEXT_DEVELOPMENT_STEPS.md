# AquaFlow — Next Development Steps

**Audit date:** 2026-05-25
**Basis:** Real codebase audit — file-by-file review of controllers, services, pages, slices
**Strategy:** Fix broken → complete partials → add value → advanced features

---

## P1 — CRITICAL BUGS (fix before anything else)

### Fix 1: Add `/api/health` endpoint

**Problem:** The backend Dockerfile healthcheck tries `GET /api/health` (→ 404) then falls back to `GET /api/auth/me` (→ 401 JSON). The container is reported as healthy via this fragile fallback. Any proper load balancer or orchestrator will reject the 401.

**File to create:** `backend/src/app.controller.ts`

```typescript
import { Controller, Get } from '@nestjs/common';
import { ApiOperation, ApiTags } from '@nestjs/swagger';

@ApiTags('health')
@Controller()
export class AppController {
  @Get('health')
  @ApiOperation({ summary: 'Liveness check — no auth required' })
  health() {
    return { status: 'ok', timestamp: new Date().toISOString() };
  }
}
```

**File to edit:** `backend/src/app.module.ts`
```typescript
// Add import at top
import { AppController } from './app.controller';

// Add controllers array to @Module decorator
@Module({
  controllers: [AppController],   // <-- add this line
  imports: [ ... ],               // existing imports unchanged
})
```

**Verify:** `curl http://localhost:3001/api/health` → `{ "status": "ok", "timestamp": "..." }`

**Effort:** 15 minutes

---

### Fix 2: Wire `api` workflow block to `HttpRequestHandler`

**Problem:** `backend/src/execution/engine/node-executor.ts` line 60:
```typescript
case 'api': return { request: node.data, input, mocked: true };
```
`HttpRequestHandler` is already instantiated at line 31 (`private readonly httpRequestHandler = new HttpRequestHandler()`).

**File to edit:** `backend/src/execution/engine/node-executor.ts`

Before (line 60):
```typescript
case 'api':         return { request: node.data, input, mocked: true };
```
After:
```typescript
case 'api':         return this.httpRequestHandler.execute(node, input);
```

**Verify:** Create a workflow with an `api` block pointing to `https://httpbin.org/get`. Execute it. Confirm the response contains `url` and `headers` fields, not `mocked: true`.

**Effort:** 5 minutes

---

### Fix 3: Wire `notification` workflow block to `NotificationsService`

**Problem:** `backend/src/execution/engine/node-executor.ts` line 61:
```typescript
case 'notification': return { notified: true, channel: node.data?.channel, input };
```

**File to create:** `backend/src/execution/handlers/notification.handler.ts`
```typescript
import { WorkflowNode } from '../../common/types/workflow.types';
import { NotificationsService } from '../../notifications/notifications.service';

export class NotificationHandler {
  constructor(private readonly notificationsService: NotificationsService) {}

  async execute(node: WorkflowNode, input: unknown) {
    const title = String(node.data?.title || 'Workflow Notification');
    const message = String(node.data?.message || JSON.stringify(input)).slice(0, 500);
    await this.notificationsService.createBroadcast({ title, message, severity: node.data?.severity || 'info' });
    return { notified: true, title, message };
  }
}
```

**File to edit:** `backend/src/execution/engine/node-executor.ts`
1. Import `NotificationHandler` and inject `NotificationsService`
2. Add `private readonly notificationHandler: NotificationHandler;` field
3. In constructor, `this.notificationHandler = new NotificationHandler(notificationsService);`
4. Change line 61: `case 'notification': return this.notificationHandler.execute(node, input);`

**File to edit:** `backend/src/notifications/notifications.service.ts`
Add a `createBroadcast({ title, message, severity })` method that saves a `Notification` and calls `realtimeService.broadcastToAll('notification-created', ...)`.

**Note:** `NotificationsService` must be exported from `NotificationsModule` and imported into `FlowsModule` (or `ExecutionModule` if you add one). Check `flows.module.ts` for the imports array.

**Effort:** 1.5 hours

---

### Fix 4: Add `/admin/notifications` route

**Problem:** `AdminNavbar.js` line 228 links to `/admin/notifications` which does not exist in `routes.js` or `Admin.js`. Clicking it causes a redirect to `/admin/dashboard`.

**Option A (quick — 30 min):** Add a simple notifications list page.

**File to create:** `frontend/src/modules/notifications/pages/NotificationsPage.jsx`
```jsx
import { useEffect } from 'react';
import { useDispatch, useSelector } from 'react-redux';
import { Container, Card, CardHeader, Table, Badge, Button } from 'reactstrap';
import { fetchNotifications, markAllNotificationsRead, selectNotifications, selectNotificationsMeta } from '../../../store/slices/notificationsSlice';

export default function NotificationsPage() {
  const dispatch = useDispatch();
  const notifications = useSelector(selectNotifications);
  const meta = useSelector(selectNotificationsMeta);

  useEffect(() => {
    dispatch(fetchNotifications({ limit: 50, page: 1 }));
  }, [dispatch]);

  return (
    <>
      <div className="header bg-gradient-primary pb-8 pt-5 pt-md-8">
        <Container fluid>
          <h1 className="text-white mb-0">Notifications</h1>
          <p className="text-white-50 mb-0">{meta.total} total</p>
        </Container>
      </div>
      <Container className="mt--7" fluid>
        <Card className="shadow">
          <CardHeader className="border-0 d-flex justify-content-between align-items-center">
            <h3 className="mb-0">All Notifications</h3>
            <Button size="sm" color="primary" onClick={() => dispatch(markAllNotificationsRead())}>
              Mark all read
            </Button>
          </CardHeader>
          <Table className="align-items-center table-flush" responsive>
            <thead className="thead-light">
              <tr><th>Title</th><th>Message</th><th>Severity</th><th>Time</th><th>Read</th></tr>
            </thead>
            <tbody>
              {notifications.map((n) => (
                <tr key={n.id} className={!n.readAt ? 'bg-lighter' : ''}>
                  <th scope="row">{n.title || '—'}</th>
                  <td>{n.message}</td>
                  <td><Badge color={n.severity === 'critical' ? 'danger' : 'info'}>{n.severity}</Badge></td>
                  <td className="text-sm text-muted">{new Date(n.createdAt).toLocaleString()}</td>
                  <td>{n.readAt ? '✓' : '—'}</td>
                </tr>
              ))}
            </tbody>
          </Table>
        </Card>
      </Container>
    </>
  );
}
```

**File to edit:** `frontend/src/routes.js`
```javascript
import NotificationsPage from 'modules/notifications/pages/NotificationsPage';
// Add to routes array:
{
  path: '/notifications',
  name: 'Notifications',
  icon: 'ni ni-bell-55 text-primary',
  component: <NotificationsPage />,
  layout: '/admin',
},
```

**Verify:** Click bell → "View all notifications" → lands on `/admin/notifications` with the full list.

**Effort:** 30–45 minutes

---

## P2 — HIGH VALUE (complete the core platform)

### P2-A: Add sensor filter bar to MonitoringPage

**Problem:** `frontend/src/modules/monitoring/pages/MonitoringPage.jsx` line 78 calls `dispatch(fetchSensors())` with no filters.

**File to edit:** `frontend/src/modules/monitoring/pages/MonitoringPage.jsx`

1. Add filter state after line 68:
```javascript
const [stationFilter, setStationFilter] = useState('');
const [typeFilter, setTypeFilter] = useState('');
```

2. Replace the `useEffect` at line 77:
```javascript
useEffect(() => {
  const params = {};
  if (stationFilter) params.stationId = stationFilter;
  if (typeFilter) params.type = typeFilter;
  dispatch(fetchSensors(params));
  dispatch(fetchStations());
}, [dispatch, stationFilter, typeFilter]);
```

3. Add a filter bar row inside `<CardHeader>` (after the `<h3>` tag, before the closing `</CardHeader>`):
```jsx
<Row className="mt-2 align-items-center gx-2">
  <Col md="4">
    <Input type="select" bsSize="sm" value={stationFilter}
      onChange={(e) => setStationFilter(e.target.value)}>
      <option value="">All Stations</option>
      {stations.map((s) => <option key={s.id} value={s.id}>{s.name}</option>)}
    </Input>
  </Col>
  <Col md="4">
    <Input type="select" bsSize="sm" value={typeFilter}
      onChange={(e) => setTypeFilter(e.target.value)}>
      <option value="">All Types</option>
      {['pressure','flow','temperature','quality','level','ph','turbidity','chlorine']
        .map((t) => <option key={t} value={t}>{t}</option>)}
    </Input>
  </Col>
  {(stationFilter || typeFilter) && (
    <Col xs="auto">
      <Button size="sm" color="link" className="p-0 text-muted"
        onClick={() => { setStationFilter(''); setTypeFilter(''); }}>
        Clear
      </Button>
    </Col>
  )}
</Row>
```

**Verify:** Select a station → only that station's sensors appear.

**Effort:** 1.5 hours

---

### P2-B: Wire station history chart into StationDetailsPage

**Problem:** `GET /api/analytics/stations/:id/history` exists and returns per-sensor time-series but `StationDetailsPage.jsx` never calls it.

**File to edit:** `frontend/src/modules/stations/pages/StationDetailsPage.jsx`

1. Add imports at top:
```javascript
import { Line } from 'react-chartjs-2';
import { analyticsService } from '../../../services/analyticsService';
```

2. Add state after line 44:
```javascript
const [history, setHistory] = useState(null);
const [historyLoading, setHistoryLoading] = useState(false);
```

3. Add a second effect after the existing `useEffect`:
```javascript
useEffect(() => {
  if (!stationId) return;
  setHistoryLoading(true);
  analyticsService.getStationHistory(stationId, { granularity: 'hour' })
    .then(setHistory)
    .catch(() => {})
    .finally(() => setHistoryLoading(false));
}, [stationId]);
```

4. Build chart data from `history.sensors` (each has a `.timeSeries` array) and render a `<Line>` chart below the alerts table.

**Effort:** 2.5 hours

---

### P2-C: Add maintenance filter bar and `assignedTo` field

**File to edit:** `frontend/src/modules/maintenance/pages/MaintenancePage.jsx`

1. Extend `initialForm` to include `assignedTo: ''`
2. Add `assignedTo` `<Input>` field in the modal body
3. Add filter state: `statusFilter = ''`, `priorityFilter = ''`
4. Replace `dispatch(fetchMaintenance())` with:
   ```javascript
   const params = {};
   if (statusFilter) params.status = statusFilter;
   if (priorityFilter) params.priority = priorityFilter;
   dispatch(fetchMaintenance(params));
   ```
5. Add filter bar inside `<CardHeader>` — status dropdown + priority dropdown + clear button

**Effort:** 2 hours

---

### P2-D: Persist workflow execution history to DB

**Problem:** `backend/src/flows/flow-executor.service.ts` is 17 lines and saves nothing to the DB. `WorkflowExecution` entity is defined and registered.

**File to edit:** `backend/src/flows/flow-executor.service.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { WorkflowExecution } from '../database/entities/WorkflowExecution.entity';
import { Workflow } from '../database/entities/Workflow.entity';
import { ExecutionResult, WorkflowGraph } from '../common/types/workflow.types';
import { WorkflowRunner } from '../execution/engine/workflow-runner';
import { FlowValidatorService } from './flow-validator.service';

@Injectable()
export class FlowExecutorService {
  constructor(
    private readonly validator: FlowValidatorService,
    private readonly runner: WorkflowRunner,
    @InjectRepository(WorkflowExecution)
    private readonly executionRepo: Repository<WorkflowExecution>,
    @InjectRepository(Workflow)
    private readonly workflowRepo: Repository<Workflow>,
  ) {}

  async execute(
    graph: WorkflowGraph,
    input: Record<string, unknown> = {},
    workflowId?: string,
  ): Promise<ExecutionResult> {
    this.validator.validate(graph);

    const execution = this.executionRepo.create({
      workflow: workflowId ? { id: workflowId } as Workflow : undefined,
      status: 'running',
      input,
      startedAt: new Date(),
    });
    await this.executionRepo.save(execution);

    const startTime = Date.now();
    try {
      const result = await this.runner.run(graph, input);
      execution.status = 'completed';
      execution.output = result as Record<string, unknown>;
      execution.duration = Date.now() - startTime;
      execution.nodeStates = result as Record<string, unknown>;

      if (workflowId) {
        await this.workflowRepo.update(workflowId, {
          executionCount: () => 'execution_count + 1',
          lastExecutedAt: new Date(),
        });
      }
      return result;
    } catch (err) {
      execution.status = 'failed';
      execution.errorMessage = err instanceof Error ? err.message : String(err);
      execution.duration = Date.now() - startTime;
      throw err;
    } finally {
      await this.executionRepo.save(execution);
    }
  }
}
```

Also update `FlowsController` to pass `workflowId` when executing a saved workflow, and add `GET /flows/:id/executions`:
```typescript
@Get(':id/executions')
@ApiOperation({ summary: 'List execution history for a workflow' })
getExecutions(@Param('id') id: string) {
  return this.flowsService.getExecutions(id);
}
```

**File to edit:** `backend/src/flows/flows.module.ts` — add `WorkflowExecution` to `TypeOrmModule.forFeature([..., WorkflowExecution])`.

**Effort:** 3 hours

---

### P2-E: Add alert detail modal

**Problem:** `AlertsPage.jsx` has no way to view full alert data. `GET /api/alerts/:id` exists.

**File to edit:** `frontend/src/modules/alerts/pages/AlertsPage.jsx`

1. Add state: `const [detailAlert, setDetailAlert] = useState(null);`
2. Add a "View" button in the actions column: `<Button size="sm" color="default" onClick={() => fetchAndShowAlert(alert.id)}>View</Button>`
3. Add `fetchAndShowAlert`:
   ```javascript
   const fetchAndShowAlert = async (id) => {
     const data = await alertService.getAlert(id);
     setDetailAlert(data);
   };
   ```
4. Add a detail Modal:
   ```jsx
   <Modal isOpen={!!detailAlert} toggle={() => setDetailAlert(null)} size="lg">
     <ModalHeader toggle={() => setDetailAlert(null)}>Alert Detail</ModalHeader>
     <ModalBody>
       {detailAlert && (
         <dl className="row">
           <dt className="col-sm-4">Type</dt><dd className="col-sm-8">{detailAlert.type}</dd>
           <dt className="col-sm-4">Severity</dt><dd className="col-sm-8"><Badge color={SEVERITY_COLORS[detailAlert.severity]}>{detailAlert.severity}</Badge></dd>
           <dt className="col-sm-4">Message</dt><dd className="col-sm-8">{detailAlert.message}</dd>
           <dt className="col-sm-4">Description</dt><dd className="col-sm-8">{detailAlert.description || '—'}</dd>
           <dt className="col-sm-4">Station</dt><dd className="col-sm-8">{detailAlert.station?.name || '—'}</dd>
           <dt className="col-sm-4">Sensor</dt><dd className="col-sm-8">{detailAlert.sensor?.name || '—'}</dd>
           <dt className="col-sm-4">Created</dt><dd className="col-sm-8">{new Date(detailAlert.createdAt).toLocaleString()}</dd>
           <dt className="col-sm-4">Acknowledged</dt><dd className="col-sm-8">{detailAlert.acknowledgedAt ? new Date(detailAlert.acknowledgedAt).toLocaleString() : '—'}</dd>
           <dt className="col-sm-4">Resolved</dt><dd className="col-sm-8">{detailAlert.resolvedAt ? new Date(detailAlert.resolvedAt).toLocaleString() : '—'}</dd>
           {detailAlert.data && <><dt className="col-sm-4">Data</dt><dd className="col-sm-8"><pre className="text-sm">{JSON.stringify(detailAlert.data, null, 2)}</pre></dd></>}
         </dl>
       )}
     </ModalBody>
   </Modal>
   ```
5. Add `alertService.getAlert` to `frontend/src/services/alertService.js` if missing:
   ```javascript
   getAlert: async (id) => {
     const response = await apiClient.get(`/alerts/${id}`);
     return response.data;
   },
   ```

**Effort:** 2 hours

---

## P3 — PLANNED FEATURES ✅ ALL COMPLETE (2026-05-26)

### P3-A: UsersModule + User Management Page ✅ COMPLETE — see TASK_1_REPORT.md

**Backend — new module `backend/src/users/`:**
- `GET /api/users` — paginated list (admin only)
- `GET /api/users/:id` — user detail (admin only)
- `PATCH /api/users/:id` — update role, isActive (admin only)
- `PATCH /api/auth/profile` — current user updates own firstname/lastname/password

**Frontend — new page `frontend/src/modules/users/pages/UsersPage.jsx`:**
- Table: email, name, role, status, joined
- Role change dropdown (admin only)
- Deactivate/reactivate button
- Add to `routes.js` under `/admin/users`

**Effort:** 2 days

---

### P3-B: Dashboard Trend Charts ✅ COMPLETE — see TASK_2_REPORT.md

**File to create:** `frontend/src/modules/dashboard/components/TrendCharts.jsx`
On mount, call `analyticsService.getOverview()` to get sensor counts, then `analyticsService.getSensorStats(firstPressureSensorId, { from: 24hAgo })` and `getSensorStats(firstFlowSensorId, ...)`. Render two `<Line>` charts side by side.

**File to edit:** `frontend/src/modules/dashboard/pages/DashboardPage.jsx`
Replace the "Operational Focus" `<Card>` placeholder with `<TrendCharts />`.

**Effort:** 1.5 days

---

### P3-C: Workflow Scheduling & MQTT-Triggered Execution ✅ COMPLETE — see TASK_3_REPORT.md

**Backend changes:**
1. `npm install @nestjs/schedule`
2. Add `ScheduleModule.forRoot()` to `AppModule`
3. Create `WorkflowSchedulerService`: on startup, query `Workflow` where `triggerType = 'scheduled'` and `isActive = true`, schedule cron jobs from `triggerConfig.cron`
4. In `IotService.handleMessage()`: after saving a reading, check if any active workflow has `triggerType = 'sensor_threshold'` matching the sensor → execute it

**Frontend changes:**
- Add a "Trigger" section in the workflow properties panel: manual / scheduled / sensor_threshold dropdown with config fields

**Effort:** 3 days

---

### P3-D: GIS Map for Stations ✅ COMPLETE — see TASK_4_REPORT.md

**Frontend changes:**
1. `npm install leaflet react-leaflet`
2. Create `frontend/src/modules/stations/components/StationsMap.jsx`
3. Render markers colored by status; click navigates to `/admin/stations/:id`
4. Add "Map / Table" toggle to `StationsPage.jsx`

**Effort:** 1 day

---

### P3-E: CSV Export for Alerts and Sensor Data ✅ COMPLETE — see TASK_5_REPORT.md

**Backend:**
1. `GET /api/alerts/export?format=csv&from=&to=` — stream CSV
2. `GET /api/sensors/:id/data/export?format=csv&from=&to=` — stream CSV

**Frontend:**
- "Export CSV" button in `AlertsPage` toolbar → `window.URL.createObjectURL(blob)`
- "Export CSV" button in `SensorDetailsPage`

**Effort:** 1.5 days

---

### P3-F: Real-time Live Chart (Streaming Sparkline) ✅ COMPLETE — see TASK_6_REPORT.md

In `SensorDetailsPage` or `MonitoringPage`, maintain a rolling 50-reading buffer in React state. Append each `sensor-update` WS event for the selected sensor. Render as a `<Line>` chart that scrolls in real time.

**Effort:** 1 day

---

## P4 — DEVOPS / PRODUCTION HARDENING

### P4-A: Backend CI/CD Pipeline

**File to create:** `backend/.github/workflows/ci.yml`
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
          POSTGRES_DB: aquaflow_test
        options: >-
          --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
        ports: ['5432:5432']
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run test
      - run: npm run test:e2e
        env:
          DATABASE_HOST: localhost
          DATABASE_PORT: 5432
          DATABASE_USER: postgres
          DATABASE_PASSWORD: test
          DATABASE_NAME: aquaflow_test
          REDIS_HOST: localhost
          JWT_SECRET: test-secret-32-chars-minimum-here
```

**Effort:** 2 hours

---

### P4-B: Secrets and SSL for Production

- Move `JWT_SECRET` and `JWT_REFRESH_SECRET` to Docker secrets or `.env` file (already supported via `${JWT_SECRET:-change-me-...}` syntax in `docker-compose.yml`)
- Add `SMTP_HOST`, `SMTP_USER`, `SMTP_PASS` env vars to enable email notifications
- Add `REACT_APP_WORKFLOW_API_URL` if the workflow API is hosted separately; otherwise confirm it falls back to `REACT_APP_API_URL`
- Set up nginx reverse proxy with SSL termination in front of the Docker stack

**Effort:** 1 day

---

### P4-C: Expand Test Coverage

**Missing backend specs:**
- `stations.service.spec.ts` — CRUD + realtime broadcast
- `sensors.service.spec.ts` — CRUD + Redis cache invalidation
- `flows.service.spec.ts` — persistence + validation
- `iot.service.spec.ts` — MQTT message + threshold alert

**Missing frontend tests:**
- `alertsSlice.test.js` — acknowledge/resolve thunks
- `AlertsPage.test.jsx` — render + filter + buttons
- `useSocket.test.js` — event dispatch
- `SensorDetailsPage.test.jsx` — chart render

**Effort:** 3 days

---

## Recommended Implementation Order

| Week | Tasks |
|------|-------|
| **1** | Fix 1 (health endpoint) + Fix 2 (api block) + Fix 3 (notification block) + Fix 4 (notifications route) |
| **1** | P2-A (sensor filter bar) + P2-C (maintenance filter + assignedTo) |
| **2** | P2-B (station history chart) + P2-E (alert detail modal) |
| **2** | P2-D (workflow execution logging) |
| **3** | ~~P3-A (UsersModule + user page) + P3-B (dashboard trend charts)~~ ✅ DONE |
| **4** | ~~P3-C (workflow scheduling) + P3-D (GIS map)~~ ✅ DONE |
| **5** | ~~P3-E (CSV export) + P3-F (live chart)~~ ✅ DONE + P4-A (backend CI) |

---

## Quick-Reference Priority Table

| Task | Priority | Effort | Impact |
|------|----------|--------|--------|
| Fix 1: `/api/health` endpoint | 🔴 Critical | 15 min | Fixes Docker healthcheck |
| Fix 2: Wire `api` block | 🔴 Critical | 5 min | Removes silent mock |
| Fix 3: Wire `notification` block | 🔴 Critical | 1.5 h | Removes silent mock |
| Fix 4: `/admin/notifications` route | 🔴 Critical | 30–45 min | Fixes dead navbar link |
| P2-A: Sensor filter bar | 🟠 High | 1.5 h | Core UX — large deployments |
| P2-B: Station history chart | 🟠 High | 2.5 h | Uses existing API |
| P2-C: Maintenance filters + assignedTo | 🟠 High | 2 h | Completes maintenance UI |
| P2-D: Workflow execution logging | 🟠 High | 3 h | Core feature completeness |
| P2-E: Alert detail modal | 🟠 High | 2 h | Completes alert management |
| P3-A: Users module | ✅ Done | 2 d | Admin capability |
| P3-B: Dashboard trend charts | ✅ Done | 1.5 d | First-screen value |
| P3-C: Workflow scheduling | ✅ Done | 3 d | Core automation use case |
| P3-D: GIS station map | ✅ Done | 1 d | Visual value |
| P3-E: CSV export | ✅ Done | 1.5 d | Ops team request |
| P3-F: Live streaming chart | ✅ Done | 1 d | Real-time UX |
| P4-A: Backend CI pipeline | 🟢 Low | 2 h | Dev process |
| P4-C: Test coverage | 🟢 Low | 3 d | Long-term quality |
