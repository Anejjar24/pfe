# Task 8 Report — P5: Sensor Historical Data Chart

**Status:** DONE  
**Date:** 2026-05-13  
**Corresponds to:** NEXT_DEVELOPMENT_STEPS.md → P5 (Week 2)

---

## What Was Changed

### 1. `frontend/src/services/sensorService.js`

Added `getSensorById(id)` method calling `GET /api/sensors/:id`.

**Diff:**
```diff
+  async getSensorById(id) {
+    const response = await apiClient.get(`/sensors/${id}`);
+    return response.data;
+  },
+
   async createSensor(payload) {
```

---

### 2. `frontend/src/modules/monitoring/pages/SensorDetailsPage.jsx` *(new file)*

New detail page at route `/admin/monitoring/:sensorId`.

**Features:**
- Fetches sensor metadata (`GET /api/sensors/:id`) and history (`GET /api/sensors/:id/data`) in parallel via `Promise.all`
- **Line chart** (Chart.js 2.x via `react-chartjs-2` — already installed) with:
  - Blue line: actual readings, time-ordered left → right
  - Green dashed line: min threshold (only if configured)
  - Red dashed line: max threshold (only if configured)
  - Points hidden when > 100 readings (performance)
- **Limit selector**: buttons to show last 50 / 100 / 200 / 500 readings (re-fetches on click)
- **4 KPI cards**: Current Reading, Average, Min Threshold, Max Threshold
- **Sensor metadata table**: Device ID, Serial Number, Location, Alert Enabled, Last Reading, Station
- **Back button** navigates to `/admin/monitoring`
- Handles loading, error, and empty-data states

---

### 3. `frontend/src/layouts/Admin.js`

Added import and one extra `<Route>` for the detail page (not in sidebar):

**Diff:**
```diff
+import SensorDetailsPage from "modules/monitoring/pages/SensorDetailsPage";
 ...
         <Routes>
           {getRoutes(routes)}
+          <Route path="/monitoring/:sensorId" element={<SensorDetailsPage />} />
           <Route path="*" element={<Navigate to="/admin/dashboard" replace />} />
         </Routes>
```

---

### 4. `frontend/src/modules/monitoring/pages/MonitoringPage.jsx`

Added "View" button on every sensor row that navigates to the detail page.

**Diff:**
```diff
+import { useNavigate } from 'react-router-dom';
 ...
+  const navigate = useNavigate();
 ...
-                    <td className="text-right">
-                      {canManageSensors ? ( ... Edit / Delete ... ) : 'Read only'}
-                    </td>
+                    <td className="text-right">
+                      <Button size="sm" color="default"
+                        onClick={() => navigate(`/admin/monitoring/${sensor.id}`)}>
+                        View
+                      </Button>
+                      {canManageSensors && ( ... Edit / Delete ... )}
+                    </td>
```

Note: "View" is visible to all roles (including read-only); Edit/Delete remain role-restricted.

---

## Why

The backend already exposed `GET /api/sensors/:id/data` returning time-series `SensorData` records, but there was no frontend page consuming it. Operators and technicians had no way to see sensor history, detect trends, or verify threshold violations over time — which is a core SCADA requirement.

Chart.js 2.x was already bundled (it's part of the Argon Dashboard template), so no new dependencies were needed.

---

## How to Verify

### Prerequisites
```bash
docker-compose up -d postgres redis mosquitto
cd backend && npm run start:dev
cd frontend && npm start
```

### API-level tests

**1. Get token:**
```bash
TOKEN=$(curl -s -X POST http://localhost:3001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@aquaflow.com","password":"Admin1234!"}' \
  | jq -r '.access_token')
```

**2. Get a sensor ID:**
```bash
SENSOR_ID=$(curl -s http://localhost:3001/api/sensors \
  -H "Authorization: Bearer $TOKEN" | jq -r '.data[0].id')
echo $SENSOR_ID
```

**3. Fetch sensor detail (200 with token, 401 without):**
```bash
# ✅ Should return full sensor object
curl -s http://localhost:3001/api/sensors/$SENSOR_ID \
  -H "Authorization: Bearer $TOKEN" | jq '{id, name, type, unit, minThreshold, maxThreshold}'

# ❌ Should return 401
curl -s http://localhost:3001/api/sensors/$SENSOR_ID | jq '.statusCode'
```

**4. Fetch sensor historical data (200 with token, 401 without):**
```bash
# ✅ Should return array of {id, value, timestamp, ...}
curl -s "http://localhost:3001/api/sensors/$SENSOR_ID/data?limit=50" \
  -H "Authorization: Bearer $TOKEN" | jq 'length'

# ❌ Should return 401
curl -s "http://localhost:3001/api/sensors/$SENSOR_ID/data?limit=50" | jq '.statusCode'
```

### UI verification
1. Log in → go to `/admin/monitoring`
2. Each sensor row now has a **View** button (visible to all roles)
3. Click **View** → navigates to `/admin/monitoring/<uuid>`
4. Page loads with:
   - KPI cards (Current Reading, Average, Min/Max threshold)
   - Line chart with readings ordered left → right (oldest → newest)
   - Green dashed line = min threshold (if configured)
   - Red dashed line = max threshold (if configured)
   - Limit buttons: 50 / 100 / 200 / 500
5. Click **50** → chart re-fetches and shows fewer points
6. Click **Back to Monitoring** → returns to list
7. If sensor has no readings → "No historical readings available" message shown

### MQTT simulation (generate test data)
```bash
# Publish a test sensor reading via MQTT
# Replace <SENSOR_DEVICE_ID> with the sensor's deviceId field
mosquitto_pub -h localhost -p 1883 \
  -t "sensors/<SENSOR_DEVICE_ID>/data" \
  -m '{"value": 6.7, "timestamp": "<ISO_DATE>"}'

# Then re-open the detail page — new reading should appear in chart
```

---

## Expected Results

| Action | Expected |
|--------|----------|
| `GET /api/sensors/:id` without token | `401 Unauthorized` |
| `GET /api/sensors/:id` with token | `200` + sensor object |
| `GET /api/sensors/:id/data?limit=100` without token | `401 Unauthorized` |
| `GET /api/sensors/:id/data?limit=100` with token | `200` + array of readings |
| Navigate to `/admin/monitoring/:id` | Chart page loads |
| Sensor with min/max threshold | Dashed reference lines visible on chart |
| Click limit button | Chart re-fetches with new limit |
| Sensor with no readings | Empty state message |

---

## Build Verification

```
npm run build  →  ✅ Compiled successfully
Bundle size +82 KB (chart library used — already installed, no new deps)
Only pre-existing Header.js warnings remain
```
