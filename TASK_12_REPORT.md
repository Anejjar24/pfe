# TASK 12 — Real-Time Execution Feedback via Socket.IO

## Status: DONE

---

## What Was Changed and Why

When a user clicked **Run**, the canvas stayed completely static until the full
execution result appeared in the Properties panel.  For long-running workflows
(delays, HTTP calls, sensor reads) there was no indication of which node was
currently executing, which had passed, or which had errored.

The fix adds a live node-highlighting layer:

| Event | Visual |
|-------|--------|
| `workflow:node-executing` | Node border turns **amber** (node is actively running) |
| `workflow:node-executed` status=success | Border flashes **green** for 1.5 s, then resets |
| `workflow:node-executed` status=error | Border turns **red** (held until run ends + 3 s) |
| `workflow:completed` | All borders reset to original colours after 2 s |
| `workflow:failed` | Red node stays red; other nodes reset after 3 s |

A CSS transition (`stroke 0.35s ease`) makes the colour changes fade in smoothly
rather than snapping.

---

## Architecture

```
FlowExecutorService.execute()
        │ passes user.id
        ▼
WorkflowRunner.run(graph, input, userId)
        │ before each node  → broadcastToUser('workflow:node-executing')
        │ after each node   → broadcastToUser('workflow:node-executed')
        │ on success        → broadcastToUser('workflow:completed')
        │ on node error     → broadcastToUser('workflow:node-executed', { status:'error' })
        │ on fatal error    → broadcastToUser('workflow:failed')
        ▼
RealtimeService.broadcastToUser(userId, event, payload)
        │
        ▼  (Socket.IO — existing authenticated connection)
useExecutionFeedback(graphRef, workflowId)   ← new hook in the builder
        │ getCell(nodeId) on the live dia.Graph
        │ cell.attr('body/stroke', colour)
        │ setTimeout → reset to original colour
        ▼
JointJS canvas  — node borders animate via CSS transition
```

---

## Files Modified / Created

| File | Change |
|------|--------|
| `backend/src/execution/engine/workflow-runner.ts` | Injected `RealtimeService`; added `userId?` param to `run()`; emit 5 event types around the node execution loop |
| `backend/src/flows/flow-executor.service.ts` | Pass `options.user?.id` as `userId` to `runner.run()` |
| `frontend/src/hooks/useExecutionFeedback.js` | **New** — Socket.IO listener + JointJS cell highlighter |
| `frontend/src/hooks/useWorkflowEditor.js` | Import + call `useExecutionFeedback(graph.graphRef, graph.workflowId)` |
| `frontend/src/pages/workflowBuilder.css` | CSS transition on `.joint-element rect.body` |

No module changes needed — `RealtimeModule` was already imported in `FlowsModule`
and exports `RealtimeService`.

---

## Diff Summary

### `workflow-runner.ts`
```diff
-constructor(private readonly nodeExecutor: NodeExecutor) {}
+constructor(
+  private readonly nodeExecutor: NodeExecutor,
+  private readonly realtimeService: RealtimeService,
+) {}

-async run(graph, input = {}): Promise<ExecutionResult> {
+async run(graph, input = {}, userId?: string): Promise<ExecutionResult> {
   // ...
+  if (userId) this.realtimeService.broadcastToUser(userId, 'workflow:started', {...});
   while (queue.length) {
     // ...
+    if (userId) this.realtimeService.broadcastToUser(userId, 'workflow:node-executing', {...});
     try {
       output = await this.nodeExecutor.execute(node, previousInput, context);
     } catch (nodeErr) {
+      if (userId) this.realtimeService.broadcastToUser(userId, 'workflow:node-executed',
+        { ..., status: 'error' });
       throw nodeErr;
     }
+    if (userId) this.realtimeService.broadcastToUser(userId, 'workflow:node-executed',
+      { ..., status: 'success' });
   }
+  if (userId) this.realtimeService.broadcastToUser(userId, 'workflow:completed', {...});
```

### `flow-executor.service.ts`
```diff
-result = await this.runner.run(graph, input);
+result = await this.runner.run(graph, input, options.user?.id);
```

### `useExecutionFeedback.js` (new)
Key design decisions:
- **Own socket connection** — reuses the JWT from Redux store but opens a dedicated
  `io()` to avoid coupling with the app-level `useSocket` subscription list
- **`workflowIdRef`** — tracks current workflowId via ref so the socket event
  closure always sees the latest value without recreating the socket on ID changes
- **`timers: Map`** — per-node timeout handles; clearing before applying a new stroke
  prevents a rapid sequence (executing → success → executing) from leaving stale resets
- **`resetAllStrokes`** — triggered on `workflow:completed`; gives 2 s for the final
  green flashes to be visible before everything returns to normal

### `workflowBuilder.css`
```diff
+.joint-paper-host svg .joint-element rect.body {
+  transition: stroke 0.35s ease, stroke-width 0.25s ease;
+}
```

---

## How to Verify

### Manual run
1. Build a workflow: Input → Delay (500 ms) → Action → Output
2. Click **Run**
3. Watch the canvas:
   - `Input` node border goes amber → green (fast — sync node)
   - `Delay` node border goes amber (waits 500 ms) → green
   - `Action` node border goes amber → green
   - `Output` node border goes amber → green
   - After 2 s, all borders return to their original colours

### Error node (red highlight)
1. Add an `HTTP Request` node with URL left as `https://` (blank host)
2. Run — the HTTP Request node should flash red (invalid URL returns error)

### Multi-workflow isolation
1. Load Workflow A on the canvas
2. In a different browser tab, trigger a run of Workflow B
3. Workflow A's canvas should NOT highlight (events filtered by `workflowId`)

### Socket event inspection (DevTools)
1. Open DevTools → Network → WS
2. Click Run
3. Observe frames: `workflow:started`, `workflow:node-executing` (×N),
   `workflow:node-executed` (×N with status), `workflow:completed`

### Unauthenticated fallback
1. Log out (token cleared from Redux store)
2. `useExecutionFeedback` short-circuits (`if (!token) return`) — no socket opened

---

## Expected Results

| Action | Before | After |
|--------|--------|-------|
| Click Run | Canvas is static | Nodes pulse amber → green as they execute |
| Failing node | Only post-run error in Properties panel | Node turns red immediately |
| Long async node (delay/HTTP) | No progress indicator | Amber border for the full duration |
| Run completes | Green flash stays forever (no cleanup) | Borders reset to original after 2 s |
| Switching workflows | N/A | Events filtered — only current workflow highlighted |

---

## Side Effects / Follow-up Notes

- **All 12 tasks are now complete.** Phase 1 (bug fixes, Tasks 1–8) and Phase 2
  (new enhancements, Tasks 9–12) are done.
- **No DB migration required** for any of the 12 tasks.
- **`WorkflowSchedulerService`** (scheduled/MQTT triggers) currently calls
  `this.runner.run(graph, input)` without a `userId` — scheduled runs won't emit
  Socket.IO events.  That's intentional: nobody is watching the canvas for an
  automated run.  If needed, the scheduler could look up the workflow owner's userId
  and pass it through.
