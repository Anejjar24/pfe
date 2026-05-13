# Task 10 Report — P7: Industrial Workflow Blocks

**Status:** DONE  
**Date:** 2026-05-13  
**Corresponds to:** NEXT_DEVELOPMENT_STEPS.md → P7 (Week 3)

---

## What Was Changed

### 1. `frontend/src/data/blocks.js`

Added **7 new block definitions** to the `workflowBlocks` array.  
(`delay` already existed in the file — not duplicated.)

---

## New Blocks Added

### Industrial category

#### `sensor-read` — Sensor Read
- **Color:** `#0ea5e9` (blue)
- **Icon:** `fa-microchip`
- **Inputs:** `trigger`
- **Outputs:** `value`, `status`
- **Properties:**
  - `label` (text)
  - `sensorId` (text) — UUID of the target sensor
- **Runtime contract:** backend handler queries `SensorRepository` by `sensorId`, returns `{ value, unit, timestamp, status }` on the `value` port and sensor status on the `status` port.

---

#### `threshold-check` — Threshold Check
- **Color:** `#f59e0b` (amber)
- **Icon:** `fa-sliders`
- **Inputs:** `value`
- **Outputs:** `pass`, `breach`
- **Properties:**
  - `label` (text)
  - `minThreshold` (number, default 0)
  - `maxThreshold` (number, default 100)
  - `mode` (select: `between` / `above_max` / `below_min`)
- **Runtime contract:** compares incoming value against thresholds; routes to `pass` if within bounds or `breach` if outside.

---

#### `pump-control` — Pump Control
- **Color:** `#6366f1` (indigo)
- **Icon:** `fa-rotate`
- **Inputs:** `trigger`
- **Outputs:** `sent`
- **Properties:**
  - `label` (text)
  - `deviceId` (text) — target device identifier
  - `command` (select: `start` / `stop` / `toggle`)
  - `topic` (text, optional override — defaults to `devices/{deviceId}/commands`)
- **Runtime contract:** publishes `{ command, deviceId, timestamp }` to the MQTT topic via `MqttClient.publish()`.

---

#### `alert-trigger` — Alert Trigger
- **Color:** `#ef4444` (red)
- **Icon:** `fa-triangle-exclamation`
- **Inputs:** `trigger`
- **Outputs:** `alert`
- **Properties:**
  - `label` (text)
  - `severity` (select: `info` / `warning` / `error` / `critical`)
  - `type` (select: `threshold_violation` / `sensor_offline` / `maintenance_due` / `system_error` / `anomaly` / `critical_event`)
  - `message` (textarea)
  - `stationId` (text, optional)
- **Runtime contract:** calls `AlertsService.create()` with block properties; alert is persisted to DB and broadcast via WebSocket (`alert-created` event).

---

#### `mqtt-publish` — MQTT Publish
- **Color:** `#8b5cf6` (violet)
- **Icon:** `fa-tower-broadcast`
- **Inputs:** `payload`
- **Outputs:** `sent`
- **Properties:**
  - `label` (text)
  - `topic` (text, default `aquaflow/commands`)
  - `qos` (select: `0` / `1` / `2`)
  - `payload` (textarea, static JSON — merged with incoming `payload` port at runtime)
- **Runtime contract:** calls `MqttClient.publish(topic, mergedPayload)`.

---

#### `station-control` — Station Control
- **Color:** `#10b981` (emerald)
- **Icon:** `fa-building`
- **Inputs:** `trigger`
- **Outputs:** `done`
- **Properties:**
  - `label` (text)
  - `stationId` (text)
  - `status` (select: `normal` / `warning` / `critical` / `offline`)
- **Runtime contract:** calls `StationsService.update(stationId, { status })`, which persists to DB and emits `station-status` WebSocket event automatically.

---

### Integration category

#### `http-request` — HTTP Request
- **Color:** `#f97316` (orange)
- **Icon:** `fa-globe`
- **Inputs:** `body`
- **Outputs:** `response`, `error`
- **Properties:**
  - `label` (text)
  - `method` (select: `GET` / `POST` / `PUT` / `PATCH` / `DELETE`)
  - `url` (text)
  - `headers` (textarea, JSON)
  - `body` (textarea, static JSON — merged with incoming `body` port at runtime)
- **Runtime contract:** makes an HTTP call; routes response to `response` port on success, error details to `error` port on failure.

---

## Why

The workflow builder had only generic blocks (input/action/decision/output/delay/api/notification). None of them were aware of AquaFlow's domain — sensors, stations, MQTT, or alerts. Operators had no way to build automation rules like "if sensor X exceeds threshold → stop pump Y → create critical alert." These 7 blocks define the full vocabulary for industrial automation workflows on this platform.

The blocks are **data-only** in this task. Backend execution handlers are implemented in Task 11 (P8).

---

## How It Works (Architecture)

```
blocks.js                  → defines block vocabulary (this task)
blockRegistry.js           → auto-indexes blocks by type (no change needed)
blockFactory.js            → creates JointJS nodes from registry (no change needed)
BlockSidebar.jsx           → groups blocks by category, renders drag list (no change needed)
execution/handlers/*.ts    → implement runtime behavior (Task 11)
```

The `blockRegistry` builds a `Map` from `workflowBlocks` on import — adding blocks to `blocks.js` is sufficient for them to appear in the sidebar, be draggable onto the canvas, and have their properties editable in the Properties Panel.

---

## How to Verify

### UI verification (no backend needed)
1. Start the frontend: `cd frontend && npm start`
2. Log in → go to `/admin/builder`
3. In the left sidebar (Block panel), verify new categories appear:
   - **Industrial**: Sensor Read, Threshold Check, Pump Control, Alert Trigger, MQTT Publish, Station Control
   - **Integration**: HTTP Request (alongside existing API block)
4. Drag any new block to the canvas → node appears with correct color and label
5. Click the node → Properties Panel shows the correct fields for that block type
6. Search for "pump" in the block search bar → only Pump Control appears

### Registry verification (browser console)
```js
// Open browser console on /admin/builder
import { getBlockCategories } from './registry/blockRegistry';
// Or check via Redux DevTools / React DevTools
```

### Expected block counts
| Category | Blocks |
|----------|--------|
| Data | Input, Output |
| Logic | Action, Decision |
| Timing | Delay |
| Integrations | API |
| Messaging | Notification |
| **Industrial** | **Sensor Read, Threshold Check, Pump Control, Alert Trigger, MQTT Publish, Station Control** |
| **Integration** | **HTTP Request** |

### API tests (blocks are frontend-only in this task)
No backend endpoints changed. Backend execution handlers for these blocks are implemented in Task 11.

---

## Expected Results

| Action | Expected |
|--------|----------|
| Open `/admin/builder` | 7 new blocks visible in sidebar |
| Industrial category | 6 blocks listed |
| Integration category | HTTP Request block listed (alongside existing API) |
| Drag Sensor Read to canvas | Node appears in `#0ea5e9` blue with `trigger` input port and `value`/`status` output ports |
| Drag Threshold Check to canvas | Node appears in `#f59e0b` amber with `value` input, `pass`/`breach` output ports |
| Click any new block | Properties panel shows correct fields |
| Search "alert" | Alert Trigger block shown |
| Search "mqtt" | MQTT Publish block shown |

---

## Build Verification

```
npm run build  →  ✅ Compiled successfully (data-only change, no new imports)
Only pre-existing Header.js warnings remain
```
