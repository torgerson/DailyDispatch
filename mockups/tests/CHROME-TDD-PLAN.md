# Chrome TDD Plan: Sections Drag-and-Drop

Target: `http://localhost:8091/` (single-file app served locally)
Tools: Claude in Chrome MCP (`computer`, `javascript_tool`, `navigate`)
Existing unit tests: `mockups/tests/sections-test.html` (data + CSS class tests only)

---

## 1. What to Test in Chrome

### 1.1 Core Drag Scenarios

| ID | Scenario | Setup | Action | Assertions |
|----|----------|-------|--------|------------|
| D1 | Drag section down (3+ sections) | 3 user sections [A, B, C] | Drag A below C | Order becomes [B, C, A] |
| D2 | Drag section up (3+ sections) | 3 user sections [A, B, C] | Drag C above A | Order becomes [C, A, B] |
| D3 | Drag to middle position | 4 sections [A, B, C, D] | Drag D above B | Order becomes [A, D, B, C] |
| D4 | Drag to empty space below all | 3 sections [A, B, C] | Drag A to empty space far below C | A moves to end: [B, C, A] |
| D5 | Drag Help section | Seeded sections [Help, Meetings] + user [A] | Drag Help below A | Order: [Meetings, A, Help] |
| D6 | Add section then immediately drag | 3 sections, click "+ Section", fill name | Drag new section to top | New section is first in order |
| D7 | Drag adjacent sections | [A, B, C] | Drag A below B (just one position) | Order: [B, A, C] |
| D8 | Drag and drop on self (no-op) | [A, B, C] | Drag A, release on A | Order unchanged: [A, B, C] |

### 1.2 Visual Integrity During Drag

| ID | Scenario | Assertion |
|----|----------|-----------|
| V1 | No sections disappear during drag | Every non-dragged `.rs-panel` has `offsetHeight > 0` |
| V2 | Dragged section collapses | The dragged `.rs-panel.sec-dragging` has `offsetHeight === 0` |
| V3 | Drop indicator appears | Exactly one panel has `sec-drop-above` or `sec-drop-below` class |
| V4 | Drop indicator is amber line | `::before` pseudo-element on target panel has `background` matching `--accent` |
| V5 | No layout thrashing | `sec-dragging` class stays on the dragged panel throughout the entire drag (never removed then re-added during dragover) |
| V6 | Inline styles don't block collapse | Panels with `style="height:260px;flex:none"` still collapse when `sec-dragging` is applied (the `!important` override works) |

### 1.3 Persistence

| ID | Scenario | Assertion |
|----|----------|-----------|
| P1 | Order persists after drop | After drag-drop, `db.sections.map(s=>s.id)` matches expected new order |
| P2 | Order persists after re-render | Call `renderAllSections()` after drop; DOM order matches `db.sections` |
| P3 | Order survives page reload | After drop, reload page; section order in DOM matches the order saved before reload |

### 1.4 Edge Cases

| ID | Scenario | Assertion |
|----|----------|-----------|
| E1 | Drag with only 1 section | Single section; drag has no target to drop on. Drop should be no-op. |
| E2 | Drag with 2 sections | Swap positions; verify both survive |
| E3 | Rapid successive drags | Drag A below C, then immediately drag B above A; both reorders applied correctly |
| E4 | Drag cancel (Escape key) | Start drag, press Escape; all classes cleared, order unchanged |

---

## 2. How to Test in Chrome

### 2.1 Overall Approach

Each test follows this lifecycle:

```
1. Navigate to http://localhost:8091/
2. Reset state (clear sections, seed test data via javascript_tool)
3. Capture pre-state (screenshot + DOM assertions)
4. Perform drag action (computer left_click_drag)
5. Capture post-state (screenshot + DOM assertions)
6. Report pass/fail
```

### 2.2 Tool Mapping

| Step | MCP Tool | Purpose |
|------|----------|---------|
| Navigate | `navigate` | Load the app at `http://localhost:8091/` |
| Setup test data | `javascript_tool` | Manipulate `db.sections`, call `save()`, `renderAllSections()` |
| Find coordinates | `javascript_tool` | `getBoundingClientRect()` on `.rs-panel[data-sid="..."]` to get drag source/target coords |
| Perform drag | `computer left_click_drag` | Real browser drag from source coords to target coords |
| Assert DOM state | `javascript_tool` | Query `offsetHeight`, class lists, `db.sections` order, `data-sid` attributes |
| Visual verification | `computer screenshot` | Capture before/during/after screenshots |
| Page reload test | `navigate` | Navigate to same URL again, then assert order |

### 2.3 Coordinate Calculation Strategy

The drag source and target coordinates must be computed dynamically because section panel positions depend on viewport size and panel heights.

```javascript
// Example: get center of a section panel for drag start
function getPanelCenter(sid) {
  const el = document.querySelector(`.rs-panel[data-sid="${sid}"]`);
  if (!el) return null;
  const r = el.getBoundingClientRect();
  return { x: r.left + r.width / 2, y: r.top + r.height / 2 };
}

// Example: get position just below a target panel (for "drop below")
function getDropBelowTarget(sid) {
  const el = document.querySelector(`.rs-panel[data-sid="${sid}"]`);
  if (!el) return null;
  const r = el.getBoundingClientRect();
  return { x: r.left + r.width / 2, y: r.bottom + 5 };
}

// Example: get position just above a target panel (for "drop above")
function getDropAboveTarget(sid) {
  const el = document.querySelector(`.rs-panel[data-sid="${sid}"]`);
  if (!el) return null;
  const r = el.getBoundingClientRect();
  return { x: r.left + r.width / 2, y: r.top - 5 };
}
```

Call these via `javascript_tool`, then feed the returned `{x, y}` into `computer left_click_drag`.

### 2.4 Drag Execution Pattern

For each drag test:

```
1. javascript_tool: const src = getPanelCenter('sec_a');
                    const dst = getDropBelowTarget('sec_c');
                    JSON.stringify({src, dst});

2. computer left_click_drag:
     start_coordinate: [src.x, src.y]
     coordinate: [dst.x, dst.y]
     tabId: <tab>

3. computer screenshot (capture result)

4. javascript_tool: // Assert final order
     const ids = [...document.querySelectorAll('#sb-sections-body .rs-panel')]
       .map(el => el.dataset.sid);
     const dbIds = db.sections.map(s => s.id);
     JSON.stringify({ domOrder: ids, dbOrder: dbIds });
```

### 2.5 Mid-Drag Assertions (V1-V6)

To check state *during* a drag (before drop), we need to split the drag into stages:

1. **mousedown + small move** -- Start the drag by doing `left_click_drag` from the source to a point only ~20px away (still over the source). This initiates the drag without completing it.

   *Problem:* `left_click_drag` is atomic -- it completes the drag. We cannot pause mid-drag with MCP tools.

   **Alternative approach for mid-drag assertions:** Use `javascript_tool` to synthetically dispatch drag events while asserting between them:

   ```javascript
   // Synthetic dragstart
   const srcEl = document.querySelector('.rs-panel[data-sid="sec_a"]');
   const startEvent = new DragEvent('dragstart', {
     bubbles: true, cancelable: true,
     dataTransfer: new DataTransfer()
   });
   srcEl.dispatchEvent(startEvent);
   ```

   Then wait (the setTimeout in dragSecStart applies sec-dragging after 0ms), then assert:

   ```javascript
   // After a microtask, check mid-drag state
   await new Promise(r => setTimeout(r, 50));
   const panels = document.querySelectorAll('#sb-sections-body .rs-panel');
   const results = [...panels].map(p => ({
     sid: p.dataset.sid,
     height: p.offsetHeight,
     isDragging: p.classList.contains('sec-dragging')
   }));
   JSON.stringify(results);
   ```

   **However:** Synthetic DragEvents have a read-only `dataTransfer` in most browsers, so `setData` may fail. This means synthetic drag events are unreliable for full testing.

   **Recommended hybrid approach:**
   - Use *real* `left_click_drag` for end-to-end drag-drop tests (D1-D8, P1-P3).
   - Use *javascript_tool* class manipulation for mid-drag visual assertions (V1-V6) -- this is what the existing `sections-test.html` does, but we run it inside the real app's DOM with the real CSS.
   - This hybrid catches both categories of bugs: data/event bugs (real drag) and CSS/layout bugs (class assertion in real app context).

### 2.6 Checking for Layout Thrashing (V5)

Use a MutationObserver to detect if `sec-dragging` is ever removed then re-added during a drag:

```javascript
// Install observer BEFORE starting drag
window._dragClassFlicker = 0;
const observer = new MutationObserver(mutations => {
  for (const m of mutations) {
    if (m.type === 'attributes' && m.attributeName === 'class') {
      const el = m.target;
      if (el.classList.contains('sec-dragging')) {
        // class was added (could be re-added after removal)
        window._dragClassFlicker++;
      }
    }
  }
});
document.querySelectorAll('.rs-panel').forEach(el => {
  observer.observe(el, { attributes: true, attributeFilter: ['class'] });
});
window._dragObserver = observer;
```

After the drag completes:

```javascript
window._dragObserver.disconnect();
// _dragClassFlicker should be exactly 1 (the initial add)
// If > 1, the class was removed and re-added (layout thrashing)
JSON.stringify({ flickerCount: window._dragClassFlicker });
```

---

## 3. Test Harness Structure

### 3.1 State Management Helper (injected via javascript_tool)

```javascript
// Inject once at start of test session
window.DnDTest = {
  // Clear all sections and reseed with named test sections
  setup(names) {
    db.sections = [];
    db.defaultSectionsSeeded = true; // prevent auto-seeding
    names.forEach((name, i) => {
      db.sections.push({
        id: 'sec_test_' + name.toLowerCase(),
        name: name,
        query: '#' + name.toLowerCase(),
        height: 200,
        col: 'right'
      });
    });
    save();
    renderAllSections();
    return db.sections.map(s => s.id);
  },

  // Setup with Help section included (for D5)
  setupWithHelp(userNames) {
    db.sections = [];
    db.defaultSectionsSeeded = false;
    ensureHelpSection(); // adds Help + Meetings
    userNames.forEach(name => {
      db.sections.push({
        id: 'sec_test_' + name.toLowerCase(),
        name: name,
        query: '#' + name.toLowerCase(),
        height: 200,
        col: 'right'
      });
    });
    save();
    renderAllSections();
    return db.sections.map(s => s.id);
  },

  // Get center coordinates for a section panel
  getCenter(sid) {
    const el = document.querySelector(`.rs-panel[data-sid="${sid}"]`);
    if (!el) return null;
    const r = el.getBoundingClientRect();
    return [Math.round(r.left + r.width / 2), Math.round(r.top + r.height / 2)];
  },

  // Get coordinate just below a panel's bottom edge
  getBelow(sid) {
    const el = document.querySelector(`.rs-panel[data-sid="${sid}"]`);
    if (!el) return null;
    const r = el.getBoundingClientRect();
    return [Math.round(r.left + r.width / 2), Math.round(r.bottom + 5)];
  },

  // Get coordinate just above a panel's top edge
  getAbove(sid) {
    const el = document.querySelector(`.rs-panel[data-sid="${sid}"]`);
    if (!el) return null;
    const r = el.getBoundingClientRect();
    return [Math.round(r.left + r.width / 2), Math.round(r.top - 5)];
  },

  // Assert section order matches expected IDs
  assertOrder(expectedIds) {
    const domIds = [...document.querySelectorAll('#sb-sections-body .rs-panel')]
      .map(el => el.dataset.sid);
    const dbIds = db.sections.map(s => s.id);
    const domMatch = JSON.stringify(domIds) === JSON.stringify(expectedIds);
    const dbMatch = JSON.stringify(dbIds) === JSON.stringify(expectedIds);
    return { pass: domMatch && dbMatch, domIds, dbIds, expectedIds };
  },

  // Assert no panels have zero height (except dragging ones)
  assertAllVisible() {
    const panels = document.querySelectorAll('#sb-sections-body .rs-panel:not(.sec-dragging)');
    const hidden = [...panels].filter(p => p.offsetHeight === 0);
    return {
      pass: hidden.length === 0,
      total: panels.length,
      hidden: hidden.map(p => p.dataset.sid)
    };
  },

  // Assert all drag classes are cleared
  assertClean() {
    const dragging = document.querySelectorAll('.rs-panel.sec-dragging').length;
    const above = document.querySelectorAll('.rs-panel.sec-drop-above').length;
    const below = document.querySelectorAll('.rs-panel.sec-drop-below').length;
    return { pass: dragging === 0 && above === 0 && below === 0, dragging, above, below };
  },

  // Get full state snapshot
  snapshot() {
    const panels = [...document.querySelectorAll('#sb-sections-body .rs-panel')];
    return panels.map(p => ({
      sid: p.dataset.sid,
      height: p.offsetHeight,
      classes: p.className,
      inlineHeight: p.style.height,
      inlineFlex: p.style.flex
    }));
  }
};
```

### 3.2 Test Execution Protocol

Each test is executed as a sequence of MCP tool calls. The pattern:

```
Step 1: javascript_tool  -- DnDTest.setup(['A','B','C'])
Step 2: computer screenshot  -- "before" visual
Step 3: javascript_tool  -- get source/target coords
Step 4: computer left_click_drag  -- perform the drag
Step 5: computer screenshot  -- "after" visual
Step 6: javascript_tool  -- DnDTest.assertOrder([...])
Step 7: javascript_tool  -- DnDTest.assertClean()
Step 8: javascript_tool  -- DnDTest.assertAllVisible()
```

### 3.3 Test Session Initialization

At the start of each test session:

```
1. tabs_context_mcp (get/create tab)
2. navigate to http://localhost:8091/
3. javascript_tool: inject DnDTest helper object
4. computer screenshot: verify app loaded
```

### 3.4 Reload Persistence Test (P3)

```
1. DnDTest.setup(['A','B','C'])
2. Drag A below C  (left_click_drag)
3. assertOrder(['sec_test_b','sec_test_c','sec_test_a'])
4. navigate to http://localhost:8091/  (reload)
5. Re-inject DnDTest helper
6. assertOrder(['sec_test_b','sec_test_c','sec_test_a'])
```

### 3.5 Results Tracking

Maintain a results array in the page for the session:

```javascript
window.DnDResults = [];
window.DnDTest.record = function(testId, description, result) {
  window.DnDResults.push({
    id: testId,
    desc: description,
    pass: result.pass,
    detail: result,
    time: new Date().toISOString()
  });
  return result;
};
window.DnDTest.summary = function() {
  const passed = window.DnDResults.filter(r => r.pass).length;
  const failed = window.DnDResults.filter(r => !r.pass).length;
  return { passed, failed, total: passed + failed, results: window.DnDResults };
};
```

---

## 4. Known Pitfalls

### 4.1 Inline Styles Override Class-Based Height

**Problem:** Each `.rs-panel` is rendered with `style="height:${sec.height||200}px;flex:none"` (line 6123 of index.html). When `sec-dragging` sets `height: 0`, the inline style wins unless `!important` is used.

**Current fix:** `.sec-dragging` uses `height: 0 !important; flex: none !important;` (line 1337).

**Test:** V6 must verify that a panel with an explicit inline height (e.g., `height:260px`) still collapses to 0 when `sec-dragging` is applied.

### 4.2 min-height Blocks Collapse

**Problem:** If `.rs-panel` has `min-height: 80px` (from general styling), then `height: 0` alone does nothing.

**Current fix:** `min-height: 0 !important` in `.sec-dragging`.

**Test:** V2 asserts `offsetHeight === 0` on the dragged panel, which will fail if min-height is not overridden.

### 4.3 Layout Thrashing from Class Removal

**Problem (was the main bug):** The old `_clearSecDropIndicators` function cleared *both* `sec-drop-above/below` AND `sec-dragging` on every `dragover` event. This caused the dragged panel to expand back to full height on every mouse move, then collapse again after the timeout in `dragSecStart` -- but `dragSecStart` only runs once, so the class was lost permanently. Other panels shifted and disappeared.

**Current fix:** Split into two functions:
- `_clearSecDropOnly()` -- clears only drop indicators. Called on every `dragover`.
- `_clearAllSecDrag()` -- clears everything. Called only on `dragend` and `drop`.

**Test:** V5 uses a MutationObserver to detect if `sec-dragging` is ever removed and re-added during a drag. The flicker count should be exactly 1 (the initial add in `dragSecStart`).

### 4.4 Container-Level Drop Handler

**Problem:** Dropping in empty space below all sections (between the last panel and the sidebar bottom) didn't trigger any panel's `ondrop`. The drop was silently lost.

**Current fix:** A `dragover`/`drop` listener on `#sb-sections-body` handles drops that don't hit a `.rs-panel`. It finds the nearest visible panel and uses the drop indicator, or appends to the end if no panels exist.

**Test:** D4 specifically drags to empty space below all sections. The container handler must catch it.

### 4.5 setTimeout in dragSecStart

**Problem:** `dragSecStart` applies `sec-dragging` via `setTimeout(fn, 0)`. This is intentional -- the browser needs the original element dimensions for the drag ghost image. But it means there's a 1-frame window where the panel is full-size after drag begins.

**Impact on testing:** After `left_click_drag` starts, the class is applied asynchronously. For mid-drag assertions via javascript, add a small delay (`await new Promise(r => setTimeout(r, 50))`) before checking.

### 4.6 Synthetic vs Real Drag Events

**Problem:** Synthetic `DragEvent` created via `new DragEvent(...)` in Chrome has a read-only `dataTransfer.items`. Calling `setData` may throw or silently fail. The app's `dragSecStart` calls `e.dataTransfer.setData('text/plain', secId)`, which works with real user drags but may fail with synthetic ones.

**Mitigation:** Use real `left_click_drag` via the `computer` tool for all end-to-end drag tests. Only use synthetic events (with the `javascript_tool`) for targeted mid-drag state checks where we manually call the app's internal functions (`dragSecId = 'sec_a'; ...`) rather than dispatching DOM events.

### 4.7 Scroll Position

**Problem:** If the sidebar has many sections and is scrolled, `getBoundingClientRect()` returns viewport-relative coords. The `left_click_drag` tool also uses viewport coords, so they should match. But if the sidebar scrolls *during* a drag (because panels collapse), the target coords computed before the drag may be wrong.

**Mitigation:** Keep test setups to 3-5 sections with modest heights (200px) so the sidebar doesn't need to scroll. For scroll tests, compute coords *after* the drag has started (not feasible with atomic `left_click_drag`), or accept that scroll-during-drag is a separate test category that may need the synthetic approach.

### 4.8 Drag Ghost Offset

**Problem:** `left_click_drag` moves the cursor from start to end. The browser positions the drop based on cursor position, not the ghost image center. The `dragSecOver` handler uses `e.clientY` to determine above/below relative to each panel's midpoint. If the cursor ends up at the exact midpoint of a panel, the above/below decision is ambiguous.

**Mitigation:** Target coordinates should be clearly above or below panel midpoints. Use `getAbove(sid)` (top - 5px) and `getBelow(sid)` (bottom + 5px) rather than panel centers for drop targets.

---

## 5. Test Execution Order

Run tests in this order (each group depends on the previous passing):

1. **Session setup** -- navigate, inject helpers
2. **V1-V6** -- visual integrity (class-based, in real app CSS context)
3. **D1** -- basic drag down (simplest real drag)
4. **D2** -- basic drag up
5. **D7** -- adjacent swap
6. **D8** -- self-drop no-op
7. **D3** -- drag to middle
8. **D4** -- drop in empty space
9. **D5** -- Help section drag
10. **D6** -- add-then-drag
11. **P1-P2** -- persistence after drop
12. **P3** -- persistence after reload
13. **E1-E4** -- edge cases

---

## 6. Success Criteria

All tests pass means:
- No section visually disappears during any drag operation
- Drop indicators appear at the correct position
- Section order updates correctly in both DOM and `db.sections`
- Order survives page reload (persisted to localStorage)
- Inline styles (`height:Npx;flex:none`) do not prevent drag collapse
- The `sec-dragging` class is never removed during `dragover`
- Container-level drops (empty space) work correctly
- Help and freshly-created sections are draggable like any other section
