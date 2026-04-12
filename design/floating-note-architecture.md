# Floating Note Window — Architecture Redesign

## Feature Intent (User Story)

As a user on a video call, I want to click one button, get a small note window pinned to my screen edge, type meeting notes, and when I close that window, my notes appear as a tab in the main DailyDispatch window — saved, titled, and visible in my feed. **Zero data loss. Zero user action beyond close.**

## The Problem

The current implementation has a fundamental architectural flaw: it relies on a chain of async operations (IndexedDB writes, BroadcastChannel messages, localStorage events, focus handlers) across two independent browser windows that share an IndexedDB but have separate in-memory state. Any break in that chain — popup closing too fast, IDB flush not completing, storage event not firing, focus not triggering — causes data loss or empty notes.

## Root Cause Analysis

### Why data is lost:
1. **Race condition on create**: Main window creates entry + calls `save()` (debounced 100ms). Popup opens immediately, calls `loadDB()`, may get stale IDB without the entry.
2. **Race condition on close**: Popup's `beforeunload` calls `flushSave()` (async, returns Promise) but the handler is synchronous — browser tears down before IDB write completes.
3. **Tab never opens**: Main window relies on `storage` event or `focus` event to detect popup close. Both are unreliable: `storage` events may not fire during window teardown, `focus` may not fire if main window was already focused.

### Why patches keep failing:
Each fix addresses one race condition but introduces another. The architecture is fundamentally fragile because it treats two independent windows as a distributed system without proper coordination primitives.

## Design Principles for the Rebuild

1. **localStorage is the source of truth during floating mode** — not IndexedDB. localStorage is synchronous, survives window teardown, and is shared across same-origin windows instantly.
2. **The popup writes to localStorage on every change** — not IDB. This eliminates the async flush problem entirely.
3. **The main window polls localStorage** — not events. A simple `setInterval` is more reliable than `storage` events + `focus` events + `visibilitychange` events combined.
4. **IDB is only written on close** — the popup attempts an IDB flush on close as a bonus, but localStorage is the primary store.
5. **The main window reads from localStorage first, IDB second** — when reopening the note, check localStorage backup before IDB.

## New Architecture

### Data Flow

```
USER CLICKS "FLOATING NOTE"
  ↓
Main window:
  1. Generate noteId
  2. Write empty entry to localStorage: dd_float_note = {id, content:'', ts, ...}
  3. Open popup window with ?floating=1#noteId (synchronous, in user gesture)
  4. Push empty entry to db.entries + save() (background, for feed display)

POPUP WINDOW OPENS
  ↓
Popup:
  1. Detect floating mode from URL
  2. Read noteId from hash
  3. Read entry from localStorage (dd_float_note) — NOT from IDB
  4. If not in localStorage, read from IDB as fallback
  5. Render CM6 editor with entry content

USER TYPES
  ↓
Popup (on every change):
  1. Update in-memory entry
  2. Write FULL entry to localStorage: dd_float_note = {id, content, title, tags, ts}
  3. (Do NOT write to IDB on every keystroke — too slow, unnecessary)

USER CLOSES POPUP
  ↓
Popup (beforeunload, synchronous):
  1. Write final entry to localStorage: dd_float_note (already current from step above)
  2. Write close signal: dd_float_closed = {noteId, ts}
  3. Attempt IDB flush (async, may not complete — that's OK)

MAIN WINDOW DETECTS CLOSE
  ↓
Main window (via 1-second poll on dd_float_closed):
  1. Read dd_float_note from localStorage
  2. Update db.entries with full content from localStorage
  3. Save to IDB (now the main window does the IDB write, which it can await)
  4. Remove dd_float_note and dd_float_closed from localStorage
  5. Open the note as a tab
  6. Re-render feed (note now shows with content and title)
```

### Key Differences from Current Approach

| Aspect | Current (Broken) | New (Proposed) |
|--------|-----------------|----------------|
| **Where content lives during editing** | In-memory + IDB flush attempts | localStorage (synchronous) |
| **How popup saves** | `save()` + `flushSave()` (async IDB) | `localStorage.setItem()` (sync) |
| **How main window detects close** | storage event + focus event + visibilitychange | 1-second setInterval poll |
| **Who writes to IDB** | Popup (may fail on teardown) | Main window (after reading localStorage) |
| **BroadcastChannel needed?** | Yes (for live sync during editing) | No (localStorage is already shared) |
| **Entry in IDB before popup opens?** | Required (race condition source) | Not required (popup reads localStorage) |

### localStorage Keys

- `dd_float_note` — full entry object: `{id, content, noteTitle, tags, ts, updatedTs, date}`
- `dd_float_closed` — close signal: `{noteId, ts}`

Both are cleaned up by the main window after successful recovery.

### Functions to Rewrite

1. **`openFloatingNote()`** — create entry, write to localStorage, open popup
2. **`openFloatingNoteFor(id)`** — write existing entry to localStorage, open popup
3. **`initFloatingMode()`** — read from localStorage (not IDB), set up editor, save to localStorage on change
4. **`noteContentChanged()` floating path** — write to localStorage only (no IDB)
5. **`onClose` handler** — write close signal to localStorage (IDB flush is bonus)
6. **Main window poll** — setInterval checks `dd_float_closed`, reads `dd_float_note`, updates db, opens tab

### Functions to Remove

- All `BroadcastChannel` code for floating sync (unnecessary with localStorage)
- The `_ddReloadFromIDB` call in recovery path (main window has the data from localStorage)
- The complex backup/recovery logic in `_ddOpenFloatingClosedNote`

## Testing Requirements

All tests MUST run in actual Chrome with two windows, not simulated:

1. **Create + type + close → tab opens with content**
2. **Create + type rapidly + close immediately → content preserved**
3. **Create + type + close browser tab (not just window) → content preserved**
4. **Create + don't type + close → empty note cleaned up (no orphan)**
5. **Create floating note while another is open → handles gracefully**
6. **Main window reload while floating is open → floating keeps working**
7. **Close floating + immediately reopen main window → tab appears**

## Implementation Order

1. Strip all existing floating note cross-window code
2. Implement localStorage-based data flow
3. Test each step in Chrome
4. Remove debug logging after confirmed working
