# AquaFlow — Manual QA & Testing Checklist

**Session date:** 2026-05-25  
**Scope:** All P1 + P2 changes implemented in this session  
**Stack:** Backend `http://localhost:3001` · Frontend `http://localhost:3000` · DB PostgreSQL on port 5432  
**Hash router:** all frontend routes are `http://localhost:3000/#/admin/<path>`  
**Admin credentials:** `admin@aquaflow.local` / `Admin123!`  
**Technician credentials:** `technician@aquaflow.local` / `Tech123!`  
**Analyst credentials:** `analyst@aquaflow.local` / `Analyst123!`

---

## Table of Contents

1. [/api/health endpoint](#1-apihealth-endpoint)
2. [API workflow node — real HTTP execution](#2-api-workflow-node--real-http-execution)
3. [Notification workflow node — DB persistence + WebSocket](#3-notification-workflow-node--db-persistence--websocket)
4. [/admin/notifications page](#4-adminnotifications-page)
5. [MonitoringPage — sensor filters](#5-monitoringpage--sensor-filters)
6. [StationDetailsPage — history chart](#6-stationdetailspage--history-chart)
7. [MaintenancePage — filters + assignedTo column](#7-maintenancepage--filters--assignedto-column)
8. [Workflow execution logging](#8-workflow-execution-logging)
9. [AlertsPage — detail modal](#9-alertspage--detail-modal)
10. [Full smoke test scenario](#10-full-smoke-test-scenario)
11. [Real-time WebSocket test scenario](#11-real-time-websocket-test-scenario)
12. [Docker / container health verification](#12-docker--container-health-verification)
13. [Backend log checklist](#13-backend-log-checklist)
14. [Browser console error checklist](#14-browser-console-error-checklist)

---

# 1. /api/health endpoint

## Purpose

`GET /api/health` returns `{ status: 'ok', timestamp }` with no authentication required. Docker healthcheck resolves cleanly instead of using the fragile 401-fallback.

## Preconditions

- Backend container or dev server is running on port 3001.

## Manual Test Steps

1. Open a terminal (no browser needed).
2. Run:
   ```bash
   curl -s http://localhost:3001/api/health
   ```
3. Run with verbose headers:
   ```bash
   curl -sv http://localhost:3001/api/health
   ```
4. Open browser to `http://localhost:3001/api/health` (no login required).
5. Open `http://localhost:3001/api/docs` → find the **health** tag → expand `GET /health` → click **Try it out** → **Execute**.

## Expected Success Result

- HTTP status: `200 OK`
- Response body:
  ```json
  { "status": "ok", "timestamp": "2026-05-25T10:00:00.000Z" }
  ```
- No `Authorization` header required.
- Swagger shows the endpoint under the `health` tag.
- `timestamp` value changes on every call (it is `new Date().toISOString()`).

## Expected Failure Cases

| Scenario | How to trigger | Expected result |
|----------|---------------|-----------------|
| Backend is down | Stop the backend process | `curl: (7) Failed to connect` — no response |
| Wrong path typo | `curl http://localhost:3001/api/helath` | `404 Not Found` JSON from NestJS |
| POST instead of GET | `curl -X POST http://localhost:3001/api/health` | `404 Not Found` (no POST route registered) |

## Regression Checks

- `GET /api/auth/me` without a token still returns `401` (not affected).
- `GET /api/docs` still returns the Swagger UI.
- All other existing endpoints still require the JWT bearer token.

## DevTools / Network Checks

Open Chrome DevTools → Network tab, then visit `http://localhost:3001/api/health`:
- Request method: `GET`
- Status: `200`
- Response body: `{"status":"ok","timestamp":"..."}`
- `Content-Type: application/json`
- No `Authorization` request header present or needed.

## Database Verification

None — this endpoint has no DB interaction.

---

# 2. API workflow node — real HTTP execution

## Purpose

The `api` block type in the workflow builder now executes a real HTTP request instead of returning `{ mocked: true }`.

## Preconditions

- Full stack running.
- Logged in as admin.
- A publicly reachable HTTP endpoint available for testing (use `https://httpbin.org/get` or `https://jsonplaceholder.typicode.com/todos/1`).
- At least one workflow saved in the builder, or use the ad-hoc execute endpoint.

## Manual Test Steps

### Test A — via Swagger (ad-hoc execute)

1. Open `http://localhost:3001/api/docs`.
2. Click **Authorize** → enter the Bearer token from login.
3. Find `POST /flows/execute` under the **flows** tag.
4. Click **Try it out** → paste this body:
   ```json
   {
     "graph": {
       "nodes": [
         { "id": "n1", "type": "input" },
         {
           "id": "n2",
           "type": "api",
           "data": {
             "method": "GET",
             "url": "https://jsonplaceholder.typicode.com/todos/1"
           }
         },
         { "id": "n3", "type": "output" }
       ],
       "edges": [
         { "source": "n1", "target": "n2" },
         { "source": "n2", "target": "n3" }
       ]
     },
     "input": {}
   }
   ```
5. Click **Execute**.

### Test B — via curl

```bash
# Get access token first
TOKEN=$(curl -s -X POST http://localhost:3001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@aquaflow.local","password":"Admin123!"}' \
  | python -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

curl -s -X POST http://localhost:3001/api/flows/execute \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "graph": {
      "nodes": [
        {"id":"n1","type":"input"},
        {"id":"n2","type":"api","data":{"method":"GET","url":"https://jsonplaceholder.typicode.com/todos/1"}},
        {"id":"n3","type":"output"}
      ],
      "edges": [{"source":"n1","target":"n2"},{"source":"n2","target":"n3"}]
    },
    "input": {}
  }'
```

### Test C — POST with body merging

```bash
curl -s -X POST http://localhost:3001/api/flows/execute \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "graph": {
      "nodes": [
        {"id":"n1","type":"input"},
        {"id":"n2","type":"api","data":{"method":"POST","url":"https://httpbin.org/post","body":"{\"source\":\"aquaflow\"}"}},
        {"id":"n3","type":"output"}
      ],
      "edges": [{"source":"n1","target":"n2"},{"source":"n2","target":"n3"}]
    },
    "input": {"sensorId":"abc123"}
  }'
```

## Expected Success Result

- HTTP `201` from the `/flows/execute` endpoint.
- Response `steps` array contains a step for node `n2` whose `output` looks like:
  ```json
  {
    "ok": true,
    "status": 200,
    "statusText": "OK",
    "data": { "userId": 1, "id": 1, "title": "...", "completed": false },
    "branch": "response"
  }
  ```
- **No** `"mocked": true` in the output — this confirms the stub is gone.
- For the POST test (Test C): `output.data.json` should contain both `{"source":"aquaflow"}` merged with `{"sensorId":"abc123"}`.

## Expected Failure Cases

| Scenario | How to trigger | Expected |
|----------|---------------|----------|
| URL not configured | Set `"url": ""` in node data | `output: { error: "url not configured", ok: false }` |
| Unreachable host | Set `"url": "http://10.255.255.1"` (timeout) | `output: { ok: false, error: "fetch failed", branch: "error" }` |
| HTTP 4xx from remote | Set `"url": "https://httpbin.org/status/404"` | `output: { ok: false, status: 404, branch: "error" }` |
| No auth token | Remove `Authorization` header | `401 Unauthorized` from NestJS before handler runs |
| Malformed headers JSON | `"headers": "not-json"` | Headers ignored (catch block), request still executes |

## Regression Checks

- `http-request` block type still works (it uses the same `HttpRequestHandler`).
- Other node types (`sensor-read`, `alert-trigger`, etc.) still execute correctly.

## DevTools / Network Checks

In the workflow builder, open DevTools → Network:
- `POST /api/flows/execute` request
- Request payload contains the graph with `type: "api"`
- Response `steps[1].output.ok` should be `true` for a valid URL

## Database Verification

After executing (Test B or C), query the new execution record:
```sql
SELECT id, status, input, output, duration, trigger_source, started_at
FROM workflow_executions
ORDER BY started_at DESC
LIMIT 1;
```
Expected: one row with `status = 'completed'`, `trigger_source = 'manual'`, `duration > 0`, `output` JSON matching the `todo` object.

---

# 3. Notification workflow node — DB persistence + WebSocket

## Purpose

The `notification` block in workflows now creates a real DB record and broadcasts a `notification-created` WebSocket event (for `in_app` channel), or makes a real HTTP POST (for `webhook` channel), instead of returning a stub.

## Preconditions

- Full stack running.
- Logged in as admin in **two browser tabs** (Tab A = the tester, Tab B = WebSocket observer with DevTools open).
- Access token available.

## Manual Test Steps

### Test A — in_app channel

1. In Tab B, open DevTools → Console → run:
   ```javascript
   // observe WS events
   window._wsEvents = [];
   ```
   *(The app's Socket.IO client will dispatch to Redux; you can observe via Redux DevTools instead — see below.)*

2. Open Redux DevTools extension in Tab B. Keep it visible.

3. In a terminal, run:
   ```bash
   curl -s -X POST http://localhost:3001/api/flows/execute \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $TOKEN" \
     -d '{
       "graph": {
         "nodes": [
           {"id":"n1","type":"input"},
           {"id":"n2","type":"notification","data":{"channel":"in_app","subject":"QA Test Alert","message":"This is a test notification from workflow execution"}},
           {"id":"n3","type":"output"}
         ],
         "edges": [{"source":"n1","target":"n2"},{"source":"n2","target":"n3"}]
       },
       "input": {}
     }'
   ```

4. Switch to Tab B and watch Redux DevTools.

### Test B — webhook channel (requires a webhook receiver)

Start a local listener:
```bash
# In one terminal
npx --yes http-echo-server 9999
```
Then:
```bash
curl -s -X POST http://localhost:3001/api/flows/execute \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "graph": {
      "nodes": [
        {"id":"n1","type":"input"},
        {"id":"n2","type":"notification","data":{"channel":"webhook","subject":"QA Webhook","webhookUrl":"http://localhost:9999"}},
        {"id":"n3","type":"output"}
      ],
      "edges": [{"source":"n1","target":"n2"},{"source":"n2","target":"n3"}]
    },
    "input": {"sensorReading": 42.5}
  }'
```

### Test C — unsupported channel

```bash
curl -s -X POST http://localhost:3001/api/flows/execute \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "graph": {
      "nodes": [
        {"id":"n1","type":"input"},
        {"id":"n2","type":"notification","data":{"channel":"sms","subject":"SMS test"}},
        {"id":"n3","type":"output"}
      ],
      "edges": [{"source":"n1","target":"n2"},{"source":"n2","target":"n3"}]
    },
    "input": {}
  }'
```

## Expected Success Result

**Test A — in_app:**
- Curl returns `201` with `steps[1].output`:
  ```json
  { "notified": true, "channel": "in_app", "notificationId": "<uuid>" }
  ```
- Redux DevTools in Tab B shows a `notifications/notificationReceived` action fired.
- The notification bell counter in the top navbar increments by 1.
- Navigating to `/#/admin/notifications` shows the new notification at the top of the list with subject `"QA Test Alert"`.

**Test B — webhook:**
- The local listener receives a `POST` with body:
  ```json
  { "subject": "QA Webhook", "content": "...", "data": {"sensorReading": 42.5}, "timestamp": "..." }
  ```
- Curl output: `{ "notified": true, "channel": "webhook", "status": 200, "ok": true }`

**Test C — unsupported channel:**
- Curl output: `{ "notified": false, "reason": "unsupported channel: sms" }`
- Execution still completes (`status: 'completed'` in the result).

## Expected Failure Cases

| Scenario | How to trigger | Expected |
|----------|---------------|----------|
| Webhook URL empty | `"channel":"webhook"` with no `webhookUrl` | `{ notified: false, reason: "webhookUrl not configured" }` |
| Webhook URL unreachable | `"webhookUrl":"http://localhost:9998"` (nothing listening) | `{ notified: false, channel: "webhook", error: "fetch failed" }` |
| DB down during in_app | Stop postgres container | `500` from `/flows/execute`, execution record has `status = 'failed'` |

## Regression Checks

- The existing `notifyAlertCreated` in `NotificationsService` (triggered on alert creation) is **not** affected — it uses a separate code path.
- Notifications bell still increments when alerts are created via `POST /api/alerts`.

## DevTools / Network Checks

- After Test A: `GET /api/notifications/unread-count` should return `{ count: N }` where N > previous.
- After Test A: `GET /api/notifications?page=1&limit=20` should have the new record as first item.

## Database Verification

```sql
-- Confirm in_app notification was persisted
SELECT id, type, channel, status, subject, recipient, created_at
FROM notifications
WHERE channel = 'in_app'
ORDER BY created_at DESC
LIMIT 3;
```
Expected: most recent row has `subject = 'QA Test Alert'`, `channel = 'in_app'`, `status = 'delivered'`, `recipient = 'all'`.

---

# 4. /admin/notifications page

## Purpose

The "View all notifications" link in the notification bell dropdown now navigates to a real page that lists all in-app notifications with mark-read actions, instead of bouncing to the dashboard.

## Preconditions

- Full stack running.
- At least one notification exists (run Test A from section 3, or create an alert to trigger a notification).
- Logged in as admin.

## Manual Test Steps

1. Open `http://localhost:3000/#/admin/dashboard`.
2. Click the **bell icon** in the top-right navbar.
3. In the dropdown, click **"View all notifications"**.
4. Confirm the browser navigates to `http://localhost:3000/#/admin/notifications`.
5. Verify the page renders a table with notification rows.
6. Click **"Mark read"** button on an unread (blue-highlighted) notification row.
7. Confirm the row badge changes from `Unread` → `Read` and the blue background disappears.
8. Click **"Mark all as read"** button (visible only when `unreadCount > 0`).
9. Confirm unread count in the navbar bell drops to 0.
10. Verify pagination: if more than 20 notifications exist, `Next »` button appears — click it and verify page 2 loads.

## Expected Success Result

- Route `/#/admin/notifications` renders without redirecting to dashboard.
- Page header shows: `"Notifications"` + unread count badge if `unreadCount > 0`.
- Table columns: Type · Subject · Content · Received · Status · (action button).
- Unread rows have a light blue background (`#f0f8ff`).
- After "Mark read": row badge changes to grey `Read`, `unreadCount` decreases by 1 in Redux.
- After "Mark all as read": all badges become grey, bell counter resets to 0, `"Mark all as read"` button disappears.
- Network tab shows `PATCH /api/notifications/<id>/read` → `200` for per-item, `PATCH /api/notifications/read-all` → `200` for bulk.

## Expected Failure Cases

| Scenario | How to trigger | Expected |
|----------|---------------|----------|
| No notifications exist | Fresh DB with no alerts | Table shows "No notifications found." |
| Backend offline | Stop backend | Table stays empty, no crash; spinner may show then stop |
| Auth expired | Let token expire | `401` → app redirects to login page via `apiClient` interceptor |
| Analyst role | Login as analyst@aquaflow.local | Page renders same (no role restriction on GET /notifications) |

## Regression Checks

- Notification bell badge still shows correct unread count on other pages (dashboard, alerts).
- `allNotificationsCleared` Redux action fires when "Mark all as read" is clicked (verify via Redux DevTools).
- `notificationReceived` still increments the count when a new WS event arrives while on this page.

## DevTools / Network Checks

On page load:
- `GET /api/notifications?page=1&limit=20` → `200` with `{ data: [...], meta: { total, page, limit, pages } }`

On "Mark read":
- `PATCH /api/notifications/<uuid>/read` → `200` with updated notification object

On "Mark all as read":
- `PATCH /api/notifications/read-all` → `200` with `{ updated: N }`

## Database Verification

```sql
-- Verify mark-read updates readAt
SELECT id, status, read_at
FROM notifications
WHERE recipient = 'all'
ORDER BY created_at DESC
LIMIT 5;
```
After clicking "Mark read" on a row, its `read_at` should be a recent timestamp and `status = 'read'`.

```sql
-- Verify bulk mark-all-read
SELECT COUNT(*) FROM notifications
WHERE recipient = 'all' AND read_at IS NULL;
```
Expected: `0` after "Mark all as read".

---

# 5. MonitoringPage — sensor filters

## Purpose

The Monitoring page now shows two filter dropdowns (Station + Type) that re-fetch the sensor list from the backend when changed.

## Preconditions

- Full stack running.
- Seed data loaded (15 sensors across 5 stations with various types).
- Logged in as any role.

## Manual Test Steps

1. Navigate to `http://localhost:3000/#/admin/monitoring`.
2. Confirm both dropdowns appear in the card header: `"All Stations"` and `"All Types"`.
3. Open DevTools → Network tab. Filter by `XHR`.
4. Select `"Tunis Main Station"` (or any station) from the station dropdown.
5. Observe the network request fired.
6. Count the sensor rows — only sensors belonging to that station should appear.
7. Reset station to `"All Stations"`. Then select `"pressure"` from the type dropdown.
8. Observe the network request fired.
9. Verify only pressure sensors appear.
10. Combine filters: select a station AND a type simultaneously.
11. Click `"Clear filters"` link — verify both dropdowns reset to "All" and all sensors reload.
12. Verify `"Clear filters"` link only appears when at least one filter is active.

## Expected Success Result

- Selecting a station fires `GET /api/sensors?stationId=<uuid>`.
- Selecting a type fires `GET /api/sensors?type=pressure`.
- Combining fires `GET /api/sensors?stationId=<uuid>&type=pressure`.
- Table rows update without a full page reload.
- `"Clear filters"` text link appears when at least one filter is active and disappears after clearing.
- If no sensors match the filter combination, table shows `"No sensors found."`.

## Expected Failure Cases

| Scenario | How to trigger | Expected |
|----------|---------------|----------|
| Station with no sensors | Select a station that has 0 sensors | Table shows "No sensors found." |
| Filter returns empty | Select type `"turbidity"` if no turbidity sensors exist | "No sensors found." |
| Both filters active, no match | e.g. station A + type `"ph"` with no pH sensors in station A | "No sensors found." |
| Backend offline | Stop backend, then change filter | Table shows last cached result or empty; error message visible |

## Regression Checks

- Creating a new sensor still works (modal + form unchanged).
- Editing and deleting sensors still works.
- Real-time `sensor-update` WebSocket events still update sensor `lastReading` in the table.
- After clearing filters, the full sensor list reloads correctly.
- Navigating to a sensor's detail page via the **View** button still works.

## DevTools / Network Checks

| Action | Expected request URL |
|--------|---------------------|
| Page load | `GET /api/sensors` (no params) |
| Select station | `GET /api/sensors?stationId=<uuid>` |
| Select type | `GET /api/sensors?type=pressure` |
| Both filters | `GET /api/sensors?stationId=<uuid>&type=pressure` |
| Clear filters | `GET /api/sensors` (no params) |

Response shape: `{ data: [...], meta: { total, page, limit, pages } }`

## Database Verification

```sql
-- Confirm backend returns correct counts for station filter
SELECT s.name AS station, COUNT(sen.id) AS sensor_count
FROM stations s
LEFT JOIN sensors sen ON sen.station_id = s.id
GROUP BY s.id, s.name
ORDER BY s.name;
```
The row counts in the UI should match.

```sql
-- Confirm type filter
SELECT type, COUNT(*) FROM sensors GROUP BY type ORDER BY type;
```

---

# 6. StationDetailsPage — history chart

## Purpose

The Station Details page now displays a sensor history line chart with 24h / 7d / 30d presets and a per-sensor selector, using data from `GET /api/analytics/stations/:id/history`.

## Preconditions

- Full stack running.
- At least one station exists with sensors that have historical `SensorData` records.
  - If running fresh: inject readings via `POST /api/sensors/:id/reading` or let MQTT messages arrive, OR run the seed which does not create SensorData rows (chart will show "No sensor data available").
- Logged in.

## Manual Test Steps

1. Navigate to `http://localhost:3000/#/admin/stations`.
2. Click on any station name or its detail button to open `/#/admin/stations/<uuid>`.
3. Scroll down past the KPI cards and the sensors table.
4. Locate the **"Sensor History"** card.
5. Verify the card renders with three buttons: `24 h`, `7 d`, `30 d`.
6. Click `"7 d"` — observe a network request fires.
7. Click `"30 d"` — observe a second network request fires with `granularity=day`.
8. If the station has multiple sensors, open the `"All sensors"` dropdown and select a specific sensor.
9. Verify the chart updates to show only that sensor's data.
10. Click back to `"All sensors"` — verify multiple datasets appear on the chart.

## Expected Success Result

- On load: `GET /api/analytics/stations/<uuid>/history?from=<24h-ago>&granularity=hour` fires automatically.
- Chart renders with one or more colored lines, one per sensor (up to 3 if "All sensors" selected).
- Preset buttons are styled: active preset has `color="primary"`, others `color="secondary"`.
- Legend shows sensor names and units (e.g. `"Pressure Sensor avg (bar)"`).
- Switching presets refetches data with appropriate `from` and `granularity` params.
- Sensor selector shows all sensors attached to the station.
- `"30 d"` preset sends `granularity=day` and `from` = 30 days ago.
- If no data: card shows grey text `"No sensor data available for the selected period."` instead of a broken chart.

## Expected Failure Cases

| Scenario | How to trigger | Expected |
|----------|---------------|----------|
| No SensorData rows | Fresh DB (seed doesn't insert SensorData) | Chart area shows "No sensor data available..." — no JS error |
| Invalid station ID | Navigate to `/#/admin/stations/not-a-uuid` | Error alert shown, "Back to Stations" button visible |
| Backend offline | Stop backend, then click a preset | Spinner shows then stops; chart area shows fallback text |
| Station with 0 sensors | Select a station that has no sensors attached | Sensor selector is empty; chart shows no data message |

## Regression Checks

- Station metadata card (name, type, location, capacity) still renders correctly.
- Sensors table still shows all attached sensors with last reading and status.
- Recent Alerts table still loads.
- Back to Stations button still navigates to `/#/admin/stations`.
- KPI stat cards (Status, Sensors count, Active Alerts, Capacity) still show correct values.

## DevTools / Network Checks

| Action | Expected request |
|--------|-----------------|
| Page open | `GET /api/analytics/stations/<id>/history?from=<iso>&granularity=hour` |
| Click "7 d" | `GET /api/analytics/stations/<id>/history?from=<7d-ago>&granularity=hour` |
| Click "30 d" | `GET /api/analytics/stations/<id>/history?from=<30d-ago>&granularity=day` |

Response shape:
```json
{
  "station": { "id": "...", "name": "...", "status": "..." },
  "period": { "from": "...", "to": "...", "granularity": "hour" },
  "sensors": [
    {
      "sensorId": "...",
      "sensorName": "...",
      "unit": "bar",
      "buckets": [{ "time": "...", "avg": 2.34, "min": 1.2, "max": 3.4, "count": 12 }]
    }
  ]
}
```

## Database Verification

```sql
-- Check if any SensorData exists for sensors of a given station
SELECT s.name AS sensor_name, COUNT(sd.id) AS reading_count
FROM sensors s
JOIN sensor_data sd ON sd.sensor_id = s.id
WHERE s.station_id = '<station-uuid>'
GROUP BY s.id, s.name;
```
If counts are 0, the chart will correctly show the empty state.

---

# 7. MaintenancePage — filters + assignedTo column

## Purpose

The Maintenance page now has Status and Priority filter dropdowns, and displays the assigned technician's name in a new "Assigned To" column.

## Preconditions

- Full stack running.
- Seed data loaded (4 maintenance records across various statuses and priorities).
- Logged in as admin.

## Manual Test Steps

1. Navigate to `http://localhost:3000/#/admin/maintenance`.
2. Verify the card header now contains two dropdowns: `"All Statuses"` and `"All Priorities"`.
3. Verify the table has 8 columns: Title · Station · Type · Priority · Status · **Assigned To** · Scheduled · Actions.
4. Open DevTools → Network tab.
5. Select `"In Progress"` from the Status dropdown.
6. Verify only in-progress records are shown; network request: `GET /api/maintenance?status=in_progress`.
7. Reset to `"All Statuses"`. Select `"Critical"` from Priority dropdown.
8. Verify only critical records appear; network: `GET /api/maintenance?priority=critical`.
9. Combine both filters.
10. Click `"Clear filters"` — both dropdowns reset, full list reloads.
11. Check `"Assigned To"` column: for records assigned to a user in the seed, the full name (firstname + lastname) should show. For unassigned records, `—` should appear.
12. Open the **Edit** modal for a record. Confirm the form still saves/updates without errors.

## Expected Success Result

- Status and priority dropdowns appear in the CardHeader; both have "All" as default.
- Selecting a filter fires `GET /api/maintenance?status=<value>` or `?priority=<value>`.
- Combining both fires `GET /api/maintenance?status=<s>&priority=<p>`.
- `"Clear filters"` link is visible only when at least one filter is active.
- "Assigned To" column shows `firstname + lastname` for assigned records (from `item.assignedTo.firstname` + `item.assignedTo.lastname`).
- Unassigned records show `—` (em dash).
- Spinner colSpan is correctly `8` (not the old `7`).
- "No maintenance records found." colSpan is correctly `8`.

## Expected Failure Cases

| Scenario | How to trigger | Expected |
|----------|---------------|----------|
| No records match filter | Select `"On Hold"` if no on-hold records exist | Table shows "No maintenance records found." |
| Both filters active, no match | Priority=critical + Status=completed (if no such record) | Empty state message |
| Backend offline | Stop backend then change filter | No crash; error text shown if `error` slice state is set |

## Regression Checks

- Creating a new work order via modal still works (POST /api/maintenance).
- Editing still works (PATCH /api/maintenance/:id).
- Deleting still works (DELETE /api/maintenance/:id) — admin only.
- The `scheduledDate` field still renders as a date string in the table.

## DevTools / Network Checks

| Action | Expected request |
|--------|-----------------|
| Page load | `GET /api/maintenance` |
| Select status | `GET /api/maintenance?status=in_progress` |
| Select priority | `GET /api/maintenance?priority=critical` |
| Both | `GET /api/maintenance?status=in_progress&priority=critical` |
| Clear | `GET /api/maintenance` |

## Database Verification

```sql
-- Verify assigned_to is populated for some records in seed
SELECT m.title, u.firstname, u.lastname, m.status, m.priority
FROM maintenances m
LEFT JOIN users u ON u.id = m.assigned_to
ORDER BY m.created_at DESC
LIMIT 10;
```

```sql
-- Count by status to verify filter is working end-to-end
SELECT status, COUNT(*) FROM maintenances GROUP BY status;
```

---

# 8. Workflow execution logging

## Purpose

Every call to `POST /flows/execute` now creates a `WorkflowExecution` record in the DB: RUNNING → COMPLETED (or FAILED), with execution duration, per-node steps, and input/output captured.

## Preconditions

- Full stack running.
- Access token for admin.
- PostgreSQL accessible for SQL verification.

## Manual Test Steps

### Test A — successful execution

```bash
curl -s -X POST http://localhost:3001/api/flows/execute \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "graph": {
      "nodes": [
        {"id":"n1","type":"input"},
        {"id":"n2","type":"action","data":{"label":"Multiply"}},
        {"id":"n3","type":"output"}
      ],
      "edges": [{"source":"n1","target":"n2"},{"source":"n2","target":"n3"}]
    },
    "input": {"value": 42}
  }'
```

1. Note the timestamp before the curl.
2. Run the curl above.
3. Run the DB verification query below.
4. Verify one new row with `status = 'completed'`.

### Test B — execution of a saved workflow (links workflow_id)

1. Create a workflow first:
   ```bash
   curl -s -X POST http://localhost:3001/api/flows \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $TOKEN" \
     -d '{"name":"QA Test Flow","graph":{"id":"qa-test-flow","nodes":[{"id":"n1","type":"input"},{"id":"n2","type":"output"}],"edges":[{"source":"n1","target":"n2"}]}}'
   ```
2. Note the workflow `id` from the response.
3. Execute it by sending the same graph with its id:
   ```bash
   curl -s -X POST http://localhost:3001/api/flows/execute \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $TOKEN" \
     -d '{"graph":{"id":"<workflow-id>","nodes":[{"id":"n1","type":"input"},{"id":"n2","type":"output"}],"edges":[{"source":"n1","target":"n2"}]},"input":{}}'
   ```
4. Query DB — verify `workflow_id` is set and `workflows.execution_count` incremented.

### Test C — failed execution (trigger a handler error)

```bash
# sensor-read with a non-existent sensor ID causes a DB lookup failure → execution fails
curl -s -X POST http://localhost:3001/api/flows/execute \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "graph": {
      "nodes": [
        {"id":"n1","type":"input"},
        {"id":"n2","type":"sensor-read","data":{"sensorId":"00000000-0000-0000-0000-000000000000"}},
        {"id":"n3","type":"output"}
      ],
      "edges": [{"source":"n1","target":"n2"},{"source":"n2","target":"n3"}]
    },
    "input": {}
  }'
```

## Expected Success Result

**Test A:**
- HTTP `201` returned.
- DB query shows one new row: `status = 'completed'`, `duration > 0` (ms), `trigger_source = 'manual'`, `success_count = 3` (3 nodes), `failure_count = 0`, `input = {"value": 42}`, `execution_log` is a JSON array with 3 steps.

**Test B:**
- `workflow_id` column in `workflow_executions` matches the saved workflow's UUID.
- `SELECT execution_count FROM workflows WHERE id = '<uuid>'` returns `1` (incremented from 0).
- `last_executed_at` is a recent timestamp.

**Test C:**
- HTTP `500` returned (the error propagates).
- DB row has `status = 'failed'`, `error_message` contains the error, `failure_count = 1`.
- Backend logs show the error via `this.logger.error(...)`.

## Expected Failure Cases

| Scenario | How to trigger | Expected |
|----------|---------------|----------|
| Missing `graph` in body | Send `{}` | `400 Bad Request` from ValidationPipe before executor runs — no DB record created |
| Validator rejects graph | Send graph with no nodes | `400` from `FlowValidatorService.validate()` — no DB record (validator runs before executor) |
| DB down during save | Stop postgres mid-execution | `500`; may leave orphan RUNNING record if first save succeeds but second fails |

## Regression Checks

- `FlowsService.create()`, `update()`, `findAll()`, `findOne()`, `remove()` are unaffected.
- `FlowsController` auth (JwtGuard) still required — `401` without token.
- `WorkflowRunner` still returns correct `ExecutionResult` shape.

## DevTools / Network Checks

- `POST /api/flows/execute` → `201` with `{ workflowId, status, output, steps }`.
- Response `status` field must be `'success'` for a valid graph (not `'failed'`).

## Database Verification

```sql
-- List recent executions
SELECT id, workflow_id, status, trigger_source, triggered_by,
       duration, node_execution_count, success_count, failure_count,
       error_message, started_at
FROM workflow_executions
ORDER BY started_at DESC
LIMIT 5;
```

```sql
-- Verify execution_count incremented on saved workflow
SELECT id, name, execution_count, last_executed_at
FROM workflows
ORDER BY last_executed_at DESC NULLS LAST
LIMIT 5;
```

```sql
-- Inspect execution_log (steps array)
SELECT jsonb_array_length(execution_log) AS step_count, execution_log
FROM workflow_executions
ORDER BY started_at DESC
LIMIT 1;
```

---

# 9. AlertsPage — detail modal

## Purpose

Clicking any row on the Alerts table opens a full-detail modal showing all alert fields (type, description, raw data JSON, timestamps, ack/resolve buttons).

## Preconditions

- Full stack running.
- At least one alert exists (seed creates 5 alerts).
- Logged in as admin or technician.

## Manual Test Steps

1. Navigate to `http://localhost:3000/#/admin/alerts`.
2. Verify the table loads with alert rows.
3. Click anywhere on a row (not on a button).
4. Confirm a modal opens immediately.
5. Verify the modal header shows `"[SEVERITY BADGE] Alert Details"`.
6. Check these fields are present in the modal body:
   - Message
   - Description (if set)
   - Type (human-readable, underscores replaced with spaces)
   - Severity (as a colored badge)
   - Status (as a colored badge)
   - Station name (if linked)
   - Sensor name (if linked)
   - Created timestamp
   - Acknowledged at (if acked)
   - Resolved at (if resolved)
   - Source system (if set)
   - Raw data JSON block (if `alert.data` is not empty)
7. While the modal is open, click **Acknowledge** button (if alert is active).
8. Verify the badge in the modal body immediately updates to `acknowledged`.
9. Close the modal with the `×` or **Close** button.
10. Verify the same alert row in the table now shows `acknowledged` badge.
11. Reopen the modal for an `active` alert. Click **Resolve**.
12. Click on the action buttons (Ack / Resolve) in the TABLE row directly — verify the modal does NOT open (stopPropagation working).

## Expected Success Result

- Row is clickable (cursor changes to pointer on hover).
- Modal opens with the correct alert's data.
- Fields with `null` values are hidden (not showing empty `—` labels for undefined fields).
- `Raw data` block only appears if `alert.data` is non-null and has at least one key.
- Acknowledge/Resolve in the modal dispatch the same Redux thunks as the table buttons.
- Status badge in modal updates optimistically after clicking Acknowledge/Resolve.
- Clicking the table row Ack/Resolve buttons (stopPropagation) does NOT trigger modal open.
- Modal closes cleanly with `×` or **Close**.
- After closing and reopening the same row, the updated status is reflected.

## Expected Failure Cases

| Scenario | How to trigger | Expected |
|----------|---------------|----------|
| Acknowledge an already-acked alert | Open modal for acked alert, click Acknowledge | Button is `disabled` — cannot click |
| Resolve an already-resolved alert | Open modal for resolved alert | Resolve button is `disabled` |
| Analyst role | Login as analyst | Modal opens but NO Acknowledge/Resolve buttons in modal footer (read-only) |
| Alert has no description/data | View a basic alert | Optional fields simply don't render; no empty rows |
| Backend offline | Stop backend then try ack | Redux thunk rejects; button re-enables; alert status unchanged in UI |

## Regression Checks

- Severity and Status filter dropdowns still work after modal is implemented.
- `"Clear"` filter link still works.
- Realtime: new `alert-created` WS events still add rows to the table while modal is open.
- Closing the modal doesn't reset the filter state.
- Directly clicking the **Ack** and **Resolve** buttons in the table (outside modal) still dispatches actions correctly.

## DevTools / Network Checks

When clicking Acknowledge inside modal:
- `PATCH /api/alerts/<uuid>/acknowledge` → `200` with updated alert object.

When clicking Resolve inside modal:
- `PATCH /api/alerts/<uuid>/resolve` → `200` with updated alert object.

No additional network request is made on modal open — data comes from Redux store (already loaded).

## Database Verification

```sql
-- After acknowledging via modal
SELECT id, status, acknowledged_at, acknowledged_by
FROM alerts
ORDER BY acknowledged_at DESC NULLS LAST
LIMIT 3;
```

```sql
-- After resolving via modal
SELECT id, status, resolved_at, resolved_by
FROM alerts
ORDER BY resolved_at DESC NULLS LAST
LIMIT 3;
```

---

# 10. Full Smoke Test Scenario

Run this top-to-bottom after deploying to verify the whole session's changes work together.

**Duration estimate:** ~15 minutes  
**Role:** Admin throughout

## Steps

1. **Health check**
   ```bash
   curl -sf http://localhost:3001/api/health | python -m json.tool
   ```
   Expected: `{ "status": "ok", ... }`

2. **Login**
   - Open `http://localhost:3000`.
   - Enter `admin@aquaflow.local` / `Admin123!`.
   - Click Login. Confirm redirect to `/#/admin/dashboard`.

3. **Dashboard loads**
   - KPI cards show station/sensor/alert counts.
   - No console errors (check DevTools).

4. **Monitoring with filters**
   - Navigate to `/#/admin/monitoring`.
   - Select any station from the "All Stations" dropdown.
   - Verify sensor list filters.
   - Reset to "All Stations". Select type `"pressure"`.
   - Verify only pressure sensors show.
   - Click "Clear filters" — all sensors reload.

5. **Station detail + history chart**
   - Navigate to `/#/admin/stations`.
   - Click on any station.
   - Confirm the history chart card is visible.
   - Click `"7 d"` then `"30 d"` — observe network requests.
   - If data exists, chart lines render; if not, the empty state message renders cleanly.

6. **Maintenance filters**
   - Navigate to `/#/admin/maintenance`.
   - Confirm "Assigned To" column is present.
   - Select `"Scheduled"` from Status filter.
   - Verify filtered results.
   - Click "Clear filters".

7. **Alert detail modal**
   - Navigate to `/#/admin/alerts`.
   - Click on any alert row.
   - Confirm modal opens with alert details.
   - If an active alert exists, click Acknowledge in the modal.
   - Close modal; confirm row badge updated in the table.

8. **Notifications page**
   - Click the bell icon in top navbar.
   - Click "View all notifications".
   - Confirm route changes to `/#/admin/notifications` (not the dashboard).
   - If any unread notifications, click "Mark read" on one.

9. **Workflow execution with logging**
   ```bash
   curl -s -X POST http://localhost:3001/api/flows/execute \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $TOKEN" \
     -d '{"graph":{"nodes":[{"id":"n1","type":"input"},{"id":"n2","type":"action","data":{"label":"test"}},{"id":"n3","type":"output"}],"edges":[{"source":"n1","target":"n2"},{"source":"n2","target":"n3"}]},"input":{"smoke":"test"}}'
   ```
   Check DB: `SELECT status, duration FROM workflow_executions ORDER BY started_at DESC LIMIT 1;`
   Expected: `status = 'completed'`, `duration > 0`.

10. **Workflow api block (real HTTP)**
    ```bash
    curl -s -X POST http://localhost:3001/api/flows/execute \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer $TOKEN" \
      -d '{"graph":{"nodes":[{"id":"n1","type":"input"},{"id":"n2","type":"api","data":{"method":"GET","url":"https://jsonplaceholder.typicode.com/todos/1"}},{"id":"n3","type":"output"}],"edges":[{"source":"n1","target":"n2"},{"source":"n2","target":"n3"}]},"input":{}}'
    ```
    Expected: `steps[1].output.ok === true`, no `"mocked": true` in response.

11. **Workflow notification block (in_app)**
    ```bash
    curl -s -X POST http://localhost:3001/api/flows/execute \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer $TOKEN" \
      -d '{"graph":{"nodes":[{"id":"n1","type":"input"},{"id":"n2","type":"notification","data":{"channel":"in_app","subject":"Smoke Test Notification","message":"From smoke test"}},{"id":"n3","type":"output"}],"edges":[{"source":"n1","target":"n2"},{"source":"n2","target":"n3"}]},"input":{}}'
    ```
    Expected: bell counter increments in the browser tab; `/#/admin/notifications` shows the new notification at the top.

---

# 11. Real-time WebSocket Test Scenario

## Purpose

Verify Socket.IO events still propagate correctly to the frontend after the session's changes (no regressions in realtime).

## Preconditions

- Full stack running.
- Two browser tabs open, both logged in as admin.
- Redux DevTools extension installed.

## Steps

### Alert creation → realtime broadcast

1. Open Tab A at `/#/admin/alerts`. Open Redux DevTools.
2. In a terminal:
   ```bash
   curl -s -X POST http://localhost:3001/api/alerts \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $TOKEN" \
     -d '{"type":"threshold_violation","severity":"critical","message":"WS test alert","stationId":"<any-station-id>"}'
   ```
3. In Tab A Redux DevTools, confirm `alerts/alertRealtimeReceived` action fires within 1–2 seconds.
4. Confirm the new alert row appears at the top of the table without a page refresh.
5. Confirm the bell counter in the top navbar increments (notification was also created).

### Sensor update → realtime broadcast

```bash
# Inject a sensor reading (admin/operator only)
curl -s -X POST http://localhost:3001/api/sensors/<sensor-id>/reading \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"value": 3.14, "unit": "bar", "status": "active"}'
```

In Tab A at `/#/admin/monitoring`: confirm the `Last Reading` column updates for that sensor.

### Notification created → bell counter increments

Already tested via notification workflow block in section 3. Verify the WS event fires without a page reload.

### Expected WebSocket Observability (DevTools → Network → WS tab)

1. Open Chrome DevTools → Network → filter by `WS`.
2. Find the Socket.IO connection (upgrade request to `localhost:3001`).
3. Click on it → Messages tab.
4. Trigger an alert creation (curl above).
5. Confirm an incoming message with `alert-created` event appears in the Messages tab.

---

# 12. Docker / Container Health Verification

## Steps

```bash
# Start the stack
cd pfe-project
docker-compose up -d

# Wait ~30s for healthchecks to pass, then check
docker ps --format "table {{.Names}}\t{{.Status}}"
```

Expected output — all containers `(healthy)`:
```
NAMES               STATUS
pfe-frontend        Up 30s (healthy)
pfe-backend         Up 30s (healthy)
pfe-postgres        Up 30s (healthy)
pfe-redis           Up 30s (healthy)
pfe-mosquitto       Up 30s (healthy)
```

### Verify backend healthcheck specifically

```bash
docker inspect --format='{{.State.Health.Status}}' pfe-backend
```
Expected: `healthy`

```bash
# The healthcheck now hits /api/health directly (returns 200 ok)
# Verify it no longer uses the fallback /api/auth/me path
docker inspect --format='{{json .State.Health.Log}}' pfe-backend | python -m json.tool
```
Expected: all log entries show exit code `0` and output containing `"ok"` — NOT `"Unauthorized"`.

### Previous failure (before fix)

Before this session's fix, the health log output contained `{"status":"...","message":"Unauthorized"}` (the 401 body from `/api/auth/me`). After the fix it should be `{"status":"ok","timestamp":"..."}`.

### Full restart smoke test

```bash
docker-compose down && docker-compose up -d
sleep 30
curl -s http://localhost:3001/api/health
curl -s http://localhost:3000 | head -1
```
Expected: health returns `{"status":"ok",...}`, frontend returns HTML.

---

# 13. Backend Log Checklist

Check these in `docker-compose logs backend` (or `npm run start:dev` output):

## On successful startup

- `[NestFactory] Starting Nest application...`
- `[InstanceLoader] AppModule dependencies initialized`
- `[InstanceLoader] FlowsModule dependencies initialized`
- `[RoutesResolver] AppController {/api}` — confirms health route registered
- `[RoutesResolver] FlowsController {/api/flows}` — confirms flows routes
- `[RealtimeGateway] Nest WebSocket started on port 3001`

## On workflow execution

```bash
docker-compose logs backend | grep -E "(FlowExecutor|NodeExecutor|WorkflowRunner)" | tail -20
```
Expected: no ERROR lines for successful executions. FAILED executions produce:
```
[FlowExecutorService] ERROR Workflow execution failed: <message>
```

## On notification workflow block execution

```bash
docker-compose logs backend | grep "notification" | tail -10
```
No error lines for in_app channel. Webhook connection failures show a WARN-level message.

## On realtime events

```bash
docker-compose logs backend | grep "broadcast" | tail -10
```
Confirm events like `station-status`, `alert-created`, `notification-created` are emitted.

## On health checks (Docker healthcheck daemon logs)

```bash
docker inspect pfe-backend | python -m json.tool | grep -A 20 '"Health"'
```
`Status` should be `"healthy"`. Each log entry `Output` should contain `"ok"`.

---

# 14. Browser Console Error Checklist

Open Chrome DevTools → Console tab. Check these pages and confirm **zero errors**:

| Page | URL | Things to verify |
|------|-----|-----------------|
| Dashboard | `/#/admin/dashboard` | No 401/404 errors in console |
| Monitoring | `/#/admin/monitoring` | No errors after selecting filters |
| Station detail | `/#/admin/stations/<id>` | No chart.js errors; no "Cannot read properties of undefined" |
| Alerts | `/#/admin/alerts` | No errors on row click (modal open) |
| Maintenance | `/#/admin/maintenance` | No errors after filter change |
| Notifications | `/#/admin/notifications` | No 404 on load; no "route not found" errors |

## Common errors to watch for after these changes

| Error pattern | Likely cause | Fix hint |
|--------------|-------------|----------|
| `Cannot read properties of null (reading 'sensors')` on station detail | `station` state is null before API returns | Already handled by loading spinner; if seen, check the effect cleanup |
| `Warning: Each child in a list should have a unique "key" prop` | Notification/alert list rendering | Should not appear — all lists use `n.id` or `alert.id` as key |
| `ResizeObserver loop limit exceeded` | Chart.js in a resizing container | Benign warning — not a bug |
| `Socket.io: websocket error` | Backend not running | Start the backend |
| `401 Unauthorized` on `GET /api/notifications` | Token expired | Logout and login again |
| `ChunkLoadError` | Frontend build cache mismatch | Hard refresh `Ctrl+Shift+R` |

## Network tab — requests that should NOT return 404 after this session

| Request | Previous status | Expected now |
|---------|----------------|--------------|
| `GET /api/health` | `404` | `200` |
| `GET /api/notifications` (from NotificationsPage) | Route didn't exist in frontend | `200` |
| `GET /api/analytics/stations/<id>/history` | Called but result ignored | `200`, response consumed by chart |
| `GET /api/maintenance?status=<s>` | Always called without params | `200`, filtered results |
| `GET /api/sensors?stationId=<id>` | Always called without params | `200`, filtered results |
