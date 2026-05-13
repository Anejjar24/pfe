# Task 7 Report — P4: Add Edit/Delete to MonitoringPage

**Status:** DONE  
**Date:** 2026-05-13  
**Corresponds to:** NEXT_DEVELOPMENT_STEPS.md → P4 (Week 2)

---

## Context — Tasks 1–6 Already Implemented

Before starting Task 7, a full code audit confirmed that Tasks 1–6 from the MD files were already present in the codebase:

| Task | Description | Status |
|------|-------------|--------|
| Fix 1 | JWT guard on FlowsController | Already done |
| Fix 2 | AlertsPage built | Already done |
| Fix 3 | `station-status` WS event emitted | Already done |
| P1 | Workflows persisted to PostgreSQL | Already done |
| P2 | TypeORM migration file generated | Already done |
| P3 | Maintenance CRUD UI | Already done |

Task 7 (P4) is therefore the **first genuinely new task** in this session.

Also noted: **StationsPage already had edit + delete + filter wiring** — so only `MonitoringPage` (sensors) needed work.

---

## What Was Changed

### 1. `frontend/src/services/sensorService.js`

Added `deleteSensor(id)` method that calls `DELETE /api/sensors/:id`.

**Diff:**
```diff
+  async deleteSensor(id) {
+    const response = await apiClient.delete(`/sensors/${id}`);
+    return response.data;
+  },
+
   async getSensorData(id, limit = 100) {
```

### 2. `frontend/src/store/slices/sensorsSlice.js`

- Added `isSaving: false` to initial state
- Added `updateSensor` async thunk (`PATCH /api/sensors/:id`)
- Added `deleteSensor` async thunk (`DELETE /api/sensors/:id`)
- Added extra reducers for both thunks (pending/fulfilled/rejected)
- Added `selectSensorsSaving` selector

**Diff (key additions):**
```diff
 const initialState = {
   items: [],
   meta: { total: 0, page: 1, limit: 20, pages: 0 },
   isLoading: false,
+  isSaving: false,
   error: null,
 };

+export const updateSensor = createAsyncThunk('sensors/updateSensor', async ({ id, payload }, { rejectWithValue }) => {
+  try {
+    return await sensorService.updateSensor(id, payload);
+  } catch (error) {
+    return rejectWithValue(error.response?.data?.message || 'Failed to update sensor');
+  }
+});
+
+export const deleteSensor = createAsyncThunk('sensors/deleteSensor', async (id, { rejectWithValue }) => {
+  try {
+    await sensorService.deleteSensor(id);
+    return id;
+  } catch (error) {
+    return rejectWithValue(error.response?.data?.message || 'Failed to delete sensor');
+  }
+});

+      .addCase(createSensor.pending, (state) => { state.isSaving = true; })
       .addCase(createSensor.fulfilled, (state, action) => {
+        state.isSaving = false;
         state.items.unshift(action.payload);
         state.meta.total += 1;
       })
+      .addCase(createSensor.rejected, (state) => { state.isSaving = false; })
+      .addCase(updateSensor.pending, (state) => { state.isSaving = true; })
+      .addCase(updateSensor.fulfilled, (state, action) => {
+        state.isSaving = false;
+        const index = state.items.findIndex((s) => s.id === action.payload.id);
+        if (index >= 0) state.items[index] = action.payload;
+      })
+      .addCase(updateSensor.rejected, (state) => { state.isSaving = false; })
+      .addCase(deleteSensor.fulfilled, (state, action) => {
+        state.items = state.items.filter((s) => s.id !== action.payload);
+        state.meta.total -= 1;
+      });

+export const selectSensorsSaving = (state) => state.sensors.isSaving;
```

### 3. `frontend/src/modules/monitoring/pages/MonitoringPage.jsx`

Full rewrite to add:
- `editingSensor` state — tracks which sensor is being edited (null = create mode)
- `deleteTarget` state — sensor staged for deletion
- `openEdit(sensor)` — prefills form with sensor data and opens modal
- Modal title changes: "Create Sensor" / "Edit Sensor"
- Submit dispatches `updateSensor` or `createSensor` based on `editingSensor`
- **Actions column** added to table with:
  - **Edit** button (admin + operator)
  - **Delete** button (admin only) → opens confirmation modal
  - "Read only" label for other roles
- **Delete confirmation modal** (same pattern as MaintenancePage — no `window.confirm()`)
- `isSaving` wired to disable buttons while request is in flight

---

## Why

The `MonitoringPage` was create-only. Operators had no way to correct sensor thresholds, update device IDs, or change status without going directly to the API. Admins had no way to remove decommissioned sensors from the UI. These are basic operational requirements for a SCADA platform.

---

## How to Verify

### Prerequisites
```bash
# Start infra
docker-compose up -d postgres redis mosquitto

# Start backend
cd backend && npm run start:dev

# Start frontend
cd frontend && npm start
```

### API-level tests (sensors endpoint requires JWT)

**1. Get a token:**
```bash
TOKEN=$(curl -s -X POST http://localhost:3001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@aquaflow.com","password":"Admin1234!"}' \
  | jq -r '.access_token')
```

**2. List sensors (200 with token, 401 without):**
```bash
# ✅ Should return 200 + sensor list
curl -s http://localhost:3001/api/sensors \
  -H "Authorization: Bearer $TOKEN" | jq '.data | length'

# ❌ Should return 401
curl -s http://localhost:3001/api/sensors | jq '.statusCode'
```

**3. Update a sensor (PATCH — 200 with admin token):**
```bash
SENSOR_ID=$(curl -s http://localhost:3001/api/sensors \
  -H "Authorization: Bearer $TOKEN" | jq -r '.data[0].id')

curl -s -X PATCH http://localhost:3001/api/sensors/$SENSOR_ID \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"maxThreshold": 9.5}' | jq '{id, name, maxThreshold}'
```

**4. Delete a sensor (DELETE — 200 with admin token, 401 without):**
```bash
# ✅ With token — should return {"deleted":true}
curl -s -X DELETE http://localhost:3001/api/sensors/$SENSOR_ID \
  -H "Authorization: Bearer $TOKEN" | jq .

# ❌ Without token — should return 401
curl -s -X DELETE http://localhost:3001/api/sensors/$SENSOR_ID | jq '.statusCode'
```

### UI verification
1. Log in as **admin** → go to `/admin/monitoring`
2. Each sensor row now shows **Edit** and **Delete** buttons
3. Click **Edit** → modal opens pre-filled with that sensor's data
4. Change a threshold → click "Update Sensor" → row updates in-place (no page reload)
5. Click **Delete** → confirmation modal appears with sensor name
6. Confirm → sensor disappears from list
7. Log in as **operator** → only **Edit** button visible (no Delete)
8. Log in as **analyst** → "Read only" label shown

---

## Expected Results

| Action | Expected |
|--------|----------|
| `PATCH /api/sensors/:id` without token | `401 Unauthorized` |
| `PATCH /api/sensors/:id` with valid JWT | `200` + updated sensor object |
| `DELETE /api/sensors/:id` without token | `401 Unauthorized` |
| `DELETE /api/sensors/:id` with valid admin JWT | `200` + `{"deleted":true,"id":"..."}` |
| UI edit modal | Opens pre-filled, saves changes optimistically |
| UI delete modal | Shows confirmation, removes row on confirm |
| Operator role | Edit visible, Delete hidden |
| Analyst role | "Read only" shown |

---

## Build Verification

```
npm run build  →  ✅ Compiled successfully (only pre-existing Header.js warnings)
```
