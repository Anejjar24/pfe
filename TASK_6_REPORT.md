# TASK 6 — triggerSettings Lost on Page Refresh

## Status: DONE

---

## What Was Changed and Why

`triggerSettings` (workflow name, trigger type, cron config, active flag) lived only in
`BuilderPage`'s `useState` — not in localStorage.  A hard refresh reset it to defaults
even when the graph draft was faithfully restored.

The fix adds a **second localStorage slot** alongside the graph draft:

| Slot | Key pattern | Contains |
|------|-------------|----------|
| Graph draft | `workflow-draft:{id}` | JointJS nodes + edges JSON |
| Trigger settings | `workflow-trigger:{id}` | `{ name, triggerType, triggerConfig, isActive }` |

Three persistence events write the trigger slot; one lazy `useState` initializer
reads it back without any flash or extra `useEffect`.

---

## Files Modified

| File | Change |
|------|--------|
| `frontend/src/engine/autosaveManager.js` | Added `saveTriggerSettings`, `loadTriggerSettings`, `clearTriggerSettings` |
| `frontend/src/pages/BuilderPage.jsx` | Lazy `useState` init + `saveTriggerSettings` calls in save / settings-save / load handlers; UUID migration on first backend save |

---

## Diff Summary

### `autosaveManager.js`
```diff
+const TRIGGER_PREFIX = 'workflow-trigger:';
+const triggerKey = (id) => `${TRIGGER_PREFIX}${id}`;
+
+export function saveTriggerSettings(settings, id = 'new') {
+  try { localStorage.setItem(triggerKey(id), JSON.stringify(settings)); } catch {}
+}
+
+export function loadTriggerSettings(id = 'new') {
+  try {
+    const raw = localStorage.getItem(triggerKey(id));
+    if (raw) return JSON.parse(raw);
+  } catch { localStorage.removeItem(triggerKey(id)); }
+  return null;
+}
+
+export function clearTriggerSettings(id = 'new') {
+  localStorage.removeItem(triggerKey(id));
+}
```

### `BuilderPage.jsx` — imports
```diff
-import { clearWorkflowDraft, saveWorkflowDraft } from "engine/autosaveManager";
+import {
+  clearWorkflowDraft,
+  clearTriggerSettings,
+  loadTriggerSettings,
+  saveWorkflowDraft,
+  saveTriggerSettings,
+} from "engine/autosaveManager";
```

### `BuilderPage.jsx` — triggerSettings initial state
```diff
-const [triggerSettings, setTriggerSettings] = useState({
-  name: '',
-  triggerType: 'manual',
-  triggerConfig: {},
-  isActive: false,
-});
+// Lazy initializer: restore from localStorage on mount — no flash, no extra useEffect.
+const [triggerSettings, setTriggerSettings] = useState(() => {
+  return loadTriggerSettings('new') ?? {
+    name: '',
+    triggerType: 'manual',
+    triggerConfig: {},
+    isActive: false,
+  };
+});
```

### `BuilderPage.jsx` — handleSave
```diff
   saveWorkflowDraft(workflow, editor.workflowId);
+  saveTriggerSettings(triggerSettings, editor.workflowId);
   try {
     const result = await saveWorkflow(workflow, triggerSettings);
     if (editor.workflowId === 'new' && result?.id) {
       clearWorkflowDraft('new');
+      clearTriggerSettings('new');
+      saveTriggerSettings(triggerSettings, result.id);
       editor.setWorkflowId(result.id);
     }
```

### `BuilderPage.jsx` — handleLoadWorkflow
```diff
+  const localSettings = wf.id ? loadTriggerSettings(wf.id) : null;
+  const restoredSettings = localSettings ?? {
     name: wf.name || '',
     triggerType: wf.triggerType || 'manual',
     triggerConfig: wf.triggerConfig || {},
     isActive: wf.isActive || false,
+  };
-  setTriggerSettings({ name: wf.name || '', ... });
+  setTriggerSettings(restoredSettings);
+  if (wf.id) saveTriggerSettings(restoredSettings, wf.id);
```

### `BuilderPage.jsx` — handleSettingsSave
```diff
   setTriggerSettings(settings);
+  saveTriggerSettings(settings, editor.workflowId);
   const workflow = editor.refreshWorkflow();
```

---

## How to Verify

### Refresh test (unsaved / 'new' workflow)
1. Open `/admin/builder` with no loaded workflow
2. Click **Settings** → set Name = `"My Test Flow"`, Trigger = `scheduled`, Cron = `*/5 * * * *`, toggle **Active** ON → Save
3. Observe the toolbar badge shows `⏱ */5 * * * *` + green **Active** badge
4. **Hard-refresh the page** (Ctrl+Shift+R)
5. Expected: the toolbar badge immediately shows `⏱ */5 * * * *` + **Active** — no flash, restored synchronously by the lazy `useState` initializer

### Refresh test (saved workflow)
1. Click **Save** (backend running) — workflow is saved and gets a real UUID
2. Click **Settings** → change Name to `"Renamed Flow"` → Save
3. **Hard-refresh**
4. Expected: toolbar shows the updated name in the Settings modal (`initial` prop); badge label still reflects the trigger type

### Load from picker test
1. Open the Workflow Picker, select a workflow that has a cron trigger configured on the backend
2. Observe the toolbar badge updates immediately
3. Open Settings modal — fields reflect the loaded workflow's values
4. Change the cron expression locally but do **not** click the toolbar Save
5. **Refresh** — the locally-edited cron expression is restored (preferred over the backend value), because it was saved to `workflow-trigger:{id}` when the Settings modal was submitted

### localStorage inspection
Open DevTools → Application → Local Storage → `http://localhost:3000`:
- `workflow-trigger:new` — present while working on an unsaved draft
- `workflow-trigger:{uuid}` — present after first backend save; `workflow-trigger:new` is deleted

---

## Expected Results

| Scenario | Before | After |
|----------|--------|-------|
| Refresh while on unsaved ('new') draft | Name, trigger type, cron, active all reset | All restored synchronously on mount |
| Save for first time (new → UUID) | Settings stay in 'new' slot forever | 'new' slot cleared; copied to `{uuid}` slot |
| Load from picker | Settings from DB record | Prefers any locally-saved edits, falls back to DB |
| Settings modal change | Persisted only on backend save | Persisted to localStorage immediately |

---

## Side Effects / Follow-up Notes

- **No backend changes** — the fix is entirely client-side; the DB schema and API are untouched.
- **`clearWorkflowDraft` unchanged** — graph and trigger settings are stored under separate
  keys intentionally; clearing one does not affect the other.
- **`LEGACY_TYPE_MAP` pattern** — similar single-source-of-truth discipline: if the
  `triggerSettings` shape changes in future, only `loadTriggerSettings` needs a migration shim.
- TASK 7 will standardise the `node.data` / `node.properties` field name mismatch in the
  serialiser/deserialiser.
