---
name: project-todo-html
description: >
  This skill should be used ONLY when the user explicitly asks to "generate
  todo", "create a todo", "master todo", "task tracker", "build me a tracker",
  or "what's left to build" as a FILE for a specific named project. Outputs a
  JSON todo, a PRD markdown file, and an HTML tracker with toggle between views.
  Do NOT trigger for general planning questions or casual "what should I do?"
argument-hint: "[project name]"
---

# Project Tracker Generator

Generates a three-file project tracking system:
- [slug]-todo.json -- machine-readable task list; agents read and write this
- [slug]-prd.md -- product requirements document; agents update as decisions are made
- [slug]-tracker.html -- interactive tracker with toggle between Todo and PRD views

The JSON and PRD are living documents. Agents update them between sessions.
The HTML is a snapshot -- only regenerated when the user asks to "sync" or "update".
Updating either view triggers both to sync.

---

## Three-File Architecture

Agents interact with the JSON and PRD. Humans interact with the HTML.

Todo JSON ([slug]-todo.json): Task tracking source of truth. Contains all items,
done/not-done status, badges, sections, and a change counter.

PRD ([slug]-prd.md): Product requirements document. Contains vision, users,
features, tech stack, architecture decisions, and a session handoff block. Agents
update this as requirements evolve and decisions are made.

HTML Tracker ([slug]-tracker.html): Visual interface with a toggle at the top
to switch between Todo view (default) and PRD view. All data is embedded at
generation time -- no file fetches. Only regenerated on user request.

This split means:
- Agents can update todo items and PRD content without touching the HTML
- New agents read the JSON + PRD to get full project context instantly
- The HTML never breaks from partial agent updates
- State is preserved across regenerations via localStorage

---

## JSON Schema

Same schema as before -- see the existing JSON Schema section. No changes needed.

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

## PRD Format

The PRD is a markdown file with this structure:

# [Project Name] -- Product Requirements Document

> Session Handoff (read this first):
> [5-line summary of current state, what was just completed, what's next,
> any blockers, and key decisions made. Updated by agents after each session.]

## Vision
[One paragraph -- what this product does and why it matters]

## Target Users
[Who uses this and in what context -- be specific]

## Core Features (Current State)
[What exists and works today -- bullet list with status indicators]

## Planned Features
[What's coming -- derived from todo items, grouped by priority]

## Tech Stack
[Frontend, backend, database, AI/external services, deployment]

## Architecture Decisions
[Key decisions made and WHY -- agents append here as decisions happen]
- [date]: [decision] -- [rationale]

## Current Status
[Phase: MVP / Beta / Production / etc.]
[Last updated: date]
[Open blockers: list or "none"]

---

## Agent Instructions -- Updating the JSON

Same as before: use the Read tool (not cat) to read the JSON. Mark items done,
add new items, reprioritize -- all by reading and writing the JSON file.

When another agent needs to understand project state:
  Read the [slug]-todo.json file using the Read tool.
The JSON is the ground truth. Do not read the HTML to understand project state.
Use the Read tool, not cat or Bash, to read the JSON.

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

When changes_since_html reaches 5 or more, inform the user:
  "I've made [N] changes to the project files since the HTML was last generated.
  Say 'sync' when you want to see the updated tracker."

Only regenerate the HTML if the user explicitly asks.

---

## Agent Instructions -- Updating the PRD

When an agent makes a significant decision or discovers new requirements:

1. Read [slug]-prd.md
2. Update the relevant section:
   - New feature discovered: add to "Planned Features"
   - Architecture decision made: append to "Architecture Decisions" with date
   - Feature completed: move from "Planned" to "Core Features (Current State)"
   - Blocker found: update "Current Status"
3. ALWAYS update the "Session Handoff" block at the top with current state
4. Write the PRD back to disk

The Session Handoff block is the most important part -- it's what new agents read
first to understand where things stand.

---

## Environment Detection

In Claude Code (filesystem available):
- JSON: ./[slug]-todo.json
- PRD: ./[slug]-prd.md
- HTML: ./[slug]-tracker.html
- Tell the user: "Your project tracker is ready. Open [slug]-tracker.html in
  your browser. Toggle between Todo and PRD views at the top. Say 'sync' anytime
  to refresh the HTML with the latest data."

In Claude.ai (sandbox environment):
- JSON: /mnt/user-data/outputs/[slug]-todo.json
- PRD: /mnt/user-data/outputs/[slug]-prd.md
- HTML: /mnt/user-data/outputs/[slug]-tracker.html
- Call present_files with all three paths

Deriving the slug: lowercase, hyphens, no special characters.

---

## Badge System -- Dynamic Generation

Same as before. Technical projects use the Standard Dev Badge Set (SECURITY, PERF,
AI, DATA, ARCH, UX, TEST, BIZ). Non-technical projects get contextually generated
badges.

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

Read references/html-design-spec.md for the complete visual design specification.
The key addition is the view toggle at the top of the page:

Toggle bar (between header and filter bar):
- Two buttons: "Todo" | "PRD" -- pill-shaped, side by side
- Active button: filled with accent color (#ff6b2b), white text
- Inactive button: transparent, muted text, border
- Default active: Todo
- Switching toggles visibility of the todo sections vs the PRD content
- PRD view renders the embedded PRD markdown as styled HTML
- Filter bar only shows in Todo view
- localStorage remembers which view was last active

---

## Sync Behavior

When the user says "sync", "update the todo", "update the prd", or "refresh tracker":
1. Read the current [slug]-todo.json and [slug]-prd.md
2. Regenerate [slug]-tracker.html with both datasets embedded
3. Reset changes_since_html to 0 in the JSON
4. Update html_last_generated timestamp
5. Confirm to the user what was synced

Updating EITHER the todo or PRD triggers BOTH to sync into the HTML.

---

## Output Instructions

1. Determine badge set for this project type
2. Generate the PRD first ([slug]-prd.md)
3. Generate todo items derived from the PRD ([slug]-todo.json)
4. Generate HTML with both datasets embedded ([slug]-tracker.html)
5. Detect environment and use correct paths
6. Confirm all three file paths to the user
7. State item counts per section

---

## Quality Checklist

- PRD generated before todo items (ensures alignment)
- PRD has all required sections including Session Handoff
- JSON is valid, matches schema, changes_since_html starts at 0
- HTML embeds both JSON (as TODO_DATA) and PRD (as PRD_DATA) constants
- Toggle bar present between header and filter bar
- Toggle defaults to Todo view
- PRD view renders markdown as styled HTML
- Filter bar hidden in PRD view
- localStorage remembers active view
- All three files written to correct paths
- HTML opens in browser with no console errors
