# DailyDispatch — Product Purpose

## One-line pitch

**A private, local-first work journal for tech-forward knowledge workers who want their thoughts indexed, searchable, and never shipped to someone else's server.**

## Who this is for

- **Engineers, PMs, designers, researchers, consultants, founders** — people whose work is "thinking and typing" and whose output lives in a stream of half-formed notes, meeting snippets, decisions, links, and open questions.
- **Power users** who already live in a terminal, a text editor, and keyboard shortcuts. They know markdown. They probably use (or considered) Obsidian, Logseq, Bear, Tana, or Roam.
- **Privacy-conscious people** who bristle at "sign in to sync" and don't want a third party's terms of service governing their daily journal.
- **Tinkerers** who see a single HTML file and think "oh, I could run that anywhere, inspect it, back it up however I want."

## Who this is NOT for

- **Apple Notes / Google Keep users.** They want zero-friction capture that "just syncs" and don't want to think about where their data lives. If they see a filter query, section setup, or slash command, they bounce. Do not design for them.
- **Notion / Confluence teams.** This isn't collaborative. It's a personal journal. Team workspaces are explicitly out of scope.
- **Todo-app-first users (Things, TickTick, Todoist).** Checkboxes are a feature here, not the point. If someone wants a GTD system, point them at a GTD app.
- **People who need mobile as the primary surface.** Desktop-first, period. A mobile companion may come later but it's not a driver.

## Why we build this

1. **The cloud PKM market is crowded and cloud-dependent.** Notion, Roam, Mem, Tana, Obsidian Sync — all require an account, most require the internet, and all give someone else access to the most intimate data you have: your thinking. We don't want that deal and we believe there's an audience that doesn't either.
2. **Obsidian is the closest reference** but Obsidian is a folder of files — powerful, flexible, and also demanding. It asks the user to design their own system. DailyDispatch has opinions: a **feed**, a **capture rail**, **sections as saved filters**, **events as a metacognitive trail**. We hand the user a working system on day one.
3. **A single HTML file is a statement.** No install. No build step. No dependencies. Open it in any browser and it works. That constraint is a feature — it keeps the surface area small and the user in control.
4. **Meeting notes while on a video call** is a real, recurring pain. The floating note window is built specifically for that workflow and is one of the strongest differentiators.

## Positioning against the likely alternatives

| Competitor | Their strength | Our edge |
|---|---|---|
| **Obsidian** | File-based, plugins, huge community | No install, no config, opinionated defaults, feed-first UX |
| **Notion** | Collaborative, rich blocks, cloud | Local-only, no account, no lock-in, single file |
| **Bear** | Beautiful, Apple-only | Cross-browser, no Mac tax, open data |
| **Roam/Logseq** | Outliner + graph | We're a feed, not an outline — better fit for work journaling |
| **Apple Notes** | Frictionless, ubiquitous | Power features, better search, structured views |
| **Day One** | Journaling focus, beautiful | Work-oriented (tags, mentions, sections), tech-friendly markdown |

## Design principles (guidance for future decisions)

1. **Capture is sacred.** Anything that slows down getting a thought out of a head and into the feed is a regression. Keyboard-driven, no dialogs, no confirmations.
2. **Local-first, always.** No feature should require the network. No feature should leak data across the network by default. File System Access API for sync is OK (user picks a folder). Real-time cloud sync is not.
3. **Power-user-friendly, not power-user-only.** A new user should get useful default sections, a short tour, and a working feed on first launch. A power user should have keyboard shortcuts, slash commands, query operators, and raw markdown.
4. **Opinions over options.** When we add a setting, it's because a meaningful fraction of users need it. We cut settings aggressively. Three capture modes and nine themes was two too many — we'll prune.
5. **Markdown is the source of truth for text.** Tables are a structured exception (stored as objects), but everything else should round-trip through markdown cleanly. If you can't copy-paste a note into another markdown tool, we've done something wrong.
6. **The feed is the center.** Notes, sections, pins, events — everything hangs off the feed. It is the product's point of view. Don't bury it.
7. **Every feature earns its keep.** Features that show up in the data model but nowhere in the UI (old Questions, Contexts) are debt. Ship or rip.
8. **Discoverability beats cleverness.** If a new user has to read help to find basic interactions (rename a section, save a note), the design has failed. Guided setup > hidden affordance.
9. **Copy works like a pro tool, not a polite consumer app.** No emojis in chrome unless they carry information. No animations that don't serve comprehension. Warm Notion palette, typography that doesn't shout.
10. **The floating note is the hero screenshot.** When in doubt about a design trade-off, ask: does this strengthen the "take notes while on a video call" story?

## Explicit non-goals

- Real-time collaboration, shared workspaces, comments, mentions that notify other users
- Cloud sync with authentication / accounts / paywall
- AI "chat with your notes" as a primary feature (simple semantic search is fine; a chatbot is not)
- Mobile-first design
- Rich-media authoring (video, audio recording, drawing canvas)
- Plugin ecosystem — if we can't ship it in the single file, it doesn't belong
- Enterprise features (SSO, audit logs, admin panels)

## How to use this document

When in doubt about adding a feature, cutting one, or shaping a design decision, re-read this. If a proposed change violates one of the principles or serves a non-target audience, push back. If a feature doesn't move the hero story forward, it's a lower priority by default.

**Taste is cheap to assert and expensive to ignore. Keep this ruthless.**
