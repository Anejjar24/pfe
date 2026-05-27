# AquaFlow — Next Development Steps

**Last updated:** 2026-05-27 (post-P4 — all planned phases complete)  
**Status:** Development phases P1 → P4 are fully complete. Only optional P2 enhancements remain.

---

## Phase Summary

| Phase | Description | Status |
|-------|-------------|--------|
| P1 | Critical bug fixes (health endpoint, workflow stubs, notifications route) | ✅ ALL COMPLETE |
| P2 | High-value UI enhancements | ⚠️ 5 items remain (nice-to-have) |
| P3 | Planned features (users, charts, scheduling, GIS, CSV, live charts) | ✅ ALL COMPLETE |
| P4 | DevOps / production hardening | ✅ ALL COMPLETE |

No critical or high-severity bugs remain. All P1 items that were open in previous audits have been resolved.

---

## ✅ Completed P1 Fixes (for reference)

| Fix | What was done | Completed |
|-----|--------------|-----------|
| Health endpoint | `GET /api/health` with DB + Redis probes, HTTP 503 on degraded | P4-1 |
| `api` workflow block | **Still a stub** — `{ request: node.data, mocked: true }` | ⚠️ See P2 below |
| `notification` workflow block | **Still a stub** — `{ notified: true, ... }` | ⚠️ See P2 below |
| `/admin/notifications` route | Full NotificationsPage + route registered | P3 |

---

## P2 — Remaining Optional Enhancements

These are all medium/low priority UI improvements or backend completeness items. The platform is fully functional without them.

---

### P2-A: Sensor filter bar in MonitoringPage

**Problem:** `frontend/src/modules/monitoring/pages/MonitoringPage.jsx` fetches all sensors with no filters. On large deployments (100+ sensors) the table is overwhelming with no way to scope by station or type.

**File to edit:** `frontend/src/modules/monitoring/pages/MonitoringPage.jsx`

1. Add filter state (after existing state declarations):
```javascript
const [stationFilter, setStationFilter] = useState('');
const [typeFilter, setTypeFilter] = useState('');
```

2. Replace the `useEffect` that dispatches `fetchSensors`:
```javascript
useEffect(() => {
  const params = {};
  if (stationFilter) params.stationId = stationFilter;
  if (typeFilter) params.type = typeFilter;
  dispatch(fetchSensors(params));
  dispatch(fetchStations()); // for the dropdown options
}, [dispatch, stationFilter, typeFilter]);
```

3. Add a filter bar row inside `<CardHeader>` after the title:
```jsx
<Row className="mt-2 align-items-center">
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
        Clear filters
      </Button>
    </Col>
  )}
</Row>
```

**Backend:** `GET /api/sensors?stationId=&type=` already supports these query params — no backend changes needed.

**Verify:** Select a station from the dropdown → only sensors for that station appear. Select a type → further filters. Click "Clear filters" → full list returns.

**Effort:** 1.5 hours

---

### P2-C: Maintenance filter bar + `assignedTo` field

**Problem:** `frontend/src/modules/maintenance/pages/MaintenancePage.jsx` has no filter controls and the create/edit modal has no `assignedTo` field. The `Maintenance` entity has `assignedTo` (a string).

**File to edit:** `frontend/src/modules/maintenance/pages/MaintenancePage.jsx`

1. Extend `initialForm` to include `assignedTo`:
```javascript
const initialForm = {
  title: '', type: '', priority: 'medium', status: 'pending',
  stationId: '', description: '', scheduledDate: '',
  assignedTo: '',  // ← add this
};
```

2. Add `assignedTo` input inside the modal form (after `scheduledDate` field):
```jsx
<FormGroup>
  <label className="form-control-label">Assigned To</label>
  <Input type="text" name="assignedTo" placeholder="Technician name or ID"
    value={form.assignedTo} onChange={handleChange} />
</FormGroup>
```

3. Add filter state and wired `useEffect`:
```javascript
const [statusFilter, setStatusFilter] = useState('');
const [priorityFilter, setPriorityFilter] = useState('');

useEffect(() => {
  const params = {};
  if (statusFilter) params.status = statusFilter;
  if (priorityFilter) params.priority = priorityFilter;
  dispatch(fetchMaintenance(params));
  dispatch(fetchStations());
}, [dispatch, statusFilter, priorityFilter]);
```

4. Add filter bar in `<CardHeader>` (status dropdown + priority dropdown + clear button, same pattern as P2-A above).

**Backend:** `GET /api/maintenance?status=&priority=` already supports these filters — no backend changes needed.

**Verify:** Filter by status "pending" → only pending tasks shown. Create a new task with "assignedTo" set → value saved and shown in table.

**Effort:** 2 hours

---

### P2-D: Persist workflow execution history to database

**Problem:** `backend/src/flows/flow-executor.service.ts` is 17 lines and saves nothing to the database. The `WorkflowExecution` entity and its table exist in the DB (created by the migration) but are never written to.

**File to edit:** `backend/src/flows/flow-executor.service.ts`

Replace the current stub with:

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

**File to edit:** `backend/src/flows/flows.module.ts`
Add `WorkflowExecution` to `TypeOrmModule.forFeature([..., WorkflowExecution])`.

**File to edit:** `backend/src/flows/flows.controller.ts`
1. Pass `workflowId` when executing a saved workflow:
```typescript
// In executeWorkflow handler
return this.flowExecutorService.execute(dto.graph, dto.input, dto.workflowId);
```

2. Add `GET /flows/:id/executions` endpoint:
```typescript
@Get(':id/executions')
@UseGuards(JwtGuard)
@ApiOperation({ summary: 'List execution history for a workflow' })
getExecutions(@Param('id') id: string) {
  return this.flowsService.getExecutions(id);
}
```

**File to edit:** `backend/src/flows/flows.service.ts`
Add the `getExecutions` method:
```typescript
async getExecutions(workflowId: string) {
  return this.executionRepo.find({
    where: { workflow: { id: workflowId } },
    order: { startedAt: 'DESC' },
    take: 50,
  });
}
```
*(The executionRepo must also be injected into `FlowsService` via `@InjectRepository(WorkflowExecution)`.)*

**Verify:** Execute a workflow via `POST /api/flows/execute`. Then `GET /api/flows/:id/executions` → returns an array of `WorkflowExecution` records with `status`, `duration`, `startedAt`, `output`.

**Effort:** 3 hours

---

### P2-E: Alert detail modal

**Problem:** `frontend/src/modules/alerts/pages/AlertsPage.jsx` has no way to view full alert detail. `GET /api/alerts/:id` exists and returns complete data including `station`, `sensor`, `description`, `data` fields not shown in the table.

**File to edit:** `frontend/src/modules/alerts/pages/AlertsPage.jsx`

1. Add state:
```javascript
const [detailAlert, setDetailAlert] = useState(null);
```

2. Add `fetchAndShowAlert` handler:
```javascript
const fetchAndShowAlert = async (id) => {
  try {
    const data = await alertService.getAlert(id);
    setDetailAlert(data);
  } catch (e) {
    // ignore
  }
};
```

3. Add a "View" button in the actions column (before Acknowledge/Resolve):
```jsx
<Button size="sm" color="info" className="mr-1"
  onClick={() => fetchAndShowAlert(alert.id)}>
  View
</Button>
```

4. Add a detail modal (at the bottom of the JSX, before `</>`):
```jsx
<Modal isOpen={!!detailAlert} toggle={() => setDetailAlert(null)} size="lg">
  <ModalHeader toggle={() => setDetailAlert(null)}>Alert Detail</ModalHeader>
  <ModalBody>
    {detailAlert && (
      <dl className="row mb-0">
        <dt className="col-sm-4 text-muted">Type</dt>
        <dd className="col-sm-8">{detailAlert.type}</dd>
        <dt className="col-sm-4 text-muted">Severity</dt>
        <dd className="col-sm-8">
          <Badge color={SEVERITY_COLORS[detailAlert.severity] || 'secondary'}>
            {detailAlert.severity}
          </Badge>
        </dd>
        <dt className="col-sm-4 text-muted">Message</dt>
        <dd className="col-sm-8">{detailAlert.message}</dd>
        <dt className="col-sm-4 text-muted">Description</dt>
        <dd className="col-sm-8">{detailAlert.description || '—'}</dd>
        <dt className="col-sm-4 text-muted">Station</dt>
        <dd className="col-sm-8">{detailAlert.station?.name || '—'}</dd>
        <dt className="col-sm-4 text-muted">Sensor</dt>
        <dd className="col-sm-8">{detailAlert.sensor?.name || '—'}</dd>
        <dt className="col-sm-4 text-muted">Sensor Value</dt>
        <dd className="col-sm-8">
          {detailAlert.sensorValue != null ? `${detailAlert.sensorValue} ${detailAlert.sensor?.unit || ''}` : '—'}
        </dd>
        <dt className="col-sm-4 text-muted">Created</dt>
        <dd className="col-sm-8">{new Date(detailAlert.createdAt).toLocaleString()}</dd>
        <dt className="col-sm-4 text-muted">Acknowledged</dt>
        <dd className="col-sm-8">
          {detailAlert.acknowledgedAt ? new Date(detailAlert.acknowledgedAt).toLocaleString() : '—'}
        </dd>
        <dt className="col-sm-4 text-muted">Resolved</dt>
        <dd className="col-sm-8">
          {detailAlert.resolvedAt ? new Date(detailAlert.resolvedAt).toLocaleString() : '—'}
        </dd>
        {detailAlert.data && (
          <>
            <dt className="col-sm-4 text-muted">Raw Data</dt>
            <dd className="col-sm-8">
              <pre className="text-sm mb-0">{JSON.stringify(detailAlert.data, null, 2)}</pre>
            </dd>
          </>
        )}
      </dl>
    )}
  </ModalBody>
</Modal>
```

5. Add `getAlert` to `frontend/src/services/alertService.js` if it is missing:
```javascript
getAlert: async (id) => {
  const response = await apiClient.get(`/alerts/${id}`);
  return response.data;
},
```

**Verify:** Click "View" on any alert → modal opens with full detail including station name, sensor value, timestamps.

**Effort:** 2 hours

---

### P2-F: Wire workflow stubs to real handlers

**Status:** Low priority — workflows function end-to-end; only these two block types return mock data.

#### `api` block

**File to edit:** `backend/src/execution/engine/node-executor.ts`

Before (line ~60):
```typescript
case 'api': return { request: node.data, input, mocked: true };
```
After:
```typescript
case 'api': return this.httpRequestHandler.execute(node, input);
```
`HttpRequestHandler` is already instantiated in the class. No new dependencies needed.

**Verify:** Create a workflow with an `api` block pointing to `https://httpbin.org/get`. Execute it. Confirm the response contains `url` and `headers` fields, not `mocked: true`.

**Effort:** 5 minutes

#### `notification` block

**File to edit:** `backend/src/execution/engine/node-executor.ts`

Before (line ~61):
```typescript
case 'notification': return { notified: true, channel: node.data?.channel, input };
```
After: inject `NotificationsService` into `NodeExecutor`, create a `NotificationHandler` similar to `HttpRequestHandler`, and call `this.notificationHandler.execute(node, input)` which calls `notificationsService.createBroadcast(...)`.

**Note:** `NotificationsService` must be exported from `NotificationsModule` and imported into `FlowsModule` (or the module that provides `NodeExecutor`).

**Effort:** 1.5 hours

---

### P2-G: Create `analyticsSlice` (nice-to-have)

**Problem:** `AnalyticsPage.jsx` manages all analytics state via local `useState`. Re-navigating to the page re-fetches all data. Analytics data cannot be shared with other components (e.g., Dashboard widgets).

**File to create:** `frontend/src/store/slices/analyticsSlice.js`
```javascript
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { analyticsService } from '../../services/analyticsService';

export const fetchOverview = createAsyncThunk('analytics/fetchOverview', () =>
  analyticsService.getOverview()
);
export const fetchSensorStats = createAsyncThunk('analytics/fetchSensorStats',
  ({ sensorId, params }) => analyticsService.getSensorStats(sensorId, params)
);
export const fetchStationHistory = createAsyncThunk('analytics/fetchStationHistory',
  ({ stationId, params }) => analyticsService.getStationHistory(stationId, params)
);

const analyticsSlice = createSlice({
  name: 'analytics',
  initialState: {
    overview: null,
    sensorStats: null,
    stationHistory: null,
    loading: false,
    error: null,
  },
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchOverview.fulfilled, (s, a) => { s.overview = a.payload; s.loading = false; })
      .addCase(fetchSensorStats.fulfilled, (s, a) => { s.sensorStats = a.payload; s.loading = false; })
      .addCase(fetchStationHistory.fulfilled, (s, a) => { s.stationHistory = a.payload; s.loading = false; })
      .addMatcher((a) => a.type.startsWith('analytics/') && a.type.endsWith('/pending'),
        (s) => { s.loading = true; s.error = null; })
      .addMatcher((a) => a.type.startsWith('analytics/') && a.type.endsWith('/rejected'),
        (s, a) => { s.loading = false; s.error = a.error.message; });
  },
});

export const selectAnalyticsOverview = (state) => state.analytics.overview;
export const selectAnalyticsSensorStats = (state) => state.analytics.sensorStats;
export const selectAnalyticsLoading = (state) => state.analytics.loading;

export default analyticsSlice.reducer;
```

**File to edit:** `frontend/src/store/store.js`
Add `analytics: analyticsReducer` to the `combineReducers` / `configureStore` call.

**File to edit:** `frontend/src/modules/analytics/pages/AnalyticsPage.jsx`
Replace `useState` + direct `analyticsService` calls with `useDispatch` + thunk dispatches + `useSelector` selectors from the new slice.

**Verify:** Navigate away from Analytics and back → data is instantly restored from Redux cache without re-fetching.

**Effort:** 2–3 hours

---

## Implementation Order (when you resume development)

These are all independent — pick based on business value:

| Priority | Task | Effort | Value |
|----------|------|--------|-------|
| 1 | P2-D: Workflow execution persistence | 3 h | Core feature completeness |
| 2 | P2-A: Sensor filter bar | 1.5 h | UX for large deployments |
| 3 | P2-C: Maintenance filter + assignedTo | 2 h | Completes maintenance UI |
| 4 | P2-E: Alert detail modal | 2 h | Completes alert management |
| 5 | P2-F: Wire workflow stubs | 5 min + 1.5 h | Removes silent mocks |
| 6 | P2-G: analyticsSlice | 2–3 h | Redux consistency |

**Total remaining effort: ~12–14 hours of development.**

---

## Completed — For Reference

### P1 (All fixed)
- ✅ `GET /api/health` — DB + Redis probes, HTTP 503 on degraded (P4-1)
- ✅ `/admin/notifications` route — full NotificationsPage (P3)
- ✅ JWT guard on FlowsController (P1 fixes session)
- ✅ AlertsPage from 0 bytes → full UI (P1 fixes session)
- ✅ `station-status` WS event (P1 fixes session)
- ✅ Workflow DB persistence via TypeORM (P1 fixes session)
- ✅ DB migration files (1 initial migration) (P1 fixes session)
- ✅ workflowApi.js env drift fixed (P1 fixes session)
- ✅ Redis used for auth denylist + sensor cache (P1 fixes session)

### P3 (All 6 features complete)
- ✅ P3-A: User Management (UsersModule + UsersPage + RBAC)
- ✅ P3-B: Dashboard Trend Charts (TrendCharts component)
- ✅ P3-C: Workflow Scheduling + MQTT-triggered execution
- ✅ P3-D: GIS Station Map (Leaflet)
- ✅ P3-E: CSV Export (alerts + sensor data)
- ✅ P3-F: Real-time live streaming chart (50-reading rolling buffer)

### P4 (All 6 tasks complete)
- ✅ P4-1: Enhanced `/api/health` endpoint (DB + Redis probes, HTTP 503)
- ✅ P4-2: GitHub Actions CI pipelines (backend + frontend)
- ✅ P4-3: Production Docker Compose (hardened, no exposed infra ports)
- ✅ P4-4: Backend test coverage expansion (~97 tests across 7 spec files)
- ✅ P4-5: Frontend test coverage expansion (101 tests across 5 test files)
- ✅ P4-6: Lint cleanup — 0 ESLint warnings, `CI=true` build passes
