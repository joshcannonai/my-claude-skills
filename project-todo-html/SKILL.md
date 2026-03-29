---
name: project-todo-html
description: >
  Use ONLY when the user explicitly asks to generate, create, or build a
  to-do list, task tracker, issue tracker, checklist, or project tracker
  as a FILE or HTML document for a specific named project. Trigger phrases:
  "generate todo", "create a todo", "master todo", "task tracker", "issue
  tracker", "project checklist", "what's left to build", "make a todo html",
  "build me a tracker". Output is always a JSON file (source of truth) and an
  HTML file (human-readable view). Do NOT trigger for general questions about
  what to work on next, sprint planning discussions, casual "what should I do?"
  questions, or requests that don't explicitly ask for a file/document output.
license: MIT
compatibility: Works in Claude Code (writes to project directory) and Claude.ai
  (writes to sandbox outputs). No sub-agents required. Google Fonts is the only
  external dependency in the generated HTML file.
metadata:
  author: joshcannonai
  version: "2.0"
---

# Project Master To-Do HTML Generator

Generates a two-file task tracking system for any project:
- [slug]-todo.json -- machine-readable source of truth; agents read and write this
- [slug]-todo.html -- human-readable interactive tracker; generated from the JSON

Agents update the JSON. The HTML is only regenerated when the user asks, or when
the skill detects enough changes have accumulated to make regeneration worthwhile.

---

## File Architecture

Agents interact with the JSON. Humans interact with the HTML.

The JSON is the authoritative state. It contains all items, their done/not-done
status, badge assignments, section membership, and a change counter that tracks
how many updates have happened since the HTML was last regenerated.

The HTML reads nothing from disk -- all data is embedded at generation time. When
an agent updates the JSON and the user opens the HTML, they see the old state.
When the HTML is regenerated from the updated JSON, they see the new state.

This split means:
- Agents can update the todo list mid-session without touching the HTML
- The HTML never breaks because of an agent's half-written update
- State is preserved across regenerations via localStorage in the browser

---

## JSON Schema

{
  "project": "CampusNova",
  "slug": "campusnova",
  "generated": "2026-03-29",
  "last_updated": "2026-03-29T14:30:00Z",
  "html_last_generated": "2026-03-29T14:30:00Z",
  "changes_since_html": 0,
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

Section IDs are always: critical, active, polish, longterm

---

## Agent Instructions -- Reading and Writing the JSON

When another agent needs to understand project state:
  cat [slug]-todo.json
The JSON is the ground truth. Do not read the HTML to understand project state.

When an agent completes a task and wants to mark it done:
1. Read the JSON
2. Find the item by id or by matching text
3. Set "done": true and update "updated" to today's date
4. Increment changes_since_html by 1
5. Update last_updated to current ISO timestamp
6. Write the JSON back to disk

When an agent wants to add a new item:
1. Read the JSON
2. Determine which section it belongs in
3. Assign the next available item-N id (find highest existing N, add 1)
4. Assign the appropriate badge from the badges array
5. Append to the correct section's items array
6. Increment changes_since_html by 1
7. Update last_updated and write back

When an agent wants to reprioritize (move item between sections):
1. Read the JSON
2. Remove item from current section, add to target section
3. Increment changes_since_html by 1, update timestamps, write back

When changes_since_html reaches 5 or more, ask the user:
  "I've made [N] changes to the todo list since the HTML was last generated.
  Want me to regenerate it now so you can see the updated view?"

Only regenerate the HTML if the user says yes, or if they explicitly ask.

---

## Environment Detection

In Claude Code (filesystem available):
- JSON: write to ./[slug]-todo.json
- HTML: write to ./[slug]-todo.html
- Tell the user: "Open [slug]-todo.html in your browser. [slug]-todo.json is
  the source of truth -- agents update this file to track progress."

In Claude.ai (sandbox environment):
- JSON: write to /mnt/user-data/outputs/[slug]-todo.json
- HTML: write to /mnt/user-data/outputs/[slug]-todo.html
- Call present_files with both paths

Deriving the slug: lowercase, hyphens, no special characters.
CampusNova becomes campusnova, My Portfolio becomes my-portfolio.

---

## Badge System -- Dynamic Generation

Determine the badge set before writing any items.

Technical projects (code files, APIs, databases, frameworks detected):
Use the Standard Dev Badge Set:

| Badge | Class | Color | Tooltip |
|---|---|---|---|
| SECURITY | b-1 | #ff6b6b | Auth, permissions, exposure |
| PERF | b-2 | #ffb74d | Speed, queries, rendering |
| AI | b-3 | #4dd0e1 | Models, prompts, pipelines |
| DATA | b-4 | #81c784 | Integrity, schema, parsing |
| ARCH | b-5 | #ef9a9a | Infra, systems, deployment |
| UX | b-6 | #ce93d8 | User-facing visuals and flows |
| TEST | b-7 | #9fa8da | QA, coverage, automation |
| BIZ | b-8 | #ffd54f | GTM, pricing, partnerships |

Non-technical projects (personal, creative, business, life):
Generate 6-8 contextually appropriate badges. Examples:
- Personal/life goals: CAREER, HEALTH, MONEY, MINDSET, BUILD, SHIP, PEOPLE, LEARN
- Content creation: WRITE, RECORD, EDIT, GROW, COLLAB, SYSTEMS
- Startup/business: PRODUCT, GROWTH, OPS, TEAM, LEGAL, FINANCE

Use the color palette: #ff6b6b #ffb74d #4dd0e1 #81c784 #ef9a9a #ce93d8 #9fa8da #ffd54f

---

## Content Generation

If memory context or conversation history exists: use it -- architecture, known
bugs, planned features, team members, tech stack, past decisions, current sprint.

If no context is available: ask the user 3 questions before generating:
1. What is the project and what does it do?
2. What is the tech stack or domain?
3. What are the 2-3 most pressing things right now?

Target item counts:
- Critical: 8-12 (blockers, security, crash risks, demo blockers)
- Active: 10-15 (current sprint, core feature completion)
- Polish: 10-15 (UX, performance, accessibility, empty states)
- Long-term: 10-15 (architecture evolution, future features)

Add a note to any item where the fix is non-obvious, involves specific risk,
or references a named team member or tool.

---

## HTML Design Spec

When generating the HTML, embed all JSON data as a JS constant at the top of the
script block. The HTML renders entirely from this embedded data -- no file fetch.

  const TODO_DATA = { /* paste full JSON here */ };

The rest of the JS reads from TODO_DATA rather than hardcoded HTML elements.
When regenerating, replace the TODO_DATA constant with the updated JSON.

Visual Design:
- Background: #0d0d0f, surfaces #141416 / #1a1a1d
- Fonts: IBM Plex Mono + IBM Plex Sans via Google Fonts (only external dependency)
- Accent: #ff6b2b for project title, active states, completed count
- Full-width layout, no max-width centering, padding 36px 48px on main

Header (sticky, two rows):
Row 1: Project name in orange IBM Plex Mono, subtitle, generation date
Row 2: Stats bar with three cards (Total / Completed / Remaining), gradient
progress bar, and white percentage label

Stats bar: font-size 26px for numbers, color #e8e8ec for percentage label,
gradient linear-gradient(90deg, #ff6b2b, #ffaa00) for the fill bar

Filter Bar (sticky below header, top offset set dynamically via JS):
Left to right: All / Critical / Active / Polish / Future | Hide Completed |
Search input (200 to 260px on focus, orange border when active) | Tags dropdown
(multi-select popover, data-tag uses class slug e.g. b-1) | Sync button | Copy MD

Items:
- Click toggles .done class: strikethrough + 0.42 opacity + faded badge
- Each badge has data-tip tooltip (3-6 words) shown on hover
- data-id="item-N" assigned by JS on init for localStorage mapping

Section progress bars:
3px bar below each section header, fills as items complete, color matches
section accent, animated with CSS transition

localStorage:
Key: [slug]-todo-state
Saves: { done: ["item-0",...], hideDone: false, activeFilter: "all", activeTags: [] }
Restores fully on page load before first render

Sync button:
Calls location.reload(). Since localStorage persists checkbox state, reloading
always restores the user's progress even when the agent has rewritten the HTML.

---

## Output Instructions

1. Determine badge set for this project type
2. Generate all four sections with items, notes, and badges
3. Write [slug]-todo.json to disk first
4. Generate HTML from the JSON data, embed JSON as TODO_DATA constant
5. Write [slug]-todo.html to disk
6. Detect environment and use correct paths
7. Confirm both file paths to the user
8. State item counts per section

---

## Quality Checklist

- JSON written before HTML
- JSON is valid and matches the schema, changes_since_html starts at 0
- Badge set chosen before writing items (dev set or custom)
- All badges have class, color, tooltip in the JSON badges array
- HTML embeds full JSON as TODO_DATA constant in script block
- Header has both rows: title row AND stats bar
- Stats bar: Total (white), Completed (orange), Remaining (muted), 26px numbers
- Percentage label is #e8e8ec (white, not muted grey)
- Filter bar top set dynamically via JS (not hardcoded px)
- Tag pills use data-tag="b-1" style (class slug, not badge name)
- Sync button present, calls location.reload()
- Section progress bars present under each header
- localStorage key uses project slug
- Blue section label reads "Long-term Features / Future"
- Both files written to correct paths for current environment
- HTML opens in browser with no console errors
