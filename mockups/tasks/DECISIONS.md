# DailyDispatch v2 — Locked Design Decisions

## Status: All open questions resolved. Ready to build.

---

## Navigation

| Decision | Resolution |
|---|---|
| Sidebar | Left only. No right sidebar. |
| Primary nav | Today, Tasks, Search — top of sidebar |
| Today view | Combined: tasks due today/overdue + journal entries (the Feed). Today IS the Feed. |
| Feed | Does not exist as a separate nav item. Today replaces it. |
| Tab bar | Preserved. Notes open as tabs. Today is the permanent leftmost tab. + creates a new note tab. |
| Quick Open | Cmd+Shift+O — modal overlay, same as current. |

## Sidebar Structure (all collapsible)

| Section | Behavior |
|---|---|
| Calendar | Collapsible. Click date → navigates Today to that date. |
| Notes | Collapsible. Reverse-chron list. Click → opens note in a tab. |
| Sections | Collapsible. Click a section → expands inline showing matching entries. Click an entry in expanded section → opens in a tab. Can add/edit/delete sections. |
| Tags | Collapsible. Compact cloud. Click → filters. |
| Footer icons | Floating note, Gallery, Settings — always visible at bottom. |

## Calendar Behavior

| Click | What happens |
|---|---|
| Past date | Shows notes **edited** that day (not just created). No tasks. |
| Today | Shows tasks due today/overdue + today's journal. The default view. |
| Future date | Shows tasks due that day. Typing in capture creates a pre-dated note for that day. |
| Day arrows (◀ ▶) | Same behavior as calendar click — navigate day by day. |

## Tasks

| Decision | Resolution |
|---|---|
| Task checking | **Atomic.** Checking anywhere (Today, note embed, Kanban, list) syncs everywhere instantly. |
| Priority indicator | **Stripe** (left edge color bar on cards/rows). |
| Card click (Kanban) | Opens a **modal** detail view. |
| Column colors | **Customizable** per column. |
| Quick add | **Per column.** Each column has its own "+ Add task" button. |
| Done cap (Kanban) | **10 items.** Link to "View all completed" opens a filtered list view tab. |

## List View

| Decision | Resolution |
|---|---|
| Grouping | **Toggleable** — by column, priority, or due date. |
| Sorting | **Click column headers** to sort. |
| Done visibility | **Toggle** to show/hide completed tasks. |
| Inline editing | **Yes.** Click title to edit inline. |
| Drag between groups | **Yes.** Drag a row to a different group to change its column/priority. |

## Sections

| Decision | Resolution |
|---|---|
| Location | Left sidebar, collapsible group. |
| Click section | **Expands inline** in sidebar, showing matching entries below the section name. |
| Click entry in section | Opens in a tab (Today view if it's a journal entry, note tab if it's a note). |
| Collapse section | Hides the expanded entries. Same collapse pattern as Calendar/Notes/Tags. |
| Add/edit/delete | Same guided setup card pattern as current DailyDispatch. |

## Preserved from Current DailyDispatch

Everything that works today carries forward:
- Tab-based note editing with CM6 live preview
- Input Rail quick capture (Cmd+Enter)
- Slash commands, templates, status badges, tables
- Markdown editing (bold, italic, code, headings, lists, checkboxes, links)
- Floating meeting note popup
- Image paste/drop + gallery + resize
- Search tabs (Cmd+K) + Quick Open (Cmd+Shift+O)
- Themes (9 themes)
- Changelog, backup/restore, export/import
- Backlinks panel
- Events log
- @mentions, #tags, [[crosslinks]]

---

## Mockup Files

| File | What it shows |
|---|---|
| `sidebar.html` | Initial sidebar proposal (superseded by v2) |
| `task-tab-kanban.html` | Kanban board — stripe priority, per-column add, Done pinned right |
| `task-tab-list.html` | List view — grouped by column, sortable, inline edit |
| `today-with-tabs.html` | v1 hybrid: Today + tabs (superseded by v2) |
| `today-with-tabs-v2.html` | **Final design**: collapsible sidebar, Today=Feed, tabs preserved |
| `DECISIONS.md` | This file — all resolved decisions |
