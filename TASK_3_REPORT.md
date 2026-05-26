# TASK 3 COMPLETION REPORT — P3-C: Workflow Scheduling & MQTT-Triggered Execution

**Date:** 2026-05-25  
**Status:** ✅ COMPLETE

---

## Summary

Workflows can now run automatically via two trigger modes:
1. **Scheduled** — cron expression, evaluated by NestJS's `ScheduleModule`
2. **Sensor Threshold** — fires when an MQTT sensor reading crosses a configured threshold

A "Settings" button in the Automation Builder opens a modal where users configure the trigger without touching the JointJS canvas.

---

## ⚠️ Action Required Before Running

```bash
cd backend
npm install
```

The `@nestjs/schedule` package was added to `package.json` but not yet installed. Run `npm install` once before starting the backend.

---

## Backend Changes

### New File

| File | Description |
|------|-------------|
| `backend/src/flows/workflow-scheduler.service.ts` | Loads scheduled workflows on startup, registers cron jobs via `SchedulerRegistry`, registers MQTT threshold handler |

### Modified Files

| File | Change |
|------|--------|
| `backend/package.json` | Added `"@nestjs/schedule": "^4.1.0"` |
| `backend/src/app.module.ts` | Added `ScheduleModule.forRoot()` import |
| `backend/src/flows/dto/create-flow.dto.ts` | Added `triggerType?`, `triggerConfig?`, `isActive?` fields |
| `backend/src/flows/flows.service.ts` | `create()` + `update()` now persist trigger fields; added `activate(id)` and `deactivate(id)` |
| `backend/src/flows/flows.controller.ts` | Added `PATCH /flows/:id/activate` and `PATCH /flows/:id/deactivate`; injected `WorkflowSchedulerService` |
| `backend/src/flows/flows.module.ts` | Added `WorkflowSchedulerService` to providers; exports `FlowExecutorService` and `FlowsService` |
| `backend/src/iot/mqtt/mqtt.client.ts` | Added `registerHandler()` method + external handler dispatch in `handleMessage()` |

### New API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `PATCH` | `/flows/:id/activate` | JWT | Set `isActive = true`; reloads cron job if scheduled |
| `PATCH` | `/flows/:id/deactivate` | JWT | Set `isActive = false`; removes cron job |

### Trigger Config Shapes

**Scheduled:**
```json
{
  "triggerType": "scheduled",
  "triggerConfig": { "cron": "0 */6 * * *" },
  "isActive": true
}
```

**Sensor Threshold:**
```json
{
  "triggerType": "sensor_threshold",
  "triggerConfig": {
    "sensorId": "<uuid>",
    "condition": "above",
    "threshold": 7.5
  },
  "isActive": true
}
```

---

## Frontend Changes

### New File

| File | Description |
|------|-------------|
| `frontend/src/components/workflow/WorkflowSettingsModal.jsx` | Trigger config modal: name, trigger type, cron presets/custom, sensor selector, condition, threshold, active toggle |

### Modified Files

| File | Change |
|------|--------|
| `frontend/src/services/workflowApi.js` | `saveWorkflow(workflow, trigger)` now sends trigger settings; added `loadWorkflow(id)`, `loadWorkflows()`, `activateWorkflow(id)`, `deactivateWorkflow(id)` |
| `frontend/src/pages/BuilderPage.jsx` | Added `triggerSettings` state; Settings button + badge toolbar; opens `WorkflowSettingsModal`; trigger config saved to backend on modal save |

### WorkflowSettingsModal Features

| Feature | Detail |
|---------|--------|
| Workflow Name | Optional rename field |
| Trigger type selector | Manual / Scheduled / Sensor Threshold |
| Cron presets | Every minute, 5/15/30 min, hourly, 6h, daily — plus "Custom…" freetext |
| Sensor dropdown | Populated from `GET /sensors?limit=200`; shows name, type, station |
| Condition | Above / Below / Any (threshold field disabled when "Any") |
| Active toggle | Checkbox; triggers fire only when `isActive = true` |
| Toolbar badge | Shows current trigger mode + active status next to Settings button |

---

## Architecture Notes

### Scheduled Workflows
- `WorkflowSchedulerService.onModuleInit()` loads all `isActive=true, triggerType='scheduled'` workflows at startup and calls `SchedulerRegistry.addCronJob()`
- Each job is named `wf:<workflowId>` so it can be updated on `activate`/`deactivate`
- `reloadWorkflow(id)` is called by the controller after any activate/deactivate PATCH — it removes the old cron job and re-registers if the workflow is still active + scheduled
- Invalid cron strings are caught and logged, not thrown — the service remains healthy

### MQTT Sensor Threshold
- `MqttClient.registerHandler()` stores handlers in a private array; `handleMessage()` forwards every message to all registered handlers after processing IoT service logic
- `WorkflowSchedulerService` registers one handler on `onModuleInit` — no polling, no circular dependencies
- Handler parses `sensors/:sensorId/data` topics, queries `sensor_threshold` workflows, checks condition, and fires executor asynchronously (errors are caught and logged)
- DB query on every MQTT message is acceptable at typical IoT rates (1 reading/s); can be cached in a future optimization

### No Circular Dependencies
- `FlowsModule` already imports `IotModule` which exports `MqttClient`
- `WorkflowSchedulerService` lives in `FlowsModule` and injects `MqttClient` from `IotModule` — one-directional dependency

---

## Verification Steps

### 1. Run `npm install` in backend

```bash
cd backend && npm install
```

### 2. Scheduled workflow (manual test)

```bash
TOKEN=$(curl -s -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@aquaflow.io","password":"Admin123!"}' \
  | jq -r '.access_token')

# Create a minimal workflow
WF_ID=$(curl -s -X POST http://localhost:3000/flows \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Scheduler",
    "graph": { "id": "test-sched-01", "cells": [] },
    "triggerType": "scheduled",
    "triggerConfig": { "cron": "* * * * *" },
    "isActive": false
  }' | jq -r '.id')
echo "Created workflow: $WF_ID"

# Activate it — backend should log a cron registration within 60 s
curl -s -X PATCH "http://localhost:3000/flows/$WF_ID/activate" \
  -H "Authorization: Bearer $TOKEN" | jq '.isActive'
# → true

# Verify backend logs: "Registered cron "* * * * *" for workflow "Test Scheduler""
# After 60 s: "Running scheduled workflow: "Test Scheduler""
```

### 3. Sensor threshold workflow (MQTT test)

```bash
# Create a sensor_threshold workflow
WF_ID2=$(curl -s -X POST http://localhost:3000/flows \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Pressure Guard",
    "graph": { "id": "test-threshold-01", "cells": [] },
    "triggerType": "sensor_threshold",
    "triggerConfig": { "sensorId": "<real-sensor-uuid>", "condition": "above", "threshold": 7.0 },
    "isActive": true
  }' | jq -r '.id')

# Publish a test MQTT message above threshold
mosquitto_pub -h localhost -p 1883 \
  -t "sensors/<real-sensor-uuid>/data" \
  -m '{"value": 8.5}'
# Backend logs: "MQTT trigger: workflow "Pressure Guard" fired — sensor ... = 8.5 (above 7)"
```

### 4. Frontend — Automation Builder

1. Navigate to `/#/admin/builder`
2. A "Settings" button and a trigger badge appear above the canvas
3. Click Settings → modal opens
4. Select "Scheduled" → cron preset dropdown appears
5. Select "Every hour" → badge updates to `⏱ 0 * * * *`
6. Check "Active" → Save Settings
7. Canvas saves; backend logs show the cron job registered
8. Select "Sensor Threshold" → sensor dropdown loads from API
9. Pick a sensor, set condition "above" + threshold "7.5"
10. Save — MQTT message to that sensor's topic triggers execution
