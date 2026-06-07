# TASK 7 — node.data vs node.properties Field Mismatch

## Status: DONE

---

## What Was Changed and Why

Two different names were being used for the same concept — a node's runtime property
bag — depending on which layer of the stack you were looking at:

| Layer | Field name | Example |
|-------|-----------|---------|
| External JSON (serialized graph) | `data` | `{ "type":"action", "data":{"operation":"multiply"} }` |
| Backend `WorkflowNode` interface | `data` | `node.data?.operation` |
| Internal JointJS cell (`workflow` attr) | `properties` | `workflow.properties.label` |
| `createWorkflowNode` overrides param | `properties` | `overrides.properties` |

Every backend handler already used `node.data`.  The serializer already wrote `data`
to JSON.  But the internal JointJS cell stored the same bag under `properties`, and
the `createWorkflowNode` override key was also `properties`.  This caused a silent
rename on every deserialization round-trip, and meant that any developer reading
`PropertiesPanel` or `NodeEditorModal` had to remember which layer they were in.

The fix renames the internal field and the override key to `data` everywhere, making
the shape identical at all three layers.  No localStorage migration is needed because
the **serialized** format was already `data` — only the in-memory JointJS cell changed.

---

## Files Modified

| File | Change |
|------|--------|
| `frontend/src/registry/blockFactory.js` | `overrides.properties` → `overrides.data`; internal `workflow.properties` → `workflow.data` in both `createWorkflowNode` and `updateNodeProperties` |
| `frontend/src/engine/graphSerializer.js` | `workflow.properties` → `workflow.data` |
| `frontend/src/engine/graphDeserializer.js` | `properties: nodeData.data \|\| …` → `data: nodeData.data \|\| …` |
| `frontend/src/hooks/useJointGraph.js` | `workflowData.properties` → `workflowData.data` in `duplicateSelectedNode` |
| `frontend/src/components/properties/PropertiesPanel.jsx` | `workflow.properties?.[field.name]` → `workflow.data?.[field.name]` |
| `frontend/src/components/properties/NodeEditorModal.jsx` | `workflow?.properties` → `workflow?.data` |
| `frontend/src/utils/nodeHelpers.js` | `properties: { ...(workflow.properties \|\| {}) }` → `data: { ...(workflow.data \|\| {}) }` |

Backend unchanged — `workflow.types.ts` already declared `data?: Record<string, unknown>`.

---

## Diff Summary

### `blockFactory.js` — createWorkflowNode
```diff
-  const properties = {
+  // Named `data` to match the external JSON format and backend WorkflowNode interface.
+  const data = {
     ...getDefaultProperties(type),
-    ...(overrides.properties || {}),
+    ...(overrides.data || {}),
   };
   // ...
-      label: { text: properties.label || definition.title, ... },
+      label: { text: data.label || definition.title, ... },
   // ...
   node.set("workflow", {
     type, title, icon, color,
-    properties,
+    data,
   });
```

### `blockFactory.js` — updateNodeProperties
```diff
-  const nextProperties = { ...(workflow.properties || {}), ...properties };
-  node.set("workflow", { ...workflow, properties: nextProperties });
-  node.attr("label/text", nextProperties.label || workflow.title || workflow.type);
+  const nextData = { ...(workflow.data || {}), ...properties };
+  node.set("workflow", { ...workflow, data: nextData });
+  node.attr("label/text", nextData.label || workflow.title || workflow.type);
```

### `graphSerializer.js`
```diff
-        data: workflow.properties || {},
+        data: workflow.data || {},
```

### `graphDeserializer.js`
```diff
-      properties: nodeData.data || nodeData.properties || {},
+      data: nodeData.data || nodeData.properties || {},
```

### `useJointGraph.js`
```diff
-      { properties: { ...(workflowData.properties || {}) } }
+      { data: { ...(workflowData.data || {}) } }
```

### `PropertiesPanel.jsx`
```diff
-                value={workflow.properties?.[field.name]}
+                value={workflow.data?.[field.name]}
```

### `NodeEditorModal.jsx`
```diff
-    setProperties(workflow?.properties || {});
+    setProperties(workflow?.data || {});
```

### `utils/nodeHelpers.js`
```diff
-    properties: { ...(workflow.properties || {}) },
+    data: { ...(workflow.data || {}) },
```

---

## How to Verify

### Properties panel shows correct values
1. Open `/admin/builder`
2. Drag an **Action** node onto the canvas
3. Properties panel on the right should show Operation = `multiply`, Factor = `2`
4. Change Factor to `5`, confirm the canvas label does not error

### Duplicate preserves data
1. Select a node and click **Clone** (or Ctrl+D if wired)
2. Click the duplicate — Properties panel should show the same field values as the original

### Round-trip (serialize → deserialize)
1. Build a small workflow (Input → Action → Output)
2. Click **Export** — inspect the downloaded JSON; each node should have `"data": { ... }` (not `"properties"`)
3. Click **Import** with that JSON — nodes restore with correct property values

### Double-click edit modal
1. Double-click a node to open the full edit modal
2. Change a field value and click **Save**
3. The Properties panel should reflect the new value immediately

### Verification command (zero stale refs)
```
grep -rn "workflow\.properties\|overrides\.properties" frontend/src --include="*.js" --include="*.jsx"
# Expected: no output
```

---

## Expected Results

- Node data displayed in the Properties panel matches what was set during creation
- Duplicate node copies all property values correctly
- Export JSON always has `data` (never `properties`) on every node
- Import JSON with either `data` or `properties` field still deserializes correctly
  (the `nodeData.data || nodeData.properties` fallback in `graphDeserializer` handles
  old saved files transparently)
- No `console.error` or blank property panels in the browser

---

## Side Effects / Follow-up Notes

- **Backward compat preserved** — `graphDeserializer.js` still reads `nodeData.properties`
  as a fallback, so any JSON file exported before TASK 7 will still load correctly.
- **`nodeHelpers.cloneNodeOffset`** was updated even though it is currently unused
  (exported but never imported).  It's kept consistent to avoid a confusing inconsistency
  if it is imported in future.
- TASK 8 will add a **Fit-to-screen** button using `paper.fitToContent()` and an optional
  minimap/navigator.
