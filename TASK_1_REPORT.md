# TASK 1 — No Workflow List UI

## Status: DONE

---

## What Was Changed and Why

The builder had no way to browse or reload previously-saved workflows.
`workflowApi.js` already exposed `loadWorkflows()` but nothing in the UI called
it.  This task wires that call to a picker modal so operators can select any
saved workflow and instantly load it onto the canvas, with trigger settings
automatically restored.

---

## Files Modified

| File | Change |
|------|--------|
| `frontend/src/components/workflow/WorkflowPickerModal.jsx` | **NEW** — modal fetches the workflow list; user clicks a row to load |
| `frontend/src/components/canvas/CanvasToolbar.jsx` | Added `onLoad` prop + Load (folder-open) button between Save and Export |
| `frontend/src/components/canvas/FlowCanvas.jsx` | Accepted `onLoad` prop and forwarded it to `CanvasToolbar` |
| `frontend/src/pages/BuilderPage.jsx` | Added `pickerOpen` state, `handleLoadWorkflow(wf)`, wired modal |
| `frontend/src/pages/workflowBuilder.css` | Added section 18: picker list + item styles |

---

## Diff Summary

### `WorkflowPickerModal.jsx` (new — 109 lines)
- On `isOpen`: calls `loadWorkflows()` → renders scrollable list of workflow cards
- Each card shows folder icon, name, node count, saved date, trigger-type badge, run-count badge
- Click → `onSelect(wf)` (full entity incl. `.graph`) then closes
- States: loading spinner / error alert / empty-state message

### `CanvasToolbar.jsx`
```diff
-export default function CanvasToolbar({ ..., onImport, onReset, ... }) {
+export default function CanvasToolbar({ ..., onImport, onLoad, onReset, ... }) {
     <button onClick={onSave}>💾</button>
+    <button onClick={onLoad} title="Load saved workflow">📂</button>
     <button onClick={onExport}>⬇️</button>
```

### `FlowCanvas.jsx`
```diff
-export default function FlowCanvas({ editor, onSave }) {
+export default function FlowCanvas({ editor, onLoad, onSave }) {
        onImport={editor.importJsonFile}
+       onLoad={onLoad}
```

### `BuilderPage.jsx`
```diff
+import WorkflowPickerModal from "components/workflow/WorkflowPickerModal";
+const [pickerOpen, setPickerOpen] = useState(false);
+const handleLoadWorkflow = (wf) => {
+  if (wf.graph) editor.importWorkflow(wf.graph);
+  setTriggerSettings({ name: wf.name, triggerType: wf.triggerType,
+    triggerConfig: wf.triggerConfig, isActive: wf.isActive });
+};
-<FlowCanvas editor={editor} onSave={handleSave} />
+<FlowCanvas editor={editor} onLoad={() => setPickerOpen(true)} onSave={handleSave} />
+<WorkflowPickerModal isOpen={pickerOpen} onClose={...} onSelect={handleLoadWorkflow} />
```

---

## How to Verify

### Prerequisites
```bash
docker-compose up --build
# or: cd backend && npm run start:dev   (port 3001)
#     cd frontend && npm start          (port 3000)
```

### Browser steps

1. Log in: `admin@aquaflow.local / Admin123!`
2. Go to `/admin/builder`
3. Build a 2-node graph (Input → Output) → click **Save** (floppy disk icon)
4. Reload the page or drag nodes away so the canvas differs
5. Click the **folder-open** icon (new — sits between Save and Export)
6. **Load Workflow** modal opens — the saved workflow appears in the list
7. Click the workflow row — modal closes, canvas is populated with saved graph
8. Verify the trigger badge in the top toolbar reflects the loaded workflow's trigger type

### curl check
```bash
TOKEN=$(curl -s -X POST http://localhost:3001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@aquaflow.local","password":"Admin123!"}' | jq -r '.accessToken')

curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:3001/api/flows | jq '.[].name'
```

---

## Expected Results

- Folder-open button visible in the canvas toolbar between Save and Export
- Clicking it opens a scrollable modal listing all saved workflows
- Each row: name, node count, saved date, trigger-type pill, run-count pill
- Clicking a row: closes modal, loads graph onto canvas, restores triggerSettings badge

---

## Side Effects / Follow-up Notes

- **Offline**: if backend unreachable, modal shows error alert — no crash
- **triggerSettings restored**: toolbar badge and Settings modal reflect loaded workflow's config
- No extra API round-trip needed — `GET /api/flows` (findAll) returns full JSONB graph
- TASK 2 will change autosave to per-workflow keys; `handleLoadWorkflow` will need to switch
  the active key so subsequent autosaves target the correct workflow
