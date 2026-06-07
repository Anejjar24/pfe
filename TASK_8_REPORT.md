# TASK 8 — Fit-to-Screen Button

## Status: DONE

---

## What Was Changed and Why

The canvas had no way to automatically frame all nodes.  If a user zoomed in or
panned far from the origin, finding the workflow again required manual zoom-out and
scrolling.  The "Reset view" button (⊞) returns to 1:1 scale at the origin, but it
does not scale the graph to fill the canvas.

The fix adds a **Fit to screen** action that:
1. calls `paper.scaleContentToFit()` — a built-in JointJS v4 method that finds the
   smallest scale/translate that shows all elements within the current paper viewport,
2. syncs the React `zoom` state from the paper's actual scale so the toolbar readout
   stays correct, and
3. is reachable both from a new toolbar button and from the keyboard shortcut
   `Ctrl+Shift+F`.

A full minimap/navigator was evaluated but is only available in `@joint/plus` (the
paid add-on); it is deferred rather than reinventing it with a DOM overlay.

---

## Files Modified

| File | Change |
|------|--------|
| `frontend/src/hooks/useJointGraph.js` | Added `fitToScreen` callback; exported it |
| `frontend/src/components/canvas/JointPaper.jsx` | Added `Ctrl+Shift+F` keyboard shortcut |
| `frontend/src/components/canvas/CanvasToolbar.jsx` | Added `onFit` prop + new toolbar button |
| `frontend/src/components/canvas/FlowCanvas.jsx` | Passed `onFit={editor.fitToScreen}` |

---

## Diff Summary

### `useJointGraph.js`
```diff
+  const fitToScreen = useCallback(() => {
+    const paper = paperRef.current;
+    if (!paper) return;
+    paper.scaleContentToFit({
+      padding: 40,           // breathing room around outermost nodes
+      minScale: 0.2,         // never shrink below 20 %
+      maxScale: 2.0,         // never expand beyond 200 %
+      useModelGeometry: false,
+    });
+    // Sync the React zoom readout to the scale JointJS chose.
+    setZoom(paper.scale().sx);
+  }, []);
+
   return {
     ...
+    fitToScreen,
   };
```

### `JointPaper.jsx`
```diff
+      // Fit to screen: Ctrl+Shift+F / Cmd+Shift+F
+      if ((event.ctrlKey || event.metaKey) && event.shiftKey && event.key.toLowerCase() === "f") {
+        event.preventDefault();
+        editorRef.current.fitToScreen();
+      }
```

### `CanvasToolbar.jsx`
```diff
 export default function CanvasToolbar({
   ...
+  onFit,
   ...
 }) {
   ...
+        <button onClick={onFit} title="Fit to screen (Ctrl+Shift+F)" type="button">
+          <i className="fa fa-expand-arrows-alt" aria-hidden="true" />
+        </button>
         <button onClick={onReset} title="Reset view (1:1)" type="button">
```

### `FlowCanvas.jsx`
```diff
       <CanvasToolbar
         ...
+        onFit={editor.fitToScreen}
         ...
       />
```

---

## How to Verify

### Button in toolbar
1. Open `/admin/builder`
2. Zoom in to 150 % and pan so the starter workflow is off-screen
3. Click the **⤢** (expand-arrows-alt) button in the zoom group of the canvas toolbar
4. All three starter nodes should snap into view, centred, with ~40 px padding on each side
5. The zoom readout should update to reflect the new scale (e.g. `72 %`)

### Keyboard shortcut
1. Click anywhere on the canvas (so focus is on the window, not an input)
2. Press `Ctrl+Shift+F` (or `Cmd+Shift+F` on macOS)
3. Same result as clicking the button

### Boundary cases
- **Empty canvas** (all nodes deleted) — `scaleContentToFit` is a no-op when there are
  no elements; the call completes without error
- **Single node** — the node is centred with 40 px padding; zoom readout updates
- **Zoom readout** — after fit, the readout value matches what JointJS chose (not stale)
- **Reset view** button — still works independently, returning to 100 % at origin

### Existing reset view still works
The ⊞ button and `Ctrl+Shift+Z`/`Ctrl+Y` shortcuts are unaffected.

---

## Expected Results

| Action | Before | After |
|--------|--------|-------|
| Nodes panned off-screen | Manual zoom-out + scroll required | Single click / `Ctrl+Shift+F` centres everything |
| After zoom-in and fit | Zoom readout shows stale value | Readout updates to match actual JointJS scale |
| Empty canvas | N/A | No error, fit is a no-op |

---

## Side Effects / Follow-up Notes

- **`useWorkflowEditor` unchanged** — it spreads `...graph`, so `fitToScreen` is
  automatically available on the `editor` object consumed by every component.
- **Minimap deferred** — `@joint/plus` Navigator would provide a live minimap overlay.
  If the free build gains an official minimap utility, it can be added here by mounting
  a second `dia.Paper` (read-only) in a corner overlay div.
- Phase 2 starts with **TASK 9 — `data-transform` node type**.
