# TASK 5 COMPLETION REPORT — P3-E: CSV Export for Alerts and Sensor Data

**Date:** 2026-05-26  
**Status:** ✅ COMPLETE

---

## Summary

Two new CSV export endpoints added to the backend (alerts and sensor data). Two "Export CSV" buttons added to the frontend — one in the Alerts page toolbar, one in the Sensor Details chart header.

---

## Backend Changes

### Modified Files

| File | Change |
|------|--------|
| `backend/src/alerts/alerts.service.ts` | Added `exportCsv(params)` method — applies same filters as `findAll`, fetches up to 10 000 rows, builds CSV string |
| `backend/src/alerts/alerts.controller.ts` | Added `GET /alerts/export` endpoint with `@Header` decorators for Content-Type/Disposition |
| `backend/src/sensors/sensors.service.ts` | Added `exportDataCsv(sensorId, limit, from?, to?)` method — validates sensor, fetches up to 5 000 SensorData rows, includes unit from parent sensor |
| `backend/src/sensors/sensors.controller.ts` | Added `GET /sensors/:id/data/export` endpoint (declared before `:id/data` to avoid routing ambiguity) |

### New API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/api/alerts/export` | JWT | Download alerts CSV |
| `GET` | `/api/sensors/:id/data/export` | JWT | Download sensor readings CSV |

### Query Parameters

**`GET /alerts/export`**
| Param | Type | Description |
|-------|------|-------------|
| `status` | string | Filter by alert status |
| `severity` | string | Filter by severity |
| `type` | string | Filter by alert type |
| `stationId` | UUID | Filter by station |
| `sensorId` | UUID | Filter by sensor |
| `from` | ISO date | Lower bound for `createdAt` |
| `to` | ISO date | Upper bound for `createdAt` |

**`GET /sensors/:id/data/export`**
| Param | Type | Description |
|-------|------|-------------|
| `limit` | number | Max rows (default 5000) |
| `from` | ISO date | Lower bound for `timestamp` |
| `to` | ISO date | Upper bound for `timestamp` |

### CSV Formats

**Alerts CSV** (`alerts.csv`):
```
id,type,severity,status,message,station,sensor,createdAt,acknowledgedAt,resolvedAt,sourceSystem
```
- String fields (message, station name, sensor name, sourceSystem) are quoted and double-quotes are escaped
- Timestamps are ISO 8601

**Sensor Data CSV** (`sensor-data.csv`):
```
id,timestamp,value,unit,source,accuracy
```
- `unit` is repeated from the parent Sensor row (convenient for data processing)

---

## Frontend Changes

### Modified Files

| File | Change |
|------|--------|
| `frontend/src/services/alertService.js` | Added `exportCsv(params)` — calls `/alerts/export` with `responseType: 'blob'` |
| `frontend/src/services/sensorService.js` | Added `exportSensorDataCsv(id, params)` — calls `/sensors/:id/data/export` with `responseType: 'blob'` |
| `frontend/src/modules/alerts/pages/AlertsPage.jsx` | Imported `alertService`; added `isExporting` state + `handleExport` handler + "Export CSV" button in CardHeader |
| `frontend/src/modules/monitoring/pages/SensorDetailsPage.jsx` | Added `isExporting` state + `handleExport` handler + "Export CSV" button in chart CardHeader (disabled when no readings) |

### Download Pattern (both pages)
```javascript
const blob = await service.exportCsv(params);           // responseType: 'blob'
const url  = window.URL.createObjectURL(blob);
const a    = document.createElement('a');
a.href     = url;
a.download = 'filename.csv';
document.body.appendChild(a);
a.click();
document.body.removeChild(a);
window.URL.revokeObjectURL(url);
```
No external download library needed — works in all modern browsers.

---

## No New Dependencies

Neither backend nor frontend requires new packages for this task.

---

## Verification Steps

### 1. Start backend
```bash
cd backend && npm start
```

### 2. Test alerts export (curl)
```bash
TOKEN=$(curl -s -X POST http://localhost:3001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@aquaflow.io","password":"Admin123!"}' \
  | jq -r '.access_token')

# All alerts
curl -s "http://localhost:3001/api/alerts/export" \
  -H "Authorization: Bearer $TOKEN" \
  -o alerts.csv
head -3 alerts.csv
# → id,type,severity,status,message,...

# Filtered by severity
curl -s "http://localhost:3001/api/alerts/export?severity=critical" \
  -H "Authorization: Bearer $TOKEN" \
  -o critical-alerts.csv
```

### 3. Test sensor data export (curl)
```bash
# Get a sensor ID first
SENSOR_ID=$(curl -s "http://localhost:3001/api/sensors" \
  -H "Authorization: Bearer $TOKEN" | jq -r '.data[0].id')

curl -s "http://localhost:3001/api/sensors/$SENSOR_ID/data/export?limit=100" \
  -H "Authorization: Bearer $TOKEN" \
  -o sensor-data.csv
head -3 sensor-data.csv
# → id,timestamp,value,unit,source,accuracy
```

### 4. Frontend — Alerts page
1. Navigate to `/#/admin/alerts`
2. "Export CSV" button appears in the top-right toolbar
3. Apply a severity or status filter → click "Export CSV"
4. Browser downloads `alerts.csv` containing only the filtered rows
5. Button shows "Exporting…" while in flight, re-enables on completion

### 5. Frontend — Sensor Details page
1. Navigate to any sensor detail page (`/#/admin/monitoring` → click a sensor)
2. "Export CSV" button appears in the "Historical Readings" card header (right side, after the 50/100/200/500 selector)
3. Button is disabled when no readings are loaded
4. Click → browser downloads `sensor-<uuid>-data.csv`
5. CSV contains the same number of rows as the currently selected limit (50/100/200/500)
