# Task 12 Report — Analytics Backend Module (P9)

## Status: DONE

## What Changed and Why

Built a dedicated `AnalyticsModule` providing three read-only aggregation endpoints behind JWT auth. All queries run directly against existing TypeORM entities — no schema changes needed.

### New Files

| File | Purpose |
|------|---------|
| `backend/src/analytics/dto/analytics-query.dto.ts` | Validated query params: `from`, `to` (ISO dates), `granularity` (hour/day) |
| `backend/src/analytics/analytics.service.ts` | Three service methods with TypeORM QueryBuilder aggregations |
| `backend/src/analytics/analytics.controller.ts` | Three GET routes under `/api/analytics`, all behind `@UseGuards(JwtGuard)` |
| `backend/src/analytics/analytics.module.ts` | Module wiring `Station`, `Sensor`, `Alert`, `Maintenance`, `SensorData` repos |

### Updated Files

**`backend/src/app.module.ts`** — Added `AnalyticsModule` to imports.

### Endpoint Details

#### `GET /api/analytics/overview`
Returns system-wide KPIs:
```json
{
  "totalStations": 5,
  "activeSensors": 12,
  "openAlerts": 3,
  "maintenancePending": 2,
  "stationsByStatus": [
    { "status": "normal", "count": 3 },
    { "status": "warning", "count": 1 },
    { "status": "offline", "count": 1 }
  ],
  "alertsBySeverity": [
    { "severity": "critical", "count": 1 },
    { "severity": "warning", "count": 2 }
  ]
}
```

#### `GET /api/analytics/sensors/:id/stats?from=&to=`
Returns per-sensor statistics over a time window (default: last 24 hours):
```json
{
  "sensor": { "id": "...", "name": "Pressure-01", "unit": "bar", ... },
  "period": { "from": "...", "to": "..." },
  "stats": { "avg": 4.1234, "min": 3.2, "max": 5.1, "count": 288, "stddev": 0.4321 },
  "timeSeries": [
    { "time": "2026-05-13T08:00:00.000Z", "avg": 4.1, "min": 3.8, "max": 4.5 }
  ]
}
```

#### `GET /api/analytics/stations/:id/history?granularity=hour&from=&to=`
Returns all sensor readings for a station, grouped by time bucket:
```json
{
  "station": { "id": "...", "name": "Station A", "status": "normal" },
  "period": { "from": "...", "to": "...", "granularity": "hour" },
  "sensors": [
    {
      "sensorId": "...",
      "sensorName": "Flow-01",
      "unit": "L/min",
      "buckets": [
        { "time": "...", "avg": 120.5, "min": 110.0, "max": 130.0, "count": 12 }
      ]
    }
  ]
}
```

## TypeScript Verification

```
cd backend && npx tsc --noEmit
# EXIT:0 — no errors
```

## curl Verification

### 1. Get token
```bash
curl -s -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}' \
  | jq -r '.access_token'
```

### 2. Without token → 401
```bash
curl -s -o /dev/null -w "%{http_code}" \
  http://localhost:3000/api/analytics/overview
# Expected: 401
```

### 3. Overview → 200
```bash
curl -s http://localhost:3000/api/analytics/overview \
  -H "Authorization: Bearer TOKEN" | jq .
# Expected: 200 with KPI object
```

### 4. Sensor stats (last 24h) → 200
```bash
curl -s "http://localhost:3000/api/analytics/sensors/SENSOR_ID/stats" \
  -H "Authorization: Bearer TOKEN" | jq .stats
# Expected: { avg, min, max, count, stddev }
```

### 5. Sensor stats with custom range → 200
```bash
curl -s "http://localhost:3000/api/analytics/sensors/SENSOR_ID/stats?from=2026-05-01T00:00:00Z&to=2026-05-13T00:00:00Z" \
  -H "Authorization: Bearer TOKEN" | jq .stats
```

### 6. Station history hourly → 200
```bash
curl -s "http://localhost:3000/api/analytics/stations/STATION_ID/history?granularity=hour" \
  -H "Authorization: Bearer TOKEN" | jq '.sensors | length'
# Expected: number of sensors with data
```

### 7. Station history daily → 200
```bash
curl -s "http://localhost:3000/api/analytics/stations/STATION_ID/history?granularity=day" \
  -H "Authorization: Bearer TOKEN" | jq .
```

### 8. Non-existent sensor → 404
```bash
curl -s -o /dev/null -w "%{http_code}" \
  "http://localhost:3000/api/analytics/sensors/00000000-0000-0000-0000-000000000000/stats" \
  -H "Authorization: Bearer TOKEN"
# Expected: 404
```

## Diff Summary

```
backend/src/analytics/dto/analytics-query.dto.ts   [NEW]
backend/src/analytics/analytics.service.ts         [NEW]
backend/src/analytics/analytics.controller.ts      [NEW]
backend/src/analytics/analytics.module.ts          [NEW]
backend/src/app.module.ts                          [UPDATED — added AnalyticsModule]
```
