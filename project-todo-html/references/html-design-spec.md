# HTML Design Spec

Reference for generating the visual HTML tracker from JSON data.
Read this when generating or regenerating the HTML file.

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
Saves: { done: ["item-0",...], hideDone: false, activeFilter: "all", activeTags: [] }
Restores fully on page load before first render

Sync button:
Calls location.reload(). Since localStorage persists checkbox state, reloading
always restores the user's progress even when the agent has rewritten the HTML.
