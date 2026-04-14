---
name: project-todo-html
description: >
  This skill should be used ONLY when the user explicitly asks to "generate
  todo", "create a todo", "master todo", "task tracker", "build me a tracker",
  or "what's left to build" as a FILE for a specific named project. Outputs a
  JSON todo, a PRD markdown file, a live-updating HTML tracker with a PRD/todo
  toggle, and (optionally) injects a short auto-update rule into the project's
  CLAUDE.md. Do NOT trigger for general planning questions or casual
  "what should I do?"
version: 2
argument-hint: "[project name]"
---

# Project Tracker Generator

Generates a four-file project tracking system:
- `[slug]-todo.json` — machine-readable task list; agents read and write this
- `[slug]-prd.md` — product requirements document; agents update as decisions are made
- `[slug]-data.js` — generated view layer: `window.TRACKER_DATA = { todo, prd }`
- `[slug]-tracker.html` — static HTML shell that loads data.js and auto-refreshes every 5s

The JSON and PRD are the living sources of truth. Agents edit them directly.
When either changes, the skill regenerates `data.js` (cheap — ~5KB). The HTML
shell is written once and never touched again — it auto-refreshes every 5 seconds
and picks up fresh data on each tick.

---

## Four-File Architecture

### Why four files? (the rationale)

The v1 architecture embedded JSON + PRD directly in the HTML. Every agent update
required rewriting the ~30KB HTML file — expensive in tokens, slow, and it
coupled the data layer to the visual layer. The v2 split means:

- **Agents rewrite only `data.js` (~5KB).** 5-6x fewer tokens per update.
- **The HTML shell is stable.** Write once, review once, never rewritten.
- **The user opens the HTML once** and it auto-refreshes every 5 seconds. Each
  refresh fetches the latest `data.js` via a cache-busting query string.
- **Clear boundaries:** sources of truth (JSON/MD) → generated view layer
  (data.js) → presentation (HTML shell). Each file has one job.
- **Works on `file://`.** `<script src="./[slug]-data.js">` works across all
  modern browsers. No server required, no CORS issues, no infra.

This split is the central architectural change in v2. Every later section
assumes it.

### The four files

Agents interact with the JSON and PRD. Humans interact with the HTML. The
`data.js` layer in between is the bridge that makes updates cheap.

**Todo JSON** (`[slug]-todo.json`): Task tracking source of truth. Contains all
items, done/not-done status, badges, sections, timestamps, and `schema_version: 2`.
Small (~2-5KB). Agents edit this directly.

**PRD** (`[slug]-prd.md`): Product requirements document. Contains vision, users,
features, tech stack, architecture decisions, and a session handoff block. Small
(~3-8KB). Agents edit this directly as requirements evolve.

**Data file** (`[slug]-data.js`): Generated JavaScript file that exposes the
merged todo + PRD state as `window.TRACKER_DATA = { todo, prd }`. Small (~5-10KB).
Regenerated whenever the JSON or PRD changes — this is the only file that agent
updates rewrite. The HTML shell loads this via a cache-busting `<script>` tag on
every meta-refresh tick.

**HTML shell** (`[slug]-tracker.html`): Static shell written once at generation
time. Contains the visual layout, CSS, toggle bar, filters, and a meta refresh
tag that reloads the page every 5 seconds. Loads `[slug]-data.js` via an inline
cache-busting script. Large (~25-30KB) but **only written once** — not touched
by agent updates.

---

## JSON Schema

```json
{
  "schema_version": 2,
  "project": "CampusNova",
  "slug": "campusnova",
  "generated": "2026-04-14",
  "last_updated": "2026-04-14T14:30:00Z",
  "data_last_generated": "2026-04-14T14:30:00Z",
  "pending_updates": 0,
  "badges": [
    {
      "name": "SECURITY",
      "class": "b-1",
      "color": "#ff6b6b",
      "tooltip": "Auth, permissions, exposure"
    }
  ],
  "sections": [
    {
      "id": "critical",
      "label": "Critical / Must Do Now",
      "emoji": "🔴",
      "color": "#ff4444",
      "items": [
        {
          "id": "item-0",
          "text": "Task description",
          "note": "Optional sub-note with context",
          "badge": "SECURITY",
          "badge_class": "b-1",
          "done": false,
          "created": "2026-03-29",
          "updated": "2026-03-29"
        }
      ]
    }
  ]
}
```

Section IDs are always: `critical`, `active`, `polish`, `longterm`

---

## PRD Format

The PRD is a markdown file with this structure:

```markdown
# [Project Name] — Product Requirements Document

> **Session Handoff** (read this first):
> [5-line summary of current state, what was just completed, what's next,
> any blockers, and key decisions made. Updated by agents after each session.]

## Vision
[One paragraph — what this product does and why it matters]

## Target Users
[Who uses this and in what context — be specific]

## Core Features (Current State)
[What exists and works today — bullet list with status indicators]

## Planned Features
[What's coming — derived from todo items, grouped by priority]

## Tech Stack
[Frontend, backend, database, AI/external services, deployment]

## Architecture Decisions
[Key decisions made and WHY — agents append here as decisions happen]
- [date]: [decision] — [rationale]

## Current Status
[Phase: MVP / Beta / Production / etc.]
[Last updated: date]
[Open blockers: list or "none"]
```

---

## Agent Instructions — Updating the JSON

The JSON is the ground truth for project state. Agents read it when they need to
understand where things stand, and write it when they need to mark items done,
add new items, or reprioritize. Never read the HTML or `data.js` to infer state
— those are generated views, not sources of truth.

**When an agent completes a task and wants to mark it done:**
1. Read the JSON
2. Find the item by `id` or by matching `text`
3. Set `"done": true` and update `"updated"` to today's date
4. Increment `pending_updates` by 1
5. Update `last_updated` to current ISO timestamp
6. Write the JSON back to disk

**When an agent wants to add a new item:**
1. Read the JSON
2. Determine which section it belongs in
3. Assign the next available `item-N` id (find highest existing N, add 1)
4. Assign the appropriate badge from the `badges` array
5. Append to the correct section's `items` array
6. Increment `pending_updates` by 1
7. Update `last_updated` and write back

**When an agent wants to reprioritize (move item between sections):**
1. Read the JSON
2. Remove item from current section, add to target section
3. Increment `pending_updates` by 1, update timestamps, write back

When `pending_updates` reaches 5 or more, inform the user:
> "I've made [N] changes to the project files since the HTML was last generated.
> Say 'sync' when you want to see the updated tracker."

Only regenerate the HTML if the user explicitly asks.

---

## Agent Instructions — Updating the PRD

When an agent makes a significant decision or discovers new requirements:

1. Read `[slug]-prd.md`
2. Update the relevant section:
   - New feature discovered → add to "Planned Features"
   - Architecture decision made → append to "Architecture Decisions" with date
   - Feature completed → move from "Planned" to "Core Features (Current State)"
   - Blocker found → update "Current Status"
3. ALWAYS update the "Session Handoff" block at the top with current state
4. Write the PRD back to disk

The Session Handoff block is the most important part — it's what new agents read
first to understand where things stand.

---

## Environment Detection

In Claude Code (filesystem available):
- JSON: `./[slug]-todo.json`
- PRD: `./[slug]-prd.md`
- HTML: `./[slug]-tracker.html`
- Tell the user: "Your project tracker is ready. Open `[slug]-tracker.html` in
  your browser. Toggle between Todo and PRD views at the top. Say 'sync' anytime
  to refresh the HTML with the latest data."

In Claude.ai (sandbox environment):
- JSON: `/mnt/user-data/outputs/[slug]-todo.json`
- PRD: `/mnt/user-data/outputs/[slug]-prd.md`
- HTML: `/mnt/user-data/outputs/[slug]-tracker.html`
- Call `present_files` with all three paths

Deriving the slug: lowercase, hyphens, no special characters.

---

## Badge System — Dynamic Generation

Technical projects use the Standard Dev Badge Set (SECURITY, PERF, AI, DATA,
ARCH, UX, TEST, BIZ). Non-technical projects get contextually generated badges.

**Technical projects** (code files, APIs, databases, frameworks detected):
Use the Standard Dev Badge Set:

| Badge | Class | Color | Tooltip |
|---|---|---|---|
| `SECURITY` | `b-1` | `#ff6b6b` | Auth, permissions, exposure |
| `PERF` | `b-2` | `#ffb74d` | Speed, queries, rendering |
| `AI` | `b-3` | `#4dd0e1` | Models, prompts, pipelines |
| `DATA` | `b-4` | `#81c784` | Integrity, schema, parsing |
| `ARCH` | `b-5` | `#ef9a9a` | Infra, systems, deployment |
| `UX` | `b-6` | `#ce93d8` | User-facing visuals & flows |
| `TEST` | `b-7` | `#9fa8da` | QA, coverage, automation |
| `BIZ` | `b-8` | `#ffd54f` | GTM, pricing, partnerships |

**Non-technical projects** (personal, creative, business, life):
Generate 6-8 contextually appropriate badges. Examples:

Personal/life goals: CAREER, HEALTH, MONEY, MINDSET, BUILD, SHIP, PEOPLE, LEARN
Content creation: WRITE, RECORD, EDIT, GROW, COLLAB, SYSTEMS
Startup/business: PRODUCT, GROWTH, OPS, TEAM, LEGAL, FINANCE

Use the color palette: `#ff6b6b #ffb74d #4dd0e1 #81c784 #ef9a9a #ce93d8 #9fa8da #ffd54f`

---

## Content Generation

If memory context or conversation history exists: use it to populate both the
todo items AND the PRD sections.

If no context is available: ask the user 3 questions before generating:
1. What is the project and what does it do?
2. What is the tech stack or domain?
3. What are the 2-3 most pressing things right now?

Generate the PRD first, then derive todo items from it. This ensures alignment
between the roadmap (PRD) and the task list (JSON).

Target item counts:
- Critical: 8-12
- Active: 10-15
- Polish: 10-15
- Long-term: 10-15

---

## HTML Design Spec

Read `references/html-design-spec.md` for the complete visual design specification.
The key v2 changes are summarized below.

### Meta refresh + cache-busted data.js

Every HTML shell includes in `<head>`:

```html
<meta http-equiv="refresh" content="5">
```

This reloads the full page every 5 seconds. To ensure each reload fetches fresh
`data.js`, the HTML shell does NOT hardcode a `<script src="./[slug]-data.js">`
tag. Instead, it includes a tiny inline script at the top of `<body>` that
injects the tag with a cache-busting query string:

```html
<script>
  const s = document.createElement('script');
  s.src = './[slug]-data.js?t=' + Date.now();
  document.body.appendChild(s);
  s.onload = () => window.dispatchEvent(new Event('tracker-data-loaded'));
</script>
```

The rest of the HTML waits for the `tracker-data-loaded` event before rendering:
`window.addEventListener('tracker-data-loaded', renderTracker)`.

### localStorage state preservation

To avoid losing UX context on every 5-second reload, save the following to
`localStorage` on `beforeunload` and restore on load:

- `scrollY` — page scroll position
- `activeView` — "todo" or "prd"
- `activeFilter` — section id currently filtered ("all", "critical", "active", etc.)
- `showCompleted` — boolean
- `collapsedSections` — array of section ids the user collapsed
- `prdCollapsedBlocks` — PRD sections the user collapsed

Use a single `localStorage` key per tracker: `tracker-state-[slug]` storing a
JSON blob with the six fields above.

### View toggle (unchanged from v1)

- Two pill buttons at top: "Todo" | "PRD"
- Active = filled with accent color (#ff6b2b), white text
- Inactive = transparent, muted text, border
- Default active: Todo
- Filter bar only shows in Todo view
- PRD view renders the embedded PRD markdown as styled HTML
- `activeView` in `localStorage` overrides the default on load

### Show completed counter (new in v2)

Next to the "Show completed" toggle, display a live counter: `Show completed (23)`.
The number is derived client-side at render time:

```javascript
const completedCount = window.TRACKER_DATA.todo.sections
  .flatMap(s => s.items)
  .filter(i => i.done).length;
```

No schema change — the count is computed from existing `done` flags.

---

## Sync Behavior

When the user says "sync", "update the todo", "update the prd", or "refresh tracker":
1. Read the current `[slug]-todo.json` and `[slug]-prd.md`
2. Regenerate `[slug]-data.js` with both datasets embedded as
   `window.TRACKER_DATA = { todo, prd }`
3. Update two fields in `[slug]-todo.json`: reset `pending_updates` to 0
   and set `data_last_generated` to the current ISO timestamp
4. Confirm to the user: "Data refreshed. Tracker will pick it up within 5
   seconds (the next meta refresh tick)."

**The HTML shell is NEVER regenerated on sync.** Sync rewrites `[slug]-data.js`
(full regen) plus two counter fields in `[slug]-todo.json` (small update). The
HTML shell is only regenerated when the visual design itself changes — rare and
intentional (e.g., a new skill version). This is what makes agent updates cheap:
the ~30KB HTML shell stays frozen, only small JSON field edits + a ~5KB data.js
regen per sync.

Note: agents that edit the JSON for other reasons (marking items done, adding
new items) should also increment `pending_updates` by 1 as part of their
write — see "Agent Instructions — Updating the JSON" above.

Updating EITHER the todo JSON or the PRD markdown triggers a regen of `data.js`.

---

## CLAUDE.md Auto-Update Injection

After generating the tracker files, offer the user the option to inject a short
auto-update rule into the project's `CLAUDE.md`. This is the mechanism that
makes the tracker feel "live" across sessions — agents in this project
automatically keep the todo and PRD in sync with their work.

Use `AskUserQuestion` with a single question:

**Question:** "Want me to add a short rule to this project's `CLAUDE.md` so
future agents automatically update the tracker when they complete tasks or
make decisions?"

Options:
- `Yes, add the rule (Recommended)` — inject the rule immediately
- `Show me the rule first` — print the rule text, then re-ask
- `No, I'll handle updates manually` — skip injection, print the rule anyway so
  the user can paste it later

### The rule text

If the user approves, append this block to the project's `CLAUDE.md` (create
the file if missing):

```markdown
## Project Tracker (auto-managed)

This project has a tracker at `[slug]-tracker.html` backed by `[slug]-todo.json`
and `[slug]-prd.md`. When you:
- Complete a task → announce it, then set `done: true` in the JSON
- Make an architecture decision → announce it, then append to the PRD's
  "Architecture Decisions" section with today's date and rationale
- Pivot on scope → announce it, then update the affected todo items
- Ship a planned feature → announce it, then move it from "Planned Features"
  to "Core Features (Current State)"

**Always announce the update before applying it** so the user can veto.
Do not touch `[slug]-data.js` or `[slug]-tracker.html` — they regenerate
automatically from the JSON and PRD. The user says "sync" to force-regenerate
`[slug]-data.js`.
```

### Safety: announce before applying

The "announce before applying" rule is the safety net. It means the user sees
every update in the transcript and can stop it with "no wait" before it lands.
No silent mutations. This is critical — without it, users won't trust the
auto-update mechanism.

### When to skip injection without asking

If the project's `CLAUDE.md` already contains a "Project Tracker (auto-managed)"
section (from a previous run of this skill), do not prompt — just print:

> "This project already has the tracker rule in CLAUDE.md. Skipping injection."

---

## Output Instructions

1. Determine badge set for this project type
2. Generate the PRD first (`[slug]-prd.md`)
3. Generate the todo JSON derived from the PRD (`[slug]-todo.json`) with
   `schema_version: 2` and `pending_updates: 0`
4. Generate `[slug]-data.js` with `window.TRACKER_DATA = { todo, prd }`
5. Generate the HTML shell (`[slug]-tracker.html`) with meta refresh, cache-bust
   inline script, and localStorage state logic
6. Detect environment and use correct paths (Claude Code filesystem vs Claude.ai
   sandbox)
7. Offer the CLAUDE.md auto-update injection (see prior section) unless the
   rule already exists
8. Print the clickable `file://` URL for the tracker HTML (absolute path)
9. Confirm all four file paths to the user
10. State item counts per section

### Clickable file:// URL format

After generation, print:

```
✓ Tracker ready. Cmd-click to open in your browser:
  file:///absolute/path/to/[slug]-tracker.html

  The tracker auto-refreshes every 5 seconds. Leave the tab open — any agent
  updates to [slug]-todo.json or [slug]-prd.md will appear automatically.
  Say "sync" anytime to force-regenerate [slug]-data.js.
```

Modern terminals (iTerm2, Warp, Terminal.app, Alacritty, WezTerm, Ghostty)
render `file://` URLs as clickable. Use the absolute path, not a relative path.

---

## Generation Checklist

Run through this checklist after generating all files and before printing the
final output to the user:

- [ ] PRD generated before todo items (ensures alignment)
- [ ] PRD has all required sections including Session Handoff
- [ ] JSON has `schema_version: 2`, matches schema, `pending_updates` starts at 0
- [ ] `[slug]-data.js` written with `window.TRACKER_DATA = { todo, prd }` shape
- [ ] HTML shell has `<meta http-equiv="refresh" content="5">` in `<head>`
- [ ] HTML shell has the inline cache-bust script injecting `data.js?t=Date.now()`
- [ ] HTML has the view toggle bar (Todo / PRD pill buttons)
- [ ] HTML defaults to Todo view
- [ ] Filter bar hidden in PRD view
- [ ] "Show completed" toggle includes a live-computed counter: `Show completed (N)`
- [ ] localStorage restore logic wired to `tracker-state-[slug]` key
- [ ] All four files written to correct paths for the current environment
- [ ] CLAUDE.md injection offered (or skipped because rule already present)
- [ ] Clickable `file://` URL printed with absolute path
- [ ] HTML opens in browser with no console errors
