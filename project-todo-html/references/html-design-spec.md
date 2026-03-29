# HTML Design Spec

Reference for generating the visual HTML tracker from JSON data.
Read this when generating or regenerating the HTML file.

---

## View Toggle

Between the header and filter bar, add a toggle bar:
- Container: flex row, centered, gap 8px, padding 12px 0, border-bottom 1px solid #2a2a2d
- Two pill buttons: "Todo" and "PRD"
- Active: background #ff6b2b, color white, font-weight 600
- Inactive: background transparent, color #888, border 1px solid #333
- Both: padding 8px 24px, border-radius 20px, cursor pointer, font-size 14px
- Clicking toggles a CSS class on the main content area
- Todo view: shows all todo sections + filter bar
- PRD view: shows rendered PRD content, hides filter bar and todo sections
- PRD content container: max-width 800px, padding 32px, line-height 1.7
- PRD headings: color #e8e8ec, margin-top 32px
- PRD blockquote (Session Handoff): border-left 3px solid #ff6b2b, padding-left 16px, background #1a1a1d, padding 16px, border-radius 4px
- localStorage saves active view in the state object: activeView: "todo" | "prd"

## Embedded Data

The HTML embeds two JS constants:

  const TODO_DATA = { /* full JSON */ };
  const PRD_DATA = `/* full PRD markdown */`;

PRD markdown is rendered to HTML using a lightweight inline markdown parser
(implement ~50 lines of JS for headings, lists, bold, code blocks, blockquotes).
No external markdown library needed.

---

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
Saves: { done: ["item-0",...], hideDone: false, activeFilter: "all", activeTags: [], activeView: "todo" }
Restores fully on page load before first render

Sync button:
Calls location.reload(). Since localStorage persists checkbox state, reloading
always restores the user's progress even when the agent has rewritten the HTML.
