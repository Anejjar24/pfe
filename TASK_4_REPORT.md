# TASK 4 — Cyclic Graph Vulnerability

## Status: DONE

---

## What Was Changed and Why

The workflow runner (`WorkflowRunner`) uses a BFS queue with a `visited` set, so
cycles don't crash the server — they cause nodes to be silently skipped.  But
users had no way to know their graph was invalid.  Worse, the validator accepted
cyclic graphs, so malformed workflows could be saved and later produce incorrect
execution results without any error.

This task adds a proper DFS-based cycle detector to `FlowValidatorService`.
Cyclic graphs are now rejected at both save time (`POST /flows`) and run time
(`POST /flows/execute`) with a descriptive 400 error that includes the cycle
path so the user knows exactly which nodes form the loop.

---

## Files Modified

| File | Change |
|------|--------|
| `backend/src/flows/flow-validator.service.ts` | Rewrote edge validation section to build an adjacency map; added private `detectCycles(nodeIds, adjacency)` method using three-colour DFS |

---

## Algorithm — Three-Colour DFS

```
WHITE (0) = not yet visited
GREY  (1) = on the current DFS call-stack (recursion in progress)
BLACK (2) = fully processed; no cycle reachable from this node

For each WHITE node → start DFS:
  Mark GREY, push onto path stack
  For each neighbour:
    GREY  → back-edge found → cycle!  Slice path stack to extract loop.
    WHITE → recurse
    BLACK → skip (safe, already proved acyclic)
  Mark BLACK, pop from path stack
```

The cycle path is extracted by slicing `stack` from `stack.indexOf(neighbour)` to
the end, then appending `neighbour` again so the first and last node are identical:
`["A","B","C","A"]` → error message `"A → B → C → A"`.

---

## Diff Summary

```diff
--- a/backend/src/flows/flow-validator.service.ts
+++ b/backend/src/flows/flow-validator.service.ts

@@ validate() — edge section @@

-    graph.edges.forEach((edge) => {
-      if (!nodeIds.has(edge.source) || !nodeIds.has(edge.target)) {
-        throw new BadRequestException(`Invalid edge...`);
-      }
-      if (edge.source === edge.target) {
-        throw new BadRequestException('Self-referencing...');
-      }
-    });

+    // Build adjacency list while validating each edge
+    const adjacency = new Map<string, string[]>();
+    for (const id of nodeIds) adjacency.set(id, []);
+
+    graph.edges.forEach((edge) => {
+      if (!nodeIds.has(edge.source) || !nodeIds.has(edge.target)) {
+        throw new BadRequestException(`Invalid edge from "${edge.source}"...`);
+      }
+      if (edge.source === edge.target) {
+        throw new BadRequestException(`Self-referencing edge on "${edge.source}"...`);
+      }
+      adjacency.get(edge.source)!.push(edge.target);
+    });
+
+    // ── Cycle detection ───────────────────────────────────────────────────────
+    const cycle = this.detectCycles(nodeIds, adjacency);
+    if (cycle) {
+      throw new BadRequestException(
+        `Workflow contains a cycle: ${cycle.join(' → ')}. ...`
+      );
+    }

+  private detectCycles(nodeIds, adjacency): string[] | null {
+    // THREE-COLOUR DFS — see JSDoc above
+    const dfs = (id) => { ... WHITE/GREY/BLACK logic ... };
+    for (const id of nodeIds) if (WHITE) { const cycle = dfs(id); ... }
+    return null;
+  }
```

---

## How to Verify

### Prerequisites
```bash
cd backend && npm run start:dev   # port 3001
```

### curl tests

**Get a token first:**
```bash
TOKEN=$(curl -s -X POST http://localhost:3001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@aquaflow.local","password":"Admin123!"}' | jq -r '.accessToken')
```

**Test A — Simple two-node cycle (A → B → A) should be REJECTED:**
```bash
curl -s -X POST http://localhost:3001/api/flows/execute \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "graph": {
      "nodes": [
        {"id":"A","type":"action","data":{"operation":"multiply","factor":2}},
        {"id":"B","type":"action","data":{"operation":"add","factor":1}}
      ],
      "edges": [
        {"id":"e1","source":"A","target":"B"},
        {"id":"e2","source":"B","target":"A"}
      ]
    },
    "input": {}
  }' | jq '.message'
```
Expected: `"Workflow contains a cycle: A → B → A. Cyclic graphs cannot be executed..."`

**Test B — Three-node cycle (A → B → C → A) should be REJECTED:**
```bash
curl -s -X POST http://localhost:3001/api/flows/execute \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "graph": {
      "nodes": [
        {"id":"A","type":"input","data":{"value":"1"}},
        {"id":"B","type":"action","data":{"operation":"multiply","factor":2}},
        {"id":"C","type":"action","data":{"operation":"add","factor":1}}
      ],
      "edges": [
        {"id":"e1","source":"A","target":"B"},
        {"id":"e2","source":"B","target":"C"},
        {"id":"e3","source":"C","target":"A"}
      ]
    },
    "input": {}
  }' | jq '.message'
```
Expected: `"Workflow contains a cycle: A → B → C → A. ..."`

**Test C — Linear graph (Input → Action → Output) should be ACCEPTED:**
```bash
curl -s -X POST http://localhost:3001/api/flows/execute \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "graph": {
      "nodes": [
        {"id":"n1","type":"input","data":{"value":"5"}},
        {"id":"n2","type":"action","data":{"operation":"multiply","factor":2}},
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
Expected: `"success"`

**Test D — Disconnected graph with a cycle in one component:**
```bash
# Graph: isolated node X, plus A → B → A cycle
# Both components should be checked; the cycle is still detected.
```
(Handled by the outer `for (const id of nodeIds)` loop that visits all WHITE nodes.)

### Unit test (optional)
```bash
cd backend && npm test -- --testPathPattern=flow-validator
```

---

## Expected Results

| Input | Expected HTTP status | Expected message |
|-------|---------------------|-----------------|
| Cyclic graph | `400 Bad Request` | `"Workflow contains a cycle: X → Y → X..."` |
| Self-loop | `400 Bad Request` | `"Self-referencing edge on node..."` |
| Valid DAG | `201 Created` / execution result | — |
| Disconnected DAG | Valid (no rejection) | — |

---

## Side Effects / Follow-up Notes

- The `validate()` method is called in **two** places: `FlowsService.create/update`
  (on save) AND `FlowExecutorService.execute` (on run). Both are now protected.
- The adjacency map construction moved inside `validate()` (was previously just
  edge checks). This adds O(V+E) memory but at typical workflow scale (< 200 nodes)
  this is negligible.
- Recursive DFS is used for clarity. For pathological graphs (> 1000 nodes), an
  iterative version with an explicit stack would be safer against call-stack overflow,
  but that is not a concern for this platform's workflow sizes.
- TASK 5 will remove the legacy `'api'` type from `validTypes`. That is the only
  other change needed in this file.
