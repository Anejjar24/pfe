# Task 13 Report — Analytics Frontend Page (P10)

## Status: DONE

## What Changed and Why

Built a full analytics page at `/admin/analytics` with system-wide KPIs, breakdown charts, and an interactive sensor analysis section with time-series charting.

### New Files

| File | Purpose |
|------|---------|
| `frontend/src/services/analyticsService.js` | 3 API calls: getOverview, getSensorStats, getStationHistory |
| `frontend/src/modules/analytics/pages/AnalyticsPage.jsx` | Full analytics page with KPIs, doughnut charts, sensor line chart |

### Updated Files

**`frontend/src/routes.js`** — Added `AnalyticsPage` at `/admin/analytics`:
```js
{
  path: "/analytics",
  name: "Analytics",
  icon: "ni ni-chart-pie-35 text-primary",
  component: <AnalyticsPage />,
  layout: "/admin",
}
```

### Page Structure

#### Section 1 — Header KPI Cards (loaded on mount)
- Total Stations
- Active Sensors
- Open Alerts (red icon if > 0)
- Maintenance Pending

#### Section 2 — Breakdown Charts
- **Stations by Status** — Doughnut chart with colour-coded segments (green/orange/red/grey) + count legend
- **Active Alerts by Severity** — Doughnut chart (critical=red, warning=orange, error=orange-alt, info=cyan) + count legend
- Both charts have a refresh button to re-fetch overview

#### Section 3 — Sensor Analysis
- **Sensor dropdown** — flat list from `GET /api/sensors` (shows name, unit, station)
- **Time range presets** — 24 h / 7 d / 30 d buttons
- **Custom range** — datetime-local from/to pickers (shown on "Custom" click)
- **5 stat cards** — Avg, Min, Max, Readings count, Std Dev (coloured gradient cards)
- **Line chart** — avg (solid blue fill), min (green dashed), max (red dashed) using Chart.js v2 `<Line>`
- Points hidden when > 72 buckets (performance)
- Sensor metadata footer: type, status badge, station name, thresholds

## Build Verification

```
cd frontend && npx react-scripts build
# exit code: 0 — no errors
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
# Expected: { totalStations, activeSensors, openAlerts, maintenancePending, stationsByStatus[], alertsBySeverity[] }
```

### 4. Sensor stats → 200
```bash
curl -s "http://localhost:3000/api/analytics/sensors/SENSOR_ID/stats?from=2026-05-06T00:00:00Z" \
  -H "Authorization: Bearer TOKEN" | jq '{ stats: .stats, buckets: (.timeSeries | length) }'
# Expected: { stats: { avg, min, max, count, stddev }, buckets: N }
```

### 5. UI smoke test
1. Open http://localhost:3000/#/admin/analytics
2. KPI cards load immediately
3. Doughnut charts render with station/alert breakdowns
4. Select any sensor from dropdown → stats cards and line chart appear
5. Switch between 24h / 7d / 30d → chart re-fetches automatically
6. Click "Custom" → date pickers appear → adjust range → chart updates

## Diff Summary

```
frontend/src/services/analyticsService.js                   [NEW]
frontend/src/modules/analytics/pages/AnalyticsPage.jsx      [NEW]
frontend/src/routes.js                                       [UPDATED — added /analytics route]
```
