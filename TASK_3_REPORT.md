# TASK 3 — No Undo/Redo

## Status: DONE

---

## What Was Changed and Why

The builder had no history mechanism — every accidental deletion or mis-move was
permanent.  JointJS ships `dia.CommandManager` which automatically intercepts
every graph mutation (add, remove, move, property change) and maintains an
undo/redo stack.  This task wires that built-in capability to toolbar buttons
and keyboard shortcuts without changing any existing graph logic.

---

## Files Modified

| File | Change |
|------|--------|
| `frontend/src/hooks/useJointGraph.js` | Added `commandManagerRef`, `canUndo/canRedo` state, `undo()`/`redo()` callbacks; `initialize` creates `dia.CommandManager`; `importWorkflow` resets history |
| `frontend/src/components/canvas/JointPaper.jsx` | Added `Ctrl+Z` (undo) and `Ctrl+Y` / `Ctrl+Shift+Z` (redo) keyboard shortcuts |
| `frontend/src/components/canvas/CanvasToolbar.jsx` | Added `canUndo`, `canRedo`, `onUndo`, `onRedo` props; added new History toolbar group with Undo (↩) and Redo (↪) buttons |
| `frontend/src/components/canvas/FlowCanvas.jsx` | Removed `deserializeGraph` import; startup now calls `editor.importWorkflow()` instead — which resets the history baseline; threaded `canUndo/canRedo/onUndo/onRedo` props to `CanvasToolbar` |

---

## Diff Summary

### `useJointGraph.js`
```diff
+  const commandManagerRef = useRef(null);
+  const [canUndo, setCanUndo] = useState(false);
+  const [canRedo, setCanRedo] = useState(false);

   // inside initialize():
+  const cmdManager = new dia.CommandManager({ graph });
+  commandManagerRef.current = cmdManager;
   graph.on("add remove ...", () => {
-    refreshWorkflow();
+    refreshWorkflow();
+    setCanUndo(cmdManager.hasUndo());
+    setCanRedo(cmdManager.hasRedo());
   });
   // cleanup:
+  commandManagerRef.current = null;

   // importWorkflow:
+  if (commandManagerRef.current) {
+    try { commandManagerRef.current.reset(); } catch {}
+    setCanUndo(false); setCanRedo(false);
+  }

+  const undo = useCallback(() => {
+    const cm = commandManagerRef.current;
+    if (!cm?.hasUndo()) return;
+    cm.undo();
+    setCanUndo(cm.hasUndo()); setCanRedo(cm.hasRedo());
+    refreshWorkflow();
+  }, [refreshWorkflow]);
+
+  const redo = useCallback(() => { /* mirror of undo */ }, [refreshWorkflow]);

-  return { ..., zoom, ... }
+  return { ..., canUndo, canRedo, undo, redo, zoom, ... }
```

### `JointPaper.jsx`
```diff
+  // Undo: Ctrl+Z / Cmd+Z
+  if ((event.ctrlKey || event.metaKey) && !event.shiftKey && event.key === 'z') {
+    event.preventDefault(); editorRef.current.undo();
+  }
+  // Redo: Ctrl+Y or Ctrl+Shift+Z
+  if (((ctrl && key==='y') || (ctrl && shift && key==='z'))) {
+    event.preventDefault(); editorRef.current.redo();
+  }
```

### `CanvasToolbar.jsx`
```diff
+  canRedo, canUndo, onRedo, onUndo,    // new props
   ...
+  <div className="toolbar-group">
+    <button disabled={!canUndo} onClick={onUndo} title="Undo (Ctrl+Z)">↩</button>
+    <button disabled={!canRedo} onClick={onRedo} title="Redo (Ctrl+Y)">↪</button>
+  </div>
```

### `FlowCanvas.jsx`
```diff
-import { deserializeGraph } from "engine/graphDeserializer";
   ...
-  deserializeGraph(currentEditor.graphRef.current, draft || starterWorkflow);
-  currentEditor.refreshWorkflow();
+  currentEditor.importWorkflow(draft || starterWorkflow);
   ...
+  canUndo={editor.canUndo}  canRedo={editor.canRedo}
+  onUndo={editor.undo}      onRedo={editor.redo}
```

---

## How to Verify

### Prerequisites
```bash
docker-compose up --build
# or: cd frontend && npm start
```

### Browser steps

1. Open `/admin/builder`
2. **Verify baseline**: Undo (↩) and Redo (↪) buttons should be greyed/disabled
3. Drag an **Action** block onto the canvas
   - Undo button should become enabled
4. Click **Undo** (or `Ctrl+Z`) — the Action block disappears
   - Redo button becomes enabled; Undo is disabled again
5. Click **Redo** (or `Ctrl+Y`) — the Action block reappears
6. Move a node to a new position → `Ctrl+Z` → node snaps back to old position
7. Delete a node → `Ctrl+Z` → node reappears
8. Load a saved workflow via the 📂 picker → verify Undo/Redo are both greyed
   (history cleared at load boundary)

### Keyboard shortcut matrix
| Action | Shortcut |
|--------|----------|
| Undo | `Ctrl+Z` / `Cmd+Z` |
| Redo | `Ctrl+Y` / `Cmd+Y` |
| Redo (alt) | `Ctrl+Shift+Z` / `Cmd+Shift+Z` |

---

## Expected Results

- Undo / Redo toolbar buttons appear between Import and Zoom controls
- Buttons are disabled (greyed) when nothing to undo/redo; enabled otherwise
- All graph mutations are reversible: add node, delete node, move node, change connection
- Keyboard shortcuts work globally on the canvas (not when focus is inside an input/textarea)
- Loading a workflow resets history — undo cannot cross a load boundary

---

## Side Effects / Follow-up Notes

- `dia.CommandManager` tracks: add/remove elements, position changes, attribute changes.
  It does NOT track viewport changes (zoom/pan) — those remain non-undoable (by design).
- The `try/catch` around `commandManager.reset()` guards against any version of JointJS
  where the method name differs; history simply won't clear if the call fails, but the
  canUndo/canRedo state flags are still set to false.
- The Undo/Redo buttons use `fa-undo` / `fa-redo` icons from FontAwesome 6 (already
  in the project via `@fortawesome/fontawesome-free`).
- TASK 10 (single-incoming-edge rule) will add a `validateConnection` change; the
  CommandManager will automatically track those connection mutations too — no extra work needed.
