# my-claude-skills

A collection of production-ready [Agent Skills](https://agentskills.io) for Claude Code and Claude.ai — built and maintained by [Josh Cannon](https://joshcannonai.vercel.app).

Agent Skills are modular, version-controlled folders of instructions, scripts, and reference materials that Claude loads dynamically to perform specialized tasks. Think of them as SOPs for AI agents — built once, reused across every project.

---

## Skills

### [`system-swarm-review`](./system-swarm-review/)

Deploys a configurable swarm of up to 10 specialized review agents against any software project. Each agent reviews through a distinct lens. The user selects which agents to run before anything executes.

**The roster — all named after Shawshank Redemption characters:**

*User Perspective (simulate real users):*
- **Andy (Newcomer)** — first impressions, onboarding confusion, jargon
- **Ellis (Pressured)** — high-stakes flows, error states, time pressure
- **Brooks (Expert)** — missing depth, black-box logic, power feature gaps
- **Jake (Edge Case)** — non-standard use cases, planning gaps

*Specialists (examine the system directly):*
- **Red (Security)** — auth, RLS, secrets, injection vectors, permissions
- **Heywood (Mobile)** — responsive layout, touch targets, mobile-only flows
- **Tommy (Performance)** — load times, query count, bundle size, rendering
- **Norton (Accessibility)** — WCAG 2.1 AA, keyboard nav, color contrast
- **Hadley (Code Quality)** — tech debt, error handling, test coverage
- **Skeet (Product Strategy)** — competitive gaps, missing features, growth

**Defaults on:** Andy, Ellis, Brooks, Jake, Red, Tommy — covers the most critical ground. Toggle any of the 10 on/off at the confirmation checkpoint.

**Example output:**
```
.claude/swarm-review/
├── project-brief.md
├── andy.md
├── ellis.md
├── brooks.md
├── jake.md
├── red.md
├── tommy.md
└── 20260329-1430-synthesis.md
```

**Requirements:** Claude Code with filesystem access and Task tool (sub-agent) support.

---

### [`project-todo-html`](./project-todo-html/)

Generates a two-file task tracking system: a JSON source of truth that agents read and write, and an HTML visual tracker that humans open in the browser.

**Two-file architecture:**
- `[project]-todo.json` — agents update this between sessions to track progress
- `[project]-todo.html` — regenerated from the JSON when you ask, or when enough changes accumulate

This means agents can mark tasks done, add new items, and reprioritize mid-session without touching the HTML. The HTML is always a clean snapshot generated from the JSON.

**HTML features:**
- Section filters (Critical / Active / Polish / Long-term) + full-text search + tag filtering
- Stats bar: Total / Completed / Remaining with gradient progress bar
- Dynamic badge system — dev projects get SECURITY/PERF/AI/DATA/ARCH/UX/TEST/BIZ; other projects get contextually generated categories
- localStorage persistence — checkbox state survives tab closes and page reloads
- Sync button — reloads HTML from disk while preserving checkbox state
- Copy as Markdown — exports open items for pasting into GitHub issues or agent prompts

**Works in:** Claude Code (writes to project directory) and Claude.ai (writes to outputs sandbox).

---

## Installation

### Claude Code (recommended)

```bash
# Clone the repo
git clone https://github.com/joshcannonai/my-claude-skills.git

# Copy skills to your global Claude Code skills directory
cp -r my-claude-skills/system-swarm-review ~/.claude/skills/
cp -r my-claude-skills/project-todo-html ~/.claude/skills/

# Or add to a specific project
cp -r my-claude-skills/system-swarm-review /your/project/.claude/skills/
cp -r my-claude-skills/project-todo-html /your/project/.claude/skills/
```

### Claude.ai

1. Download the skill folder you want
2. Zip it — the zip must contain the folder itself at the root, not just the contents
3. Go to **Settings → Features → Skills → Upload**

> Skills do not sync between Claude Code and Claude.ai. Install separately for each surface.

---

## Usage

Skills trigger automatically when Claude recognizes a relevant request.

```
# Triggers system-swarm-review
"Run a full system review on this project"
"Swarm review this codebase"
"Do a full audit"
"Run all the review agents"

# Triggers project-todo-html
"Generate a todo list for this project"
"Create a master task tracker"
"What's left to build?"
"Make me a todo html"
```

---

## Skill Structure

Each skill follows the [Agent Skills open standard](https://agentskills.io/specification):

```
skill-name/
├── SKILL.md          # Required: YAML frontmatter + instructions
├── LICENSE           # MIT
└── references/       # Loaded on demand — agents read only what they need
```

SKILL.md bodies are kept lean per the spec's 500-line recommendation. Reference files load only when the skill explicitly calls for them.

---

## Design Principles

**Ground everything in real context.** Every finding traces to something in the actual codebase — a file, a config value, a missing handler. Generic feedback is useless.

**Defaults over menus.** Skills make a clear decision and let you override — they don't present equal options and ask you to choose.

**One skill, one job.** Tight scope means reliable triggering and no conflicts between skills.

**Honest failure modes.** Each skill documents its own gotchas — the mistakes Claude makes without the skill, and the patterns the skill itself is prone to.

**Named characters, consistent behavior.** Andy, Ellis, Brooks, Jake, Red, Heywood, Tommy, Norton, Hadley, and Skeet appear in every review. Consistent names mean consistent mental models and comparable output across runs.

---

## Roadmap

- `agent-handoff` — Structured handoff prompts for Claude Code sub-agents with stack context, current state, task list, and verification steps
- `technical-debt-audit` — Focused standalone code quality pass
- `job-application` — Tailored cover letters and interview prep from a personal context file + job description

---

## License

MIT — see LICENSE in each skill folder.

---

*Built with [Claude Code](https://code.claude.com) + [Agent Skills](https://agentskills.io). Portfolio: [joshcannonai.vercel.app](https://joshcannonai.vercel.app)*
