# TASK 9 — Add `data-transform` Node Type

## Status: DONE

---

## What Was Changed and Why

The workflow builder had no general-purpose data-shaping node.  Operators had to
chain `action` nodes (numeric only) or `http-request` nodes (external round-trip) just
to rename a key, extract a nested field, or coerce a type.

`data-transform` fills that gap: a pure in-process node that reads the incoming value,
applies one of seven operations, and routes the result to an `out` or `error` output
port.

### Operations

| Operation | What it does |
|-----------|-------------|
| `extract_field` | Returns `input[field]` — unwraps one key from an object |
| `set_field` | Returns `{ ...input, [field]: value }` — adds/overwrites a key |
| `delete_field` | Returns a shallow copy of input without the named key |
| `to_number` | `Number(input)` |
| `to_string` | `String(input)` |
| `parse_json` | `JSON.parse(String(input))` — routes to `error` if malformed |
| `stringify_json` | `JSON.stringify(input)` |

### Port routing convention
Follows the same `{ branch: '<portId>' }` pattern already used by `decision`,
`threshold-check`, and `http-request`:
- Success → `{ value: <transformed>, branch: 'out' }`
- Failure → `{ error: <message>,     branch: 'error' }`

`WorkflowRunner.filterDecisionEdges` reads the `branch` field and activates only the
matching outgoing edge, so connecting the `error` port to an `alert-trigger` node
works out of the box.

---

## Files Modified / Created

| File | Change |
|------|--------|
| `frontend/src/data/blocks.js` | Added `data-transform` block definition (Data category, cyan, 2 outputs) |
| `backend/src/common/types/workflow.types.ts` | Added `'data-transform'` to `WorkflowNodeType` union |
| `backend/src/flows/flow-validator.service.ts` | Added `'data-transform'` to `validTypes` |
| `backend/src/execution/handlers/data-transform.handler.ts` | **New file** — `DataTransformHandler` with 7 operations |
| `backend/src/execution/engine/node-executor.ts` | Imported handler; added `case 'data-transform':` |

---

## Diff Summary

### `blocks.js`
```diff
+  {
+    type: "data-transform",
+    title: "Data Transform",
+    icon: "fa-filter",
+    category: "Data",
+    description: "Extracts, mutates, or converts fields in the incoming data payload.",
+    color: "#0891b2",
+    inputs: [{ id: "in", label: "In" }],
+    outputs: [
+      { id: "out", label: "Transformed" },
+      { id: "error", label: "Error" },
+    ],
+    properties: [
+      { name: "label", ... defaultValue: "Transform" },
+      { name: "operation", type: "select", options: ["extract_field", "set_field",
+        "delete_field", "to_number", "to_string", "parse_json", "stringify_json"] },
+      { name: "field", label: "Field Name", type: "text" },
+      { name: "value", label: "Set Value (set_field only)", type: "text" },
+    ],
+  },
```

### `workflow.types.ts`
```diff
+  // Data blocks
+  | 'data-transform'
```

### `flow-validator.service.ts`
```diff
+  // Data blocks
+  'data-transform',
```

### `data-transform.handler.ts` (new)
```typescript
export class DataTransformHandler {
  execute(node: WorkflowNode, input: unknown) {
    const operation = String(node.data?.operation || 'extract_field');
    const field = String(node.data?.field ?? '');
    const setValue = node.data?.value;
    try {
      return { value: this.transform(operation, input, field, setValue), branch: 'out' };
    } catch (err) {
      return { error: err instanceof Error ? err.message : String(err), branch: 'error' };
    }
  }
  // ... private transform(operation, input, field, setValue) switch/case
}
```

### `node-executor.ts`
```diff
+import { DataTransformHandler } from '../handlers/data-transform.handler';
 // ...
+  private readonly dataTransformHandler = new DataTransformHandler();
 // ...
+      case 'data-transform': return this.dataTransformHandler.execute(node, input);
```

---

## How to Verify

### Sidebar
1. Open `/admin/builder`
2. Expand the block sidebar — confirm **"Data Transform"** appears under the **Data** category
3. Drag it onto the canvas — it should render with a cyan border, a filter icon, `Transformed` and `Error` output ports on the right

### Properties panel
Click the node:
- Operation (select): `extract_field` (default)
- Field Name (text input)
- Set Value (text input, relevant for `set_field`)

### Backend execution — extract_field
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
        {"id":"n1","type":"input","data":{"value":"{\"temperature\":23.5,\"unit\":\"C\"}"}},
        {"id":"n2","type":"data-transform","data":{"operation":"extract_field","field":"temperature"}},
        {"id":"n3","type":"output","data":{"format":"json"}}
      ],
      "edges": [
        {"id":"e1","source":"n1","target":"n2"},
        {"id":"e2","source":"n2","sourcePort":"out","target":"n3","targetPort":"in"}
      ]
    },
    "input": {}
  }' | jq '{status,output}'
```
Expected: `{ "status": "success", "output": { "value": 23.5, "branch": "out" } }`

### Backend execution — set_field
Same as above but:
```json
{"id":"n2","type":"data-transform","data":{"operation":"set_field","field":"unit","value":"F"}}
```
Expected output: `{ "value": {"temperature":23.5,"unit":"F"}, "branch":"out" }`

### Backend execution — error routing (invalid JSON)
```json
{"id":"n2","type":"data-transform","data":{"operation":"parse_json","field":""}}
```
With input value `"not json"` — output node won't be reached (connected to `out`);
the `error` output fires instead.

### Validator — data-transform is accepted
```bash
curl -s -X POST http://localhost:3001/api/flows/validate \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"graph":{"nodes":[{"id":"n1","type":"data-transform","data":{}}],"edges":[]}}' \
  | jq '.valid'
```
Expected: `true`

---

## Expected Results

- Block appears in sidebar under **Data**, draggable onto canvas
- Properties panel shows 4 fields (label, operation select, field, value)
- Backend validates the type as legal
- All 7 operations execute correctly:
  - `extract_field` / `set_field` / `delete_field` — object mutations
  - `to_number` / `to_string` — coercions
  - `parse_json` — routes to `error` port on `SyntaxError`
  - `stringify_json` — JSON serialization
- Error path fires correctly and does not crash the runner

---

## Side Effects / Follow-up Notes

- **No migration needed** — the new type is additive; existing saved workflows are
  unaffected.
- **`WorkflowRunner.filterDecisionEdges` untouched** — it already supports any
  `branch` string, so the `out`/`error` routing works with zero engine changes.
- TASK 10 will enforce the **one-incoming-edge-per-input-port** rule in both the
  frontend `validateConnection` callback and the backend validator.
