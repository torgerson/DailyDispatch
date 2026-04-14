# DailyDispatch URI API Specification

## Problem Statement

DailyDispatch is a single-file HTML app with all data in IndexedDB. External tools (Claude Cowork, scripts, automation) have no way to query or mutate this data. We need an API that works without a server, without installing anything, and within the constraints of a local HTML file.

## Solution: Hash-Based URI API

The app detects `#api/` in the URL hash on load. Instead of rendering the normal UI, it executes the command against IndexedDB and renders a JSON response into the page body. External tools navigate to the URL, read the response, done.

---

## URI Format

```
{base_url}#api/{version}/{resource}[/{id}][/{action}][?{params}]
```

### Components

| Part | Required | Description |
|------|----------|-------------|
| `base_url` | Yes | Path to index.html (`file:///...` or `http://localhost:...`) |
| `#api` | Yes | API mode trigger — app skips normal UI rendering |
| `/{version}` | Yes | API version, always `v1` for now |
| `/{resource}` | Yes | Data type: `entries`, `tasks`, `sections`, `columns`, `tags`, `people`, `images`, `search`, `stats` |
| `/{id}` | No | Specific resource ID (e.g., `t_1234567890_abc12`) |
| `/{action}` | No | Verb: `create`, `update`, `complete`, `delete` |
| `?{params}` | No | Query parameters for filtering, pagination, or mutation data |

### Parameter Encoding

Query params use standard URL encoding after `?`, separated by `&`:
```
?key=value&key2=value2
```

Since this is a hash-based scheme, the `?` is part of the hash fragment (not a real query string). The app parses it from `window.location.hash`.

---

## Response Format

All responses are JSON rendered into a `<pre id="api-response">` element:

```json
{
  "ok": true,
  "resource": "tasks",
  "action": "list",
  "count": 12,
  "data": [ ... ],
  "ts": 1744502400000
}
```

### Error Response

```json
{
  "ok": false,
  "error": "Task not found",
  "code": "NOT_FOUND",
  "resource": "tasks",
  "action": "get",
  "ts": 1744502400000
}
```

### Error Codes

| Code | Meaning |
|------|---------|
| `NOT_FOUND` | Resource with given ID doesn't exist |
| `INVALID_PARAMS` | Missing or malformed parameters |
| `INVALID_RESOURCE` | Unknown resource type |
| `INVALID_ACTION` | Action not supported for this resource |
| `STORAGE_ERROR` | IndexedDB read/write failed |

---

## Resources & Endpoints

### 1. Entries

**List all entries**
```
#api/v1/entries
#api/v1/entries?type=note           → only notes (isNote=true)
#api/v1/entries?type=quick          → only quick entries (isNote=false)
#api/v1/entries?type=event          → only events (eventType set)
#api/v1/entries?date=2026-04-13     → entries from specific date
#api/v1/entries?date=today          → entries from today
#api/v1/entries?tag=meeting         → entries with #meeting tag
#api/v1/entries?mention=sarah       → entries mentioning @sarah
#api/v1/entries?limit=20&offset=0   → pagination (default limit=50)
#api/v1/entries?sort=newest         → sort: newest (default), oldest
```

**Get single entry**
```
#api/v1/entries/{id}
```

**Search entries** (uses existing matchEntries engine)
```
#api/v1/search?q=#meeting @sarah -draft
#api/v1/search?q="action item"&limit=10
```

**Create entry**
```
#api/v1/entries/create?content=Standup notes from today #meeting @dave&type=quick
#api/v1/entries/create?content=...&type=note&title=Sprint Notes
```

**Update entry**
```
#api/v1/entries/{id}/update?content=Updated content here
#api/v1/entries/{id}/update?tags=meeting,sprint
```

**Delete entry**
```
#api/v1/entries/{id}/delete
```

### Entry Object (returned)

```json
{
  "id": "e_1744502400000_abc12",
  "content": "Standup notes #meeting",
  "tags": ["meeting"],
  "mentions": ["dave"],
  "date": "2026-04-13",
  "ts": 1744502400000,
  "isNote": true,
  "noteTitle": "Sprint Notes",
  "updatedTs": 1744506000000,
  "editDates": [{"date": "2026-04-13", "ts": 1744506000000}]
}
```

---

### 2. Tasks

**List tasks**
```
#api/v1/tasks
#api/v1/tasks?status=open           → only incomplete tasks (default)
#api/v1/tasks?status=completed      → only completed
#api/v1/tasks?status=all            → both
#api/v1/tasks?priority=H            → filter by priority (H, M, L)
#api/v1/tasks?column=col_inbox      → filter by column ID
#api/v1/tasks?due=today             → due today
#api/v1/tasks?due=overdue           → overdue
#api/v1/tasks?due=7d                → due within 7 days
#api/v1/tasks?due=none              → no due date
#api/v1/tasks?today=true            → starred for Today
#api/v1/tasks?tag=sprint            → tasks with #sprint tag
#api/v1/tasks?sort=priority         → sort: priority (default), due, created, completed
```

**Get single task**
```
#api/v1/tasks/{id}
```

**Create task**
```
#api/v1/tasks/create?title=Fix the login bug&priority=H
#api/v1/tasks/create?title=Write docs&priority=M&due=2026-04-20&column=col_backlog
```

**Update task**
```
#api/v1/tasks/{id}/update?title=Updated title
#api/v1/tasks/{id}/update?priority=H&due=2026-04-15
#api/v1/tasks/{id}/update?column=col_inwork
#api/v1/tasks/{id}/update?description=More details here
#api/v1/tasks/{id}/update?today=true
```

**Complete/uncomplete task**
```
#api/v1/tasks/{id}/complete         → marks as done
#api/v1/tasks/{id}/uncomplete       → marks as open
```

**Delete task**
```
#api/v1/tasks/{id}/delete
```

**Add comment**
```
#api/v1/tasks/{id}/comment?text=Started working on this
```

### Task Object (returned)

```json
{
  "id": "t_1744502400000_xyz99",
  "title": "Fix the login bug",
  "priority": "H",
  "due_date": "2026-04-15",
  "column_id": "col_inwork",
  "completed": false,
  "completed_at": null,
  "note_id": "e_1744500000000_abc12",
  "created_at": 1744502400000,
  "order": 2,
  "today": true,
  "description": "Users reporting 500 error on /login",
  "comments": [
    {"id": "tc_1744503000000_qw3rt", "text": "Reproduced locally", "ts": 1744503000000}
  ],
  "tags": ["sprint", "bug"],
  "mentions": ["alex"]
}
```

---

### 3. Sections

**List sections**
```
#api/v1/sections
```

**Get section with results**
```
#api/v1/sections/{id}              → section metadata + matched entries/tasks
```

**Create section**
```
#api/v1/sections/create?name=Meetings&query=#meeting
```

**Update section**
```
#api/v1/sections/{id}/update?name=New Name
#api/v1/sections/{id}/update?query=#meeting -draft
```

**Delete section**
```
#api/v1/sections/{id}/delete
```

### Section Object (returned)

```json
{
  "id": "sec_1744502400000_m3k9p",
  "name": "Meetings",
  "query": "#meeting",
  "height": 220,
  "matchCount": 12
}
```

---

### 4. Columns

**List columns**
```
#api/v1/columns
```

**Create column**
```
#api/v1/columns/create?name=Review&color=#f59e0b
```

**Update column**
```
#api/v1/columns/{id}/update?name=New Name&color=#3b82f6
```

**Delete column**
```
#api/v1/columns/{id}/delete
```

### Column Object (returned)

```json
{
  "id": "col_inbox",
  "name": "Inbox",
  "color": "#e20074",
  "order": 0,
  "isDone": false,
  "taskCount": 5
}
```

---

### 5. Tags

**List all tags with counts**
```
#api/v1/tags
```

**Get entries by tag**
```
#api/v1/tags/{tagname}             → entries + tasks with this tag
```

### Tags Response

```json
{
  "ok": true,
  "data": [
    {"tag": "meeting", "count": 24, "color": "#3b82f6"},
    {"tag": "shipped", "count": 12, "color": "#10b981"}
  ]
}
```

---

### 6. People

**List all people with mention counts**
```
#api/v1/people
```

**Get entries mentioning a person**
```
#api/v1/people/{name}              → entries + tasks mentioning @name
```

### People Response

```json
{
  "ok": true,
  "data": [
    {"name": "Alex", "count": 15},
    {"name": "Sarah", "count": 9}
  ]
}
```

---

### 7. Images (metadata only)

**List image metadata**
```
#api/v1/images
#api/v1/images?limit=20
```

**Get image metadata**
```
#api/v1/images/{id}
```

Note: Image data (base64) is excluded from list responses for performance. Use `#api/v1/images/{id}?include=data` to get the full data URI.

### Image Object (returned)

```json
{
  "id": "img_1744502400000_f9x2r",
  "mimeType": "image/png",
  "ts": 1744502400000,
  "size": 45231
}
```

---

### 8. Stats

**Get aggregate statistics**
```
#api/v1/stats
#api/v1/stats?date=2026-04-13      → stats for a specific date
#api/v1/stats?week=2026-04-07      → stats for the week starting Mon Apr 7
```

### Stats Response

```json
{
  "ok": true,
  "data": {
    "totalEntries": 342,
    "totalNotes": 48,
    "totalTasks": 67,
    "openTasks": 23,
    "completedTasks": 44,
    "totalSections": 5,
    "totalTags": 18,
    "totalPeople": 12,
    "totalImages": 8,
    "theme": "Hot Pink",
    "version": "3.7.0",
    "lastBackup": 1744416000000
  }
}
```

---

### 9. Search (unified)

**Search across entries and tasks**
```
#api/v1/search?q=#meeting
#api/v1/search?q=@sarah "action item" -draft
#api/v1/search?q=date:today
#api/v1/search?q=completed:today
#api/v1/search?type=entries         → entries only
#api/v1/search?type=tasks           → tasks only
#api/v1/search?limit=20
```

### Search Response

```json
{
  "ok": true,
  "query": "#meeting @sarah",
  "entries": [ ... ],
  "tasks": [ ... ],
  "totalEntries": 8,
  "totalTasks": 3
}
```

---

## Pagination

All list endpoints support pagination:

| Param | Default | Description |
|-------|---------|-------------|
| `limit` | 50 | Max items returned (max 200) |
| `offset` | 0 | Skip first N items |

Response includes pagination metadata:

```json
{
  "ok": true,
  "count": 50,
  "total": 342,
  "offset": 0,
  "limit": 50,
  "hasMore": true,
  "data": [ ... ]
}
```

---

## Sorting

| Resource | Sort Options | Default |
|----------|-------------|---------|
| entries | `newest`, `oldest` | `newest` |
| tasks | `priority`, `due`, `created`, `completed` | `priority` |
| tags | `count`, `alpha` | `count` |
| people | `count`, `alpha` | `count` |

---

## Implementation Notes

### How the App Detects API Mode

On load, check `window.location.hash`:

```javascript
if(window.location.hash.startsWith('#api/')){
  initApiMode();
  return; // Skip normal UI init
}
```

### API Mode Rendering

In API mode, the app:
1. Opens IndexedDB and loads all data (same `loadDB()` path)
2. Parses the hash into `{version, resource, id, action, params}`
3. Executes the command
4. Renders JSON into `<pre id="api-response">`
5. Sets `document.title` to `DD-API: {resource}/{action}`
6. Does NOT render sidebar, feed, editor, or any UI

### How Cowork Reads the Response

```
1. navigate to URL with #api/... hash
2. wait for page load
3. read <pre id="api-response"> text content
4. parse JSON
```

Or via Chrome MCP JavaScript:
```javascript
document.getElementById('api-response').textContent
```

### Write Operations

Write operations (create, update, delete, complete) modify `db` and call `save()` to persist to IDB. The response confirms the mutation:

```json
{
  "ok": true,
  "action": "create",
  "resource": "tasks",
  "data": { ... created object ... }
}
```

### URL Length Limits

Browser URL bars support ~2000 characters. For long content (note bodies, descriptions), the `content` param will be URL-encoded. If content exceeds ~1500 chars, the caller should use multiple update calls or a different approach.

---

## Security Considerations

- **Same-origin only**: IndexedDB is sandboxed per origin. The API only works from the same `file://` or `localhost` origin.
- **No authentication**: This is a local-only, single-user tool. No auth needed.
- **No network**: All data stays local. No HTTP requests are made.
- **Destructive operations**: Delete operations are permanent (same as UI). Consider adding `?confirm=true` requirement for deletes.

---

## Future Extensions (v2)

- `#api/v1/export` — trigger JSON export download
- `#api/v1/import?url=...` — import from a file path
- `#api/v1/summary/daily?date=today` — generate daily summary
- `#api/v1/summary/weekly` — generate weekly review
- Batch operations: `#api/v1/tasks/batch/complete?ids=t_1,t_2,t_3`
- WebSocket-like polling: `#api/v1/watch?resource=entries&since=1744502400000`
