# HTML Design Spec (v2)

Reference for generating the visual HTML tracker. The shell is written once and
loads its data from a sibling `[slug]-data.js` file that agents regenerate on
every change. A meta refresh tag reloads the page every 5 seconds so the UI
stays live with whatever the agent has most recently written.

---

## Data Loading & Auto-Refresh

Every HTML shell includes a meta refresh tag in `<head>`:

```html
<meta http-equiv="refresh" content="5">
```

And a cache-busting inline script at the top of `<body>` that injects the
`<script src="./[slug]-data.js">` tag with a timestamp query string on each
page load:

```html
<script>
  const s = document.createElement('script');
  s.src = './[slug]-data.js?t=' + Date.now();
  document.body.appendChild(s);
  s.onload = () => window.dispatchEvent(new Event('tracker-data-loaded'));
</script>
```

All render logic waits for the `tracker-data-loaded` event. The data file
exposes `window.TRACKER_DATA = { todo, prd }` — a single global the rest of the
script reads from. Do NOT hardcode a `<script src="data.js">` tag in the HTML
— the browser will cache it and the refresh will show stale data.

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
- localStorage saves active view in the state object: `activeView: "todo" | "prd"`

## Data Shape

`[slug]-data.js` exposes a single global:

```js
window.TRACKER_DATA = {
  todo: { /* full JSON contents of [slug]-todo.json */ },
  prd: "/* full PRD markdown as a string */"
};
```

PRD markdown is rendered to HTML at render time using a lightweight inline
markdown parser (~50 lines of JS for headings, lists, bold, code blocks,
blockquotes). No external markdown library needed.

All render logic reads from `window.TRACKER_DATA.todo.*` and `window.TRACKER_DATA.prd`.
The HTML shell has no data literals — it's a pure view.

When regenerating, rewrite only `[slug]-data.js`. The HTML shell is never
touched after initial generation.

### Visual Design
- **Background**: `#0d0d0f`, surfaces `#141416` / `#1a1a1d`
- **Fonts**: IBM Plex Mono + IBM Plex Sans — Google Fonts (only external dependency)
- **Accent**: `#ff6b2b` — project title, active states, completed count
- **Full-width layout** — no max-width centering, `padding: 36px 48px` on `<main>`

### Header (sticky, two rows)
Row 1: Project name (orange, IBM Plex Mono), subtitle, generation date
Row 2: Stats bar — three cards (Total / Completed / Remaining) with gradient
progress bar and white percentage label

Stats bar CSS values: `font-size: 26px` for numbers, `color: #e8e8ec` for
percentage label, gradient `linear-gradient(90deg, #ff6b2b, #ffaa00)` for bar

### Filter Bar (sticky below header, top set dynamically via JS)
Left to right: All / 🔴 Critical / 🟡 Active / 🟢 Polish / 🔵 Future | Show
Completed (N) (with live count derived from `TRACKER_DATA.todo.sections[].items[].done`) |
Search (200→260px on focus, orange border when active) | Tags ▾
(multi-select popover, data-tag uses class slug e.g. `b-1`) | ↻ Sync | Copy MD

The completed counter updates on every render. It's computed inline:
```js
const completedCount = window.TRACKER_DATA.todo.sections
  .flatMap(s => s.items)
  .filter(i => i.done).length;
```

### Items
- Click toggles `.done` → strikethrough + 0.42 opacity + faded badge
- Each badge has `data-tip` tooltip (3-6 words) that appears on hover
- `data-id="item-N"` assigned by JS on init for localStorage mapping

### Section progress bars
3px bar below each section header, fills as items are checked, color matches
section accent, animated with CSS transition

### localStorage

**Key:** `tracker-state-[slug]` (one key per tracker)

**Shape:**
```js
{
  scrollY: 420,                      // page scroll position
  activeView: "todo",                // "todo" | "prd"
  activeFilter: "all",               // "all" | "critical" | "active" | "polish" | "longterm"
  showCompleted: false,              // boolean
  collapsedSections: ["polish"],     // section ids the user collapsed
  prdCollapsedBlocks: [],            // PRD block ids the user collapsed
  activeTags: [],                    // active tag filters
  done: ["item-0", "item-3"]         // locally-toggled done state (optimistic UI)
}
```

**Save:** on `beforeunload`, write the full blob.
**Restore:** on `tracker-data-loaded` (before first render), read the blob and
apply it. Scroll position should be restored AFTER `renderTracker()` runs, so
content exists to scroll against.

Note: v1 used the key `[slug]-todo-state`. v2 trackers use `tracker-state-[slug]`
— if a user regenerates an existing tracker, the old state is orphaned (not
migrated). This is intentional; the shape is different enough that attempting
migration would be fragile.

### ↻ Sync button

Calls `location.reload()` — forces an immediate refresh instead of waiting for
the 5-second meta-refresh tick. Useful when the user says "sync" to the agent
and wants to see the update land in the browser right now. Since localStorage
persists the view state, the reload is nearly invisible to the user.
