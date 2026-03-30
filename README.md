# my-claude-skills

A collection of production-ready [Agent Skills](https://agentskills.io) for Claude Code and Claude.ai — built and maintained by [Josh Cannon](https://joshcannonai.vercel.app).

Agent Skills are modular, version-controlled folders of instructions, scripts, and reference materials that Claude loads dynamically to perform specialized tasks. Think of them as SOPs for AI agents — built once, reused across every project.

---

## Skills

### [`system-swarm-review`](./system-swarm-review/)

Deploys a configurable swarm of up to 11 specialized review agents against any software project. Each agent reviews through a distinct lens. The user selects which agents to run and chooses the run mode (sub-agents or agent team) at a confirmation checkpoint before anything executes.

**User Perspective Agents** (simulate real users):
- **Newcomer** — first impressions, onboarding confusion, jargon
- **Pressured User** — high-stakes flows, error states, time pressure
- **Expert User** — missing depth, black-box logic, power feature gaps
- **Edge Case User** — non-standard use cases, planning gaps

**Specialist Agents** (examine the system directly):
- **Security** — auth, RLS, secrets, injection vectors, permissions
- **Mobile** — responsive layout, touch targets, mobile-only flows
- **Performance** — load times, query count, bundle size, rendering
- **Accessibility** — WCAG 2.1 AA, keyboard nav, color contrast
- **Code Quality** — tech debt, error handling, test coverage
- **Standards** — Anthropic best practices, framework conventions, dependency hygiene
- **Product Strategy** — competitive gaps, missing features, growth

**Defaults on:** Newcomer, Pressured User, Expert User, Edge Case User, Security, Performance, Standards (7 agents). Toggle any of the 11 on/off at the confirmation checkpoint.

**Run modes:**
- **Sub-agents** (default) — parallel, isolated context, standard token cost
- **Agent team** — collaborative, shared context, ~7x token cost

**Key features:**
- `Swarm.Sync.md` — persistent findings file updated each run with finding tally, top 10 actions (UX-weighted), and user decision log
- After user reviews findings (IMPLEMENT / CHANGE / REMOVE / DEFER), the skill auto-updates the project's todo JSON and PRD
- UX-weighted synthesis — at least 4 of the top 10 findings are UX-related

**Example output:**
```
.claude/swarm-review/
├── project-brief.md
├── newcomer.md
├── pressured-user.md
├── expert-user.md
├── edge-case-user.md
├── security.md
├── performance.md
├── standards.md
├── 20260329-1430-synthesis.md
└── Swarm.Sync.md
```

**Requirements:** Claude Code with filesystem access and Agent tool (sub-agent) support.

---

### [`project-todo-html`](./project-todo-html/)

Generates a three-file project tracking system: a JSON task list, a PRD markdown file, and an HTML tracker with a toggle between both views.

**Three-file architecture:**
- `[slug]-todo.json` — task tracking source of truth; agents read and write this between sessions
- `[slug]-prd.md` — product requirements document; agents update as decisions are made
- `[slug]-tracker.html` — interactive HTML with toggle between Todo view (default) and PRD view

The JSON and PRD are living documents — agents update them between sessions to track progress, record decisions, and maintain a "Session Handoff" block so new agents can get caught up instantly. The HTML is only regenerated when you say "sync" or "update".

**HTML features:**
- Toggle bar: switch between Todo and PRD views
- Section filters (Critical / Active / Polish / Long-term) + full-text search + tag filtering
- Stats bar: Total / Completed / Remaining with gradient progress bar
- Dynamic badge system — dev projects get SECURITY/PERF/AI/DATA/ARCH/UX/TEST/BIZ; other projects get contextually generated categories
- localStorage persistence — checkbox state and active view survive tab closes
- Sync button + Copy as Markdown

**PRD includes:**
- Session Handoff block (5-line summary for new agents)
- Vision, target users, core features, planned features
- Tech stack, architecture decisions (with dates and rationale)
- Current status and open blockers

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
3. Go to **Settings > Features > Skills > Upload**

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

**UX is the #1 pillar.** User experience is weighted highest in synthesis and rankings because it determines whether people use the product at all.

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
