# TASK 2 — Single-workflow builder (localStorage keyed "local-workflow")

## Status: DONE

---

## What Was Changed and Why

The builder stored every workflow's draft under one fixed key
(`"workflow-builder-autosave"`), so opening a second saved workflow would
overwrite the previous one's local draft.  This task gives each workflow its own
localStorage slot and promotes the key from `"new"` → real UUID after the first
backend save.

---

## Files Modified

| File | Change |
|------|--------|
| `frontend/src/engine/autosaveManager.js` | Rewritten: keyed storage (`workflow-draft:{id}`); legacy key fall-through for backward compat |
| `frontend/src/engine/graphSerializer.js` | `serializeGraph(graph, workflowId)` — omits `id` when `'new'` so the backend generates its own UUID |
| `frontend/src/hooks/useJointGraph.js` | Added `workflowIdRef` + `workflowId` state + `setWorkflowId`; passes id to serializer |
| `frontend/src/hooks/useAutosave.js` | Added `id` as second parameter; passes it to `saveWorkflowDraft` |
| `frontend/src/hooks/useWorkflowEditor.js` | Threads `graph.workflowId` into `useAutosave` |
| `frontend/src/pages/BuilderPage.jsx` | `handleSave`: migrates `'new'` → real UUID after first backend save; `handleLoadWorkflow`: calls `editor.setWorkflowId(wf.id)` |
| `frontend/src/components/canvas/FlowCanvas.jsx` | Startup effect: `loadWorkflowDraft(currentEditor.workflowId)` instead of hardcoded default |

---

## Diff Summary

### `autosaveManager.js`
```diff
-const autosaveKey = "workflow-builder-autosave";
-export function saveWorkflowDraft(workflow) { ... }
-export function loadWorkflowDraft() { ... }
-export function clearWorkflowDraft() { ... }

+const PREFIX = 'workflow-draft:';
+const LEGACY_KEY = 'workflow-builder-autosave';      // backward compat
+export function saveWorkflowDraft(workflow, id='new') { ... }
+export function loadWorkflowDraft(id='new') {
+  // 1. keyed slot; 2. legacy fallback when id==='new'
+}
+export function clearWorkflowDraft(id='new') { ... }
```

### `graphSerializer.js`
```diff
-export function serializeGraph(graph) {
+export function serializeGraph(graph, workflowId) {
   ...
-  return { id: "local-workflow", ... }
+  const hasRealId = workflowId && workflowId !== 'new' && workflowId !== 'local-workflow';
+  return { ...(hasRealId ? { id: workflowId } : {}), ... }
```

### `useJointGraph.js`
```diff
+  const workflowIdRef = useRef('new');
+  const [workflowId, setWorkflowIdState] = useState('new');
+  const setWorkflowId = useCallback((id) => {
+    workflowIdRef.current = id;
+    setWorkflowIdState(id);
+  }, []);

   const refreshWorkflow = useCallback(() => {
-    const nextWorkflow = serializeGraph(graphRef.current);
+    const nextWorkflow = serializeGraph(graphRef.current, workflowIdRef.current);
   ...
-  return { graphRef, ..., zoom, ... }
+  return { graphRef, ..., workflowId, setWorkflowId, zoom, ... }
```

### `useAutosave.js`
```diff
-export function useAutosave(workflow, enabled = true) {
+export function useAutosave(workflow, id = 'new', enabled = true) {
-    saveWorkflowDraft(workflow);
+    saveWorkflowDraft(workflow, id);
```

### `useWorkflowEditor.js`
```diff
-  const autosaveStatus = useAutosave(graph.workflow);
+  const autosaveStatus = useAutosave(graph.workflow, graph.workflowId);
```

### `BuilderPage.jsx`
```diff
+import { clearWorkflowDraft, saveWorkflowDraft } from "engine/autosaveManager";
 ...
-    saveWorkflowDraft(workflow);
-    await saveWorkflow(workflow, triggerSettings);
+    saveWorkflowDraft(workflow, editor.workflowId);
+    const result = await saveWorkflow(workflow, triggerSettings);
+    if (editor.workflowId === 'new' && result?.id) {
+      clearWorkflowDraft('new');
+      editor.setWorkflowId(result.id);
+    }
 ...
+    if (wf.id) editor.setWorkflowId(wf.id);   // in handleLoadWorkflow
```

### `FlowCanvas.jsx`
```diff
-    deserializeGraph(..., loadWorkflowDraft() || starterWorkflow);
+    const draft = loadWorkflowDraft(currentEditor.workflowId || 'new');
+    deserializeGraph(..., draft || starterWorkflow);
```

---

## How to Verify

### Prerequisites
```bash
docker-compose up --build
# or: cd backend && npm run start:dev && cd frontend && npm start
```

### Browser steps

**Test A — Multiple workflow isolation**
1. Open `/admin/builder`; add some nodes; click **Save** — note the autosave status cycles to "Saved HH:MM:SS"
2. Open browser DevTools → Application → Local Storage
3. Confirm key `workflow-draft:{uuid}` exists (NOT the old `workflow-builder-autosave`)
4. Click Load (📂) → open a different saved workflow
5. Confirm a second key `workflow-draft:{other-uuid}` appears and the first is untouched

**Test B — Backward compatibility**
1. Manually create a `workflow-builder-autosave` key in localStorage with valid workflow JSON
2. Reload the page — the builder should load that legacy draft (backward compat fallback)
3. After one autosave cycle, the data should appear under `workflow-draft:new` and the legacy key is superseded

**Test C — First save UUID migration**
1. Start fresh (no drafts); build a workflow; click **Save**
2. Watch DevTools: before save → key is `workflow-draft:new`; after successful backend save → key changes to `workflow-draft:{backend-uuid}` and `workflow-draft:new` is removed

---

## Expected Results

- Each workflow draft lives under its own key: `workflow-draft:{id}`
- Fresh/unsaved workflows use `workflow-draft:new`
- On first backend save the `'new'` slot is freed and the real UUID slot takes over
- Loading a workflow via the picker switches the active autosave slot
- Legacy drafts saved under the old key are transparently read on the first page load

---

## Side Effects / Follow-up Notes

- **`handleSettingsSave`** also updated to pass `editor.workflowId` to `saveWorkflowDraft`
- The `graphSerializer` no longer hardcodes `id: "local-workflow"` — serialized JSON exports will omit the `id` field for unsaved workflows and embed the UUID for saved ones
- The backend's `FlowsService.create()` uses TypeORM `save()` which upserts by PK — so sending the same UUID on repeated POSTs correctly updates the record
- Old localStorage entries with key `"workflow-builder-autosave"` are NOT deleted automatically; they are simply superseded. A future cleanup task could remove them after migration.
