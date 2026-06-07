# TASK 11 — Execution History Panel

## Status: DONE

---

## What Was Changed and Why

Operators had no way to see past runs of a workflow.  After clicking **Run** the
only feedback was the inline execution result shown in the Properties panel — which
disappears on the next click.  There was no audit trail, no way to compare runs, and
no visibility into which step failed on a previous execution.

The `WorkflowExecution` entity, the `FlowExecutorService` persistence logic, and the
`GET /api/flows/:id/executions` endpoint already existed and were fully functional.
Only the frontend was missing.

The fix adds:
- `loadExecutions(workflowId)` to the API service layer
- `ExecutionHistoryModal` — a scrollable modal listing the last 50 runs, with each
  row expandable to show the full per-node step log
- A **History** button in the builder toolbar that opens the modal (disabled and
  tooltipped when the workflow is unsaved)

---

## Files Modified / Created

| File | Change |
|------|--------|
| `frontend/src/services/workflowApi.js` | Added `loadExecutions(workflowId)` |
| `frontend/src/components/workflow/ExecutionHistoryModal.jsx` | **New file** — full history modal |
| `frontend/src/pages/BuilderPage.jsx` | Import + `historyOpen` state + History button + mount modal |
| `frontend/src/pages/workflowBuilder.css` | Added section 19: execution history styles |

Backend unchanged — controller, service, and entity were already complete.

---

## Diff Summary

### `workflowApi.js`
```diff
+export function loadExecutions(workflowId) {
+  return apiClient.get(`/flows/${workflowId}/executions`).then((res) => res.data);
+}
```

### `ExecutionHistoryModal.jsx` (new — 160 lines)
Key design decisions:
- **`isSaved` guard** — shows a hint instead of fetching when `workflowId === 'new'`
- **Re-fetch on open** — `useEffect` deps `[isOpen, workflowId]` so switching workflows
  then reopening the modal always shows fresh data
- **Expandable rows** — local `expandedId` state; clicking the summary row toggles the
  step log table
- **Error strip** — `ex.errorMessage` is always visible (no expand needed) to surface
  failures immediately
- **`STATUS_COLOUR` map** — maps `completed/failed/running/cancelled/paused` to
  Reactstrap badge colours

### `BuilderPage.jsx`
```diff
+import ExecutionHistoryModal from "components/workflow/ExecutionHistoryModal";
 // ...
+  const [historyOpen, setHistoryOpen] = useState(false);
 // ...
+          <Button
+            size="sm" color="secondary"
+            onClick={() => setHistoryOpen(true)}
+            disabled={editor.workflowId === 'new'}
+            title={editor.workflowId === 'new'
+              ? 'Save the workflow to view history'
+              : 'Execution history'}
+          >
+            <i className="ni ni-bullet-list-67 mr-1" />
+            History
+          </Button>
 // ...
+      <ExecutionHistoryModal
+        isOpen={historyOpen}
+        onClose={() => setHistoryOpen(false)}
+        workflowId={editor.workflowId}
+      />
```

### `workflowBuilder.css` (section 19 — new)
Styles for: `.exec-history-list`, `.exec-history-row`, `.exec-history-summary`,
`.exec-status-badge`, `.exec-time`, `.exec-meta`, `.exec-trigger-badge`,
`.exec-chevron`, `.exec-error-msg`, `.exec-step-log`, `.exec-step-table`,
`.exec-step-num`, `.exec-step-id`, `.exec-step-value`, `pre` inside step values.

---

## How to Verify

### History button state
1. Open `/admin/builder` with no loaded workflow (`workflowId === 'new'`)
2. The **History** button in the builder toolbar should be **greyed-out** (disabled)
3. Hover it — tooltip says "Save the workflow to view history"
4. Click **Save** (with backend running) — the button becomes enabled once the
   workflow is persisted and `editor.workflowId` is set to the real UUID

### Viewing history
1. With a saved workflow loaded, click **History**
2. Modal opens; shows a loading spinner briefly, then a list of past runs
3. Each row shows: status badge, timestamp, duration, node count, trigger source
4. Click any row to expand the step log — a table with columns #, Node ID, Type,
   Input, Output appears
5. Click the row again to collapse it

### Error execution is visible without expanding
1. Build a workflow with a `parse_json` data-transform node fed an invalid JSON string
2. Run it — it will succeed overall (error port fires), but if the `error` port leads
   to no node, the execution still completes
3. To get a FAILED execution: submit a graph that fails validation
4. The error message appears as a red strip directly under the summary row without
   needing to expand

### Fresh fetch on re-open
1. Click History (0 runs shown)
2. Close the modal
3. Click **Run**
4. Click History again — the new execution appears at the top (newest-first order)

### Unsaved workflow
```
workflowId === 'new'  →  "Save the workflow first to view its execution history."
```

---

## Expected Results

| Scenario | Result |
|----------|--------|
| Unsaved workflow | Button disabled; modal shows hint |
| Saved workflow, 0 runs | "No executions recorded yet — click Run..." |
| Saved workflow, N runs | Newest-first list; badge shows N |
| Expand a row | Step table with per-node input/output |
| Failed execution | Red error strip visible immediately |
| Re-open after new run | Fresh fetch, new run at top |

---

## Side Effects / Follow-up Notes

- **`loadExecutions` is used only by `ExecutionHistoryModal`** — no shared state,
  no Redux slice needed.
- The backend `take: 50` limit means the modal never shows more than 50 rows.
  If pagination is needed later, add a `?page=` query param to the API call.
- TASK 12 (the final task) will add **real-time execution feedback** via Socket.IO —
  highlighting nodes green/red on the canvas as the workflow executes.
