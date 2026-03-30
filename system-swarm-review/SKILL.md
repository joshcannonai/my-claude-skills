---
name: system-swarm-review
description: >
  This skill should be used when the user asks to "run a system review",
  "do a full audit", "swarm review", "run the review agents", or "audit
  this project". Deploys a configurable swarm of up to 11 specialized
  agents that review a codebase in parallel through distinct lenses. Do
  NOT trigger for single-topic questions like "check my security".
context: fork
argument-hint: "[project path or description]"
---

# System Swarm Review

Deploys a configurable swarm of specialized review agents against any software
project. Each agent examines the product through a distinct lens -- user experience,
security, performance, accessibility, code quality, standards compliance, mobile,
and product strategy. Findings are grounded in the actual codebase, not generic
checklists.

All agents are optional. The user selects which to run and chooses the run mode
(sub-agents or agent team) at the confirmation checkpoint.

---

## The Agent Roster

Eleven specialized agents are available. Read references/agent-roster.md for
full definitions of each agent's scope, focus areas, and output format.

User Perspective Agents (simulate real users moving through the product):

| Agent | Focus |
|---|---|
| Newcomer | First impressions, onboarding confusion, jargon, missing context |
| Pressured User | High-stakes flows, error states, what breaks under time pressure |
| Expert User | Missing depth, black-box logic, power features, precision gaps |
| Edge Case User | Non-standard use cases, gaps the product didn't plan for |

Specialist Agents (examine the system directly):

| Agent | Focus |
|---|---|
| Security | Auth, permissions, exposed secrets, injection vectors, RLS |
| Mobile | Responsive layout, touch targets, mobile-only flows, viewport issues |
| Performance | Load times, query count, bundle size, render blocking, waterfalls |
| Accessibility | WCAG 2.1 AA, keyboard nav, color contrast, screen reader paths |
| Code Quality | Tech debt, missing error handling, hardcoded values, test coverage |
| Standards | Anthropic best practices, framework conventions, dependency hygiene |
| Product Strategy | Competitive gaps, missing features, growth opportunities |

Defaults (on unless toggled off): Newcomer, Pressured User, Expert User, Edge Case User, Security, Performance, Standards

Optional (off by default): Mobile, Accessibility, Code Quality, Product Strategy

---

## Phase 1 -- Discovery

Read the project silently before doing anything else.

Scan the file structure using Glob patterns:
  **/*.{tsx,ts,jsx,js,py,vue,svelte}
Exclude: node_modules, dist, .next, __pycache__
Limit to the first 80 results.

Then read strategically:
1. README or docs at root -- stated purpose, audience, setup instructions
2. Auth/onboarding flow -- first user touchpoint
3. Route/navigation files -- what flows exist
4. Core feature screens -- the 3-5 screens a new user hits first
5. API/backend endpoints -- data flow and validation
6. Config files -- env vars, feature flags, hardcoded values, placeholder text
7. Package.json or equivalent -- dependencies, scripts, versions
8. Test files (if any) -- what's covered and what isn't

Build a project brief covering:
- What it does and who it's for (one sentence each)
- Tech stack (frontend, backend, database, AI/external services)
- Core user flows (the paths that matter)
- Immediate red flags from code reading

Read references/walkthrough-guide.md to determine appropriate walkthrough steps
based on product type.

Write the brief to disk before proceeding. Sub-agents have no access to the
parent context window -- everything they need must be in a file.

  mkdir -p .claude/swarm-review

Write to .claude/swarm-review/project-brief.md.

Read references/synthesis-template.md from the skill directory and write its
contents to .claude/swarm-review/synthesis-template.md.

Do not proceed until both files are written.

---

## Phase 2 -- Review Plan

Using the project brief, generate the review plan. Do not show the user yet.

For user-perspective agents: Generate 5-7 walkthrough steps grounded in actual
screens from the codebase.

For specialist agents: Identify 3-5 specific areas of the codebase each should
focus on. Be specific -- vague focus areas produce vague output.

Add all agent configurations to .claude/swarm-review/project-brief.md.

---

## Phase 3 -- Confirmation Checkpoint

Present the plan and let the user toggle agents and choose run mode.

  PROJECT: [Product Name] -- [one-line description]

  WALKTHROUGH: [N] steps -- [brief description of the flow being tested]

  -- AGENT SELECTOR --

    User Perspective:
    [1]  * Newcomer         -- first impressions, onboarding, jargon
    [2]  * Pressured User   -- high-stakes flows, errors, time pressure
    [3]  * Expert User      -- missing depth, power features, precision
    [4]  * Edge Case User   -- non-standard use cases, planning gaps

    Specialists:
    [5]  * Security         -- auth, secrets, injection, permissions
    [6]  . Mobile           -- responsive layout, touch, mobile flows
    [7]  * Performance      -- load times, queries, bundle, rendering
    [8]  . Accessibility    -- WCAG, keyboard nav, contrast
    [9]  . Code Quality     -- debt, error handling, test coverage
    [10] * Standards        -- best practices, conventions, dependencies
    [11] . Product Strategy -- gaps, growth, competitive positioning

    Run Mode:
    [S]  * Sub-agents       -- parallel, isolated, standard cost (default)
    [T]  . Agent team       -- collaborative, shared context, ~7x cost

    Toggle:  type numbers to flip (e.g. "6 8 11")
    Run:     type "go" to launch with current selection
    All on:  type "all"
    Custom:  type "c: [your instructions]"
             e.g. "c: add a DevOps agent focused on CI/CD pipeline"
             e.g. "c: have Security focus specifically on the Supabase RLS policies"

* = ON (will run), . = OFF (skipped)

Wait for user reply. Parse their input:
- Numbers (e.g. "6 8 11") -- flip those agents on/off state, re-display the selector
- "go" -- proceed to Phase 4 with current selection
- "all" -- turn all agents on, re-display
- "s" or "t" -- switch run mode, re-display
- "c: [text]" -- apply the custom instruction (add agent, modify focus, change scope),
  update the project brief, re-display the selector with changes applied
- Any combination (e.g. "6 8 go") -- apply toggles then launch

If the user sends numbers without "go", re-display the updated selector so they
can verify before launching. Only proceed to Phase 4 when "go" is explicitly sent
or the user says something clearly affirmative like "run it", "launch", "let's go".

---

## Phase 4 -- Parallel Agent Deployment

If sub-agents mode (default):
Spawn all active agents simultaneously using the Agent tool -- all in one response.

If agent team mode:
Create a team with each active agent as a teammate. Use natural language to
describe each teammate's role and assign their focus areas. The team will
coordinate and build on each other's findings.

Each agent prompt must include:
1. Path to project brief: .claude/swarm-review/project-brief.md
2. The agent's specific focus areas from Phase 2
3. The output file path for that agent's report
4. Reporting instructions from references/agent-roster.md

Output paths:
  .claude/swarm-review/newcomer.md
  .claude/swarm-review/pressured-user.md
  .claude/swarm-review/expert-user.md
  .claude/swarm-review/edge-case-user.md
  .claude/swarm-review/security.md
  .claude/swarm-review/mobile.md
  .claude/swarm-review/performance.md
  .claude/swarm-review/accessibility.md
  .claude/swarm-review/code-quality.md
  .claude/swarm-review/standards.md
  .claude/swarm-review/product-strategy.md

Wait for all agents to complete before Phase 5.

---

## Phase 5 -- Synthesis + Swarm.Sync.md

After all agents finish, spawn one synthesis agent (from the parent, not a sub-agent).

Synthesis agent reads all output files + the synthesis template, then writes:
1. .claude/swarm-review/[YYYYMMDD-HHMM]-synthesis.md -- timestamped report
2. .claude/swarm-review/Swarm.Sync.md -- persistent file updated each run

Swarm.Sync.md format:

  # Swarm Sync -- [Product Name]

  Last run: [date]
  Agents: [list]
  Run mode: [sub-agents / agent team]

  ## Finding Tally

  | Finding | Agents Who Flagged It | Count | Severity |
  |---------|----------------------|-------|----------|
  | [finding] | [agent names] | [N] | [level] |

  ## Top 10 Actions (UX-Weighted)

  UX is the primary evaluation pillar. At least 4 of the top 10 should be UX
  findings unless genuinely no UX issues were found. All pillars matter.

  | Rank | Action | Type | Agent(s) | Why This Rank |
  |------|--------|------|----------|--------------|
  | 1 | ... | ... | ... | ... |

  ## User Decision Log

  [Empty until user provides feedback. After user reviews:]

  | Finding | Decision | Notes |
  |---------|----------|-------|
  | [finding] | IMPLEMENT / CHANGE / REMOVE / DEFER | [user's notes] |

  ## Previous Runs
  - [date]: [N] agents, [summary of key findings]

---

## Phase 6 -- User Review + Project File Updates

After presenting the synthesis, ask the user:

  "Review the findings above. For each suggestion, tell me:
  - Implement: add to the todo as-is
  - Change: implement with modifications (describe what to change)
  - Remove: don't implement
  - Defer: add to long-term/backlog

  I'll update your todo and PRD based on your decisions."

After the user responds:
1. Update Swarm.Sync.md User Decision Log with their choices
2. Read the project's [slug]-todo.json and add IMPLEMENT/CHANGE items
3. Read the project's [slug]-prd.md and update relevant sections
4. Inform the user: "Todo and PRD updated. Say 'sync' to refresh the HTML tracker."

---

## Output

Confirm all file paths to the user:

  Review complete. [N] agents ran ([mode]). Files:

  Agent reports:   .claude/swarm-review/[agent-name].md (for each)
  Synthesis:       .claude/swarm-review/[timestamp]-synthesis.md
  Swarm Sync:      .claude/swarm-review/Swarm.Sync.md

  Review the findings and tell me what to implement, change, remove, or defer.

Give a brief verbal summary of the top 5 findings.

---

## Gotchas

- Context isolation: Sub-agents start fresh. Everything they need must be in the
  project brief file.

- Synthesis timing: Spawn synthesis only after all agent output files exist.

- Sub-agents cannot spawn sub-agents: Synthesis is always the parent's job.

- Specialist scope drift: Each agent stays in its lane. The synthesis is where
  patterns cross.

- Generic findings: "The code could be better organized" is not a finding.
  Every finding must reference a specific file, component, or behavior.

- Positive washing: If something is broken, say so. The goal is actionable
  intelligence.

- Agent teams cost ~7x more tokens: Only recommend when the user explicitly
  wants collaborative review or the project is large enough to benefit.
