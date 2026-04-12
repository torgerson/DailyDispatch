# Floating Window Note — Test Plan

## Test Setup
- App running at `http://localhost:8091/`
- Chrome browser with Claude in Chrome connected
- Main window open to DailyDispatch

## Test Cases

### T1: Floating window opens
1. Click the floating note icon in the tab bar
2. **Assert**: A popup window opens with URL containing `?floating=1#<noteId>`
3. **Assert**: The popup shows a note editor (CM6)
4. **Assert**: The note entry exists in `db.entries` in the main window

### T2: Typing in floating window saves content
1. Type text in the floating window: "Test floating note content #test"
2. Wait 500ms
3. **Assert**: `localStorage.getItem('dd_floating_backup')` contains the typed content
4. **Assert**: The note's content in main window's `db.entries` matches (via BroadcastChannel sync)

### T3: Closing floating window preserves content
1. Close the floating window (click X or navigate away)
2. **Assert**: `localStorage.getItem('dd_floating_closed')` was set with the noteId
3. **Assert**: `localStorage.getItem('dd_floating_backup')` contains the content

### T4: Main window opens the note as a tab after close
1. Focus the main window (triggers the focus event handler)
2. **Assert**: The note appears as a tab in the main window's tab bar
3. **Assert**: The note's content is visible in the editor
4. **Assert**: The note's content matches what was typed in the floating window

### T5: Note appears in the feed
1. Switch to the Today tab
2. **Assert**: The note appears in the feed with the correct title
3. **Assert**: The note shows the typed content in the preview

### T6: Note persists after reload
1. Reload the main window
2. **Assert**: The note is in `db.entries` with correct content
3. **Assert**: Opening the note shows the typed content

## How to Run
- Use `javascript_tool` for state assertions
- Use `computer` tool for clicking/typing in the popup
- Use `navigate` to open the floating window URL directly if `window.open` is blocked
- Take screenshots at each assertion point
