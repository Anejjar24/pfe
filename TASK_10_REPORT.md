# TASK 10 — One Incoming Edge Per Input Port

## Status: DONE

---

## What Was Changed and Why

Nothing in the original codebase prevented two (or more) edges from wiring into the
same input port on a node.  At execution time `WorkflowRunner` picks
`previousEdges[0]` — so a second wire is silently ignored, making the workflow
non-deterministic from the operator's perspective.

The fix enforces the invariant at **two independent layers**:

| Layer | Mechanism | When it fires |
|-------|-----------|---------------|
| UI (canvas) | JointJS `validateConnection` extended with a port-occupancy scan | While the user is dragging a new link; JointJS won't let the connection "snap" if the port is occupied |
| Backend | `FlowValidatorService` — new `occupiedPorts` Set in the edge loop | On every `POST /flows`, `PUT /flows/:id`, and `POST /flows/execute`; catches programmatic submissions that bypass the canvas |

The UI layer provides **immediate visual feedback** (the target magnet turns grey
rather than green when occupied); the backend layer is the **safety net**.

---

## Files Modified

| File | Change |
|------|--------|
| `frontend/src/hooks/useJointGraph.js` | Extended `validateConnection` with port-occupancy check using `graph.getLinks()` |
| `backend/src/flows/flow-validator.service.ts` | Added `occupiedPorts: Set<string>` in edge loop; throws `BadRequestException` on duplicate `(target, targetPort)` |

---

## Diff Summary

### `useJointGraph.js` — validateConnection
```diff
         validateConnection: (sourceView, sourceMagnet, targetView, targetMagnet) => {
           if (!sourceMagnet || !targetMagnet || sourceView === targetView) return false;
-          return (
-            sourceMagnet.getAttribute("port-group") === "output" &&
-            targetMagnet.getAttribute("port-group") === "input"
-          );
+          // Direction rule
+          if (
+            sourceMagnet.getAttribute("port-group") !== "output" ||
+            targetMagnet.getAttribute("port-group") !== "input"
+          ) return false;
+
+          // One-incoming-edge-per-input-port rule
+          const targetPortId = targetMagnet.getAttribute("port");
+          const targetCellId = String(targetView.model.id);
+          const alreadyOccupied = graph.getLinks().some((link) => {
+            const t = link.target();
+            return String(t.id) === targetCellId && t.port === targetPortId;
+          });
+          return !alreadyOccupied;
         },
```

### `flow-validator.service.ts` — edge loop
```diff
+    const occupiedPorts = new Set<string>();
+
     graph.edges.forEach((edge) => {
       // ... existing source/target existence + self-loop checks ...
+
+      if (edge.targetPort) {
+        const portKey = `${edge.target}:${edge.targetPort}`;
+        if (occupiedPorts.has(portKey)) {
+          throw new BadRequestException(
+            `Port "${edge.targetPort}" on node "${edge.target}" already has an ` +
+            'incoming connection. Each input port may have at most one incoming edge.',
+          );
+        }
+        occupiedPorts.add(portKey);
+      }
       adjacency.get(edge.source)!.push(edge.target);
     });
```

---

## How to Verify

### Canvas — connection is blocked at the UI
1. Open `/admin/builder`
2. Drag an **Action** node (one `in` port) onto the canvas and connect a source to it
3. Try to drag a **second** edge from a different node onto the same `in` port
4. Expected: the port magnet stays **grey** (not green) while hovering — JointJS refuses
   to snap the connection. Releasing the mouse drops the link without attaching it.

### Canvas — other ports on the same node remain free
1. Drag a **Decision** node (one `in`, two outputs: `true` / `false`)
2. Connect a source to `in`
3. Try to connect a second source to `in` — blocked ✓
4. Verify that the `true` and `false` output ports on the Decision node are still
   connectable outward (port-group = "output", not checked by this rule)

### Backend — validator rejects duplicate ports
```bash
TOKEN=$(curl -s -X POST http://localhost:3001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@aquaflow.local","password":"Admin123!"}' | jq -r '.accessToken')

curl -s -X POST http://localhost:3001/api/flows/execute \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "graph": {
      "nodes": [
        {"id":"n1","type":"input","data":{"value":"a"}},
        {"id":"n2","type":"input","data":{"value":"b"}},
        {"id":"n3","type":"action","data":{"operation":"multiply","factor":2}}
      ],
      "edges": [
        {"id":"e1","source":"n1","target":"n3","targetPort":"in"},
        {"id":"e2","source":"n2","target":"n3","targetPort":"in"}
      ]
    },
    "input": {}
  }' | jq '{statusCode, message}'
```
Expected: `{ "statusCode": 400, "message": "Port \"in\" on node \"n3\" already has an incoming connection..." }`

### Backend — edges without targetPort are unaffected
Old workflows serialized without port metadata (no `targetPort` field) pass the
validator unchanged — the `if (edge.targetPort)` guard skips the occupancy check for
null/undefined ports.

### UI — undo still works
1. Connect a valid edge
2. Ctrl+Z — edge is removed by undo
3. The port is now free again; a new edge can be drawn to it

---

## Expected Results

| Scenario | Before | After |
|----------|--------|-------|
| Second edge to occupied port (canvas) | Allowed, silently ignored at runtime | Blocked by `validateConnection`; magnet stays grey |
| Programmatic duplicate port (API) | Accepted, non-deterministic execution | `400 Bad Request` with clear message |
| Edges without `targetPort` field (legacy) | Validated normally | Still validated normally (guard skips null) |
| Undo of an edge frees the port | N/A | Port becomes occupiable again immediately |

---

## Side Effects / Follow-up Notes

- **`markAvailable: true`** (already set on the paper) highlights eligible target
  ports in green while dragging — the occupied port simply won't glow green, giving
  the operator immediate visual feedback with no extra CSS needed.
- **No migration needed** — the backend check uses `edge.targetPort` which is already
  stored in every edge serialised since the project started.
- TASK 11 will add the **Execution History Panel** using `GET /api/flows/:id/executions`.
