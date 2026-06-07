# TASK 5 — Duplicate "api" / "http-request" Block

## Status: DONE

---

## What Was Changed and Why

Two block types did identical things:

| Type | Category | Properties |
|------|----------|------------|
| `api` | Integrations | label, method (3 opts), url |
| `http-request` | Integration | label, method (5 opts), url, headers, body |

`http-request` is strictly richer (two output ports: response + error; headers/body
fields; all 5 HTTP methods).  Having both in the sidebar confused operators.

The fix is a **three-layer strategy**:

1. **Remove from sidebar** — delete the `api` entry from `blocks.js`. Users can no
   longer drag new `api` nodes onto the canvas.
2. **Lazy migration on load** — `graphDeserializer.js` now maps `type:"api"` →
   `type:"http-request"` via `LEGACY_TYPE_MAP` before constructing the JointJS
   element.  The first time a user opens an old workflow and saves it, all `api`
   nodes become `http-request` nodes permanently.
3. **Backend backward compat** — `'api'` remains in `validTypes` and the
   `case 'api':` executor branch remains, so workflows already saved in the
   PostgreSQL DB continue to validate and execute without any DB migration script.

---

## Files Modified

| File | Change |
|------|--------|
| `frontend/src/data/blocks.js` | Removed the 13-line `api` block object |
| `frontend/src/engine/graphDeserializer.js` | Added `LEGACY_TYPE_MAP` + `resolveNodeType()` shim; applied before `createWorkflowNode()` |
| `backend/src/execution/engine/node-executor.ts` | Added explanatory comment above `case 'api':` (no functional change) |
| `backend/src/flows/flow-validator.service.ts` | Moved `'api'` to its own line with an explanatory comment (no functional change) |

---

## Diff Summary

### `blocks.js`
```diff
-  {
-    type: "api",
-    title: "API",
-    icon: "fa-cloud-arrow-up",
-    category: "Integrations",
-    description: "Describes an external API call.",
-    color: "#dc2626",
-    inputs: [{ id: "in", label: "Request" }],
-    outputs: [{ id: "out", label: "Response" }],
-    properties: [
-      { name: "label", ..., defaultValue: "API Request" },
-      { name: "method", ..., options: ["GET","POST","PUT","PATCH","DELETE"] },
-      { name: "url", ..., defaultValue: "https://api.example.com" },
-    ],
-  },
```

### `graphDeserializer.js`
```diff
+const LEGACY_TYPE_MAP = {
+  api: 'http-request',   // consolidated in TASK 5
+};
+
+function resolveNodeType(type) {
+  return LEGACY_TYPE_MAP[type] ?? type;
+}
+
 workflow.nodes.forEach((nodeData) => {
+  const type = resolveNodeType(nodeData.type);   // ← migrate before create
-  const node = createWorkflowNode(nodeData.type, ...);
+  const node = createWorkflowNode(type, ...);
```

### `node-executor.ts`
```diff
+  // 'api' is a legacy alias for 'http-request' — kept so that workflows
+  // saved in the DB before TASK 5 can still be executed without a migration.
   case 'api': return this.httpRequestHandler.execute(node, input);
```

### `flow-validator.service.ts`
```diff
-  'input', 'output', 'action', 'decision', 'delay', 'api', 'notification',
+  'input', 'output', 'action', 'decision', 'delay', 'notification',
+  // 'api' is a legacy alias — retained so DB workflows remain valid.
+  'api',
```

---

## How to Verify

### Browser — Sidebar check
1. Open `/admin/builder`
2. Expand the block sidebar — confirm **no "API" block** appears under any category
3. Confirm **"HTTP Request"** block appears under **Integration**

### Migration — Load an old workflow with `api` nodes
1. Import this JSON file via the **Import** (⬆) button:
```json
{
  "name": "legacy-api-test",
  "nodes": [
    {"id":"n1","type":"input","position":{"x":80,"y":120},"data":{"label":"In","value":"1"}},
    {"id":"n2","type":"api","position":{"x":320,"y":120},"data":{"label":"Old API","method":"GET","url":"https://httpbin.org/get"}},
    {"id":"n3","type":"output","position":{"x":580,"y":120},"data":{"label":"Out","format":"json"}}
  ],
  "edges":[
    {"id":"e1","source":"n1","sourcePort":"out","target":"n2","targetPort":"in"},
    {"id":"e2","source":"n2","sourcePort":"out","target":"n3","targetPort":"in"}
  ]
}
```
2. The middle node should render as **"HTTP Request"** (teal/orange border, HTTP Request icon)
3. Click the node — the Properties panel should show HTTP Request fields (method, url, headers, body)
4. Click **Save** — the graph is persisted with `type:"http-request"` nodes only

### Backend — Old api nodes still executable
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
        {"id":"n1","type":"input","data":{"value":"test"}},
        {"id":"n2","type":"api","data":{"method":"GET","url":"https://httpbin.org/get"}},
        {"id":"n3","type":"output","data":{"format":"json"}}
      ],
      "edges": [
        {"id":"e1","source":"n1","target":"n2"},
        {"id":"e2","source":"n2","target":"n3"}
      ]
    },
    "input": {}
  }' | jq '.status'
```
Expected: `"success"` (the `api` node still executes via `httpRequestHandler`)

---

## Expected Results

- **Sidebar**: Only `http-request` ("HTTP Request") block visible. No "API" block.
- **Old JSONs loaded via Import**: `api` nodes silently become `http-request` nodes — correct icon, correct properties panel, correct execution.
- **Old DB workflows executed via curl**: Return `"success"` — backward compat preserved.
- **New workflows**: Can only contain `http-request` nodes (api is no longer drag-droppable).

---

## Side Effects / Follow-up Notes

- **No DB migration script needed.** The lazy-migration approach means the DB is
  cleaned up organically: every time a user opens an old workflow and saves it,
  the JSONB `graph` column is rewritten with `http-request` nodes.
- **`LEGACY_TYPE_MAP` is the single place** to record future renames. Any developer
  adding a new rename just adds one line to that map.
- TASK 9 will add the `data-transform` block. When adding it, remember to also add
  it to `validTypes` in `flow-validator.service.ts` and to the `WorkflowNodeType`
  union in `workflow.types.ts`.
