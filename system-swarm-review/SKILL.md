---
name: system-swarm-review
description: >
  Use when the user wants a full system review, product audit, code quality pass,
  UX research session, or multi-angle analysis of any software project. Trigger
  phrases: "run a system review", "do a full audit", "swarm review", "product
  review", "run the review agents", "full system check", "audit this project".
  Reads the codebase autonomously, generates a review plan with selectable agents,
  confirms which agents to run, then spawns them in parallel. Each agent reviews
  the product through a distinct lens and writes findings to disk. A synthesis
  agent compiles all reports into a ranked action list. Do NOT trigger for
  single-topic questions like "check my security" or "review this component" --
  those don't need the full swarm.
license: MIT
compatibility: Requires Claude Code with filesystem access and Task tool for
  parallel sub-agent spawning. Confirmation checkpoint is conversational.
metadata:
  author: joshcannonai
  version: "1.0"
---

# System Swarm Review

Deploys a configurable swarm of specialized review agents against any software
project. Each agent examines the product through a distinct lens -- user experience,
security, performance, accessibility, code quality, mobile, and product strategy.
Findings are grounded in the actual codebase, not generic checklists.

All agents are optional. The user selects which to run at the confirmation
checkpoint before anything executes.

---

## The Agent Roster

Ten specialized agents are available. Read references/agent-roster.md for the
full definition of each agent's scope, focus areas, and output format.

User Perspective Agents (simulate real users moving through the product):

| Agent | Focus |
|---|---|
| Andy (Newcomer) | First impressions, onboarding confusion, jargon, missing context |
| Ellis (Pressured) | High-stakes flows, error states, what breaks under time pressure |
| Brooks (Expert) | Missing depth, black-box logic, power features, precision gaps |
| Jake (Edge Case) | Non-standard use cases, gaps the product didn't plan for |

Specialist Agents (examine the system itself):

| Agent | Focus |
|---|---|
| Red (Security) | Auth, permissions, exposed secrets, injection vectors, RLS |
| Heywood (Mobile) | Responsive layout, touch targets, mobile-only flows, viewport issues |
| Tommy (Performance) | Load times, query count, bundle size, render blocking, waterfalls |
| Norton (Accessibility) | WCAG 2.1 AA, keyboard nav, color contrast, screen reader paths |
| Hadley (Code Quality) | Tech debt, missing error handling, hardcoded values, test coverage |
| Skeet (Product Strategy) | Competitive gaps, missing features, growth opportunities, business model |

Defaults (on unless toggled off): Andy, Ellis, Brooks, Jake, Red, Tommy

Optional (off by default): Heywood, Norton, Hadley, Skeet

---

## Phase 1 -- Discovery

Read the project silently before doing anything else.

Scan the file structure:

  find . -type f \( -name "*.tsx" -o -name "*.ts" -o -name "*.jsx" -o -name "*.js" \
    -o -name "*.py" -o -name "*.vue" -o -name "*.svelte" \) \
    | grep -v node_modules | grep -v dist | grep -v ".next" | grep -v __pycache__ \
    | head -80

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
- Immediate red flags from code reading (TODOs, disabled auth, exposed keys,
  console.logs, hardcoded values, missing error handlers)

Read references/walkthrough-guide.md to determine appropriate walkthrough steps
for the user-perspective agents based on product type.

Write the brief to disk before proceeding. Sub-agents have no access to the
parent context window -- everything they need must be in a file.

  mkdir -p .claude/swarm-review

Write to .claude/swarm-review/project-brief.md. Also copy the synthesis template:

  cp "$(dirname "$0")/references/synthesis-template.md" \
     .claude/swarm-review/synthesis-template.md 2>/dev/null || true

If the copy fails, read references/synthesis-template.md yourself and write its
contents to .claude/swarm-review/synthesis-template.md manually.

Do not proceed until both files are written.

---

## Phase 2 -- Review Plan

Using the project brief, generate the review plan. Do not show the user yet.

For user-perspective agents (Andy, Ellis, Brooks, Jake):
Generate 5-7 walkthrough steps grounded in actual screens from the codebase.
Read references/agent-roster.md for how to adapt each persona's archetype
to this specific product type.

For specialist agents (Red, Heywood, Tommy, Norton, Hadley, Skeet):
Identify 3-5 specific areas of the codebase each specialist should focus on,
based on what was found during discovery. Vague focus areas produce vague output --
be specific: "Red should focus on the JWT validation in /api/auth, the Supabase
RLS policies, and the user input flowing into the AI prompt in /api/chat."

Add all agent configurations to .claude/swarm-review/project-brief.md.

---

## Phase 3 -- Confirmation Checkpoint

Present the plan and let the user toggle agents. This is a conversational pause.
Do not spawn anything until the user replies.

Present it in this format -- show current on/off state clearly:

  PROJECT: [Product Name] -- [one-line description]

  AGENTS READY TO DEPLOY:

  User Perspective (simulating real users):
    [ON]  Andy (Newcomer)    -- first impressions, onboarding, jargon
    [ON]  Ellis (Pressured)  -- high-stakes flows, errors, time pressure
    [ON]  Brooks (Expert)    -- missing depth, power features, precision
    [ON]  Jake (Edge Case)   -- non-standard use cases, planning gaps

  Specialist Review:
    [ON]  Red (Security)        -- auth, secrets, injection, permissions
    [OFF] Heywood (Mobile)      -- responsive layout, touch, mobile flows
    [ON]  Tommy (Performance)   -- load times, queries, bundle, rendering
    [OFF] Norton (Accessibility)-- WCAG, keyboard nav, contrast
    [OFF] Hadley (Code Quality) -- debt, error handling, test coverage
    [OFF] Skeet (Product Strategy) -- gaps, growth, competitive positioning

  WALKTHROUGH: [N] steps -- [brief description of flow]

  Toggle any agents on/off, or say "run it" to proceed with defaults.

Wait for user reply. Apply changes to the brief file, then proceed to Phase 4.

---

## Phase 4 -- Parallel Agent Deployment

Spawn all active agents simultaneously using the Task tool -- all in one response.
Do not run sequentially.

Each Task prompt must include:
1. Path to project brief: .claude/swarm-review/project-brief.md
2. The agent's specific focus areas from Phase 2
3. The output file path for that agent's report
4. Reporting instructions from references/agent-roster.md for that agent type

Output paths:
  .claude/swarm-review/andy.md
  .claude/swarm-review/ellis.md
  .claude/swarm-review/brooks.md
  .claude/swarm-review/jake.md
  .claude/swarm-review/red.md
  .claude/swarm-review/heywood.md
  .claude/swarm-review/tommy.md
  .claude/swarm-review/norton.md
  .claude/swarm-review/hadley.md
  .claude/swarm-review/skeet.md

Only spawn agents that are toggled ON. Skip files for agents that were toggled OFF.

Wait for all active agents to complete before proceeding to Phase 5.

---

## Phase 5 -- Synthesis

After all agents finish, the parent agent (not a sub-agent) spawns one synthesis
agent. Sub-agents cannot spawn sub-agents -- synthesis is always the parent's job.

Before spawning, verify all active agent output files exist.

Synthesis Task prompt:

  You are synthesizing results from a multi-agent system review.

  Read these files in order:
  1. .claude/swarm-review/project-brief.md
  2. [each active agent output file]
  3. .claude/swarm-review/synthesis-template.md (required output structure)

  Write the completed synthesis to:
  .claude/swarm-review/[YYYYMMDD-HHMM]-synthesis.md

  Follow the template exactly. Every section is required.
  Validate before finalizing:
  - All CRITICAL findings from any agent appear in the synthesis
  - Cross-agent patterns are documented with agent attribution
  - Top 10 list is ranked by real impact, not by agent order
  - The one-sentence summary is honest -- not a marketing tagline
  - No finding was invented that isn't in the source reports

---

## Output

Confirm all file paths to the user:

  Review complete. [N] agents ran. Files written to .claude/swarm-review/

  Agent reports:   [list only the ones that ran]
  Synthesis:       .claude/swarm-review/[timestamp]-synthesis.md

Give a brief verbal summary of the top 5 findings across all agents.

---

## Gotchas

- Context isolation: Sub-agents start fresh. Everything they need must be in the
  project brief file. Never assume a sub-agent knows anything from the conversation.

- Synthesis timing: Spawn synthesis only after all agent output files exist.
  Check with: ls .claude/swarm-review/ before spawning.

- Sub-agents cannot spawn sub-agents: Synthesis is always the parent's job,
  called after the swarm completes -- not inside any agent Task.

- Specialist scope drift: Red should not comment on UX. Tommy should not audit
  auth. Each agent stays in its lane. The synthesis is where patterns cross.

- Generic findings: "The code could be better organized" is not a finding.
  "The student.py file is 3,200 lines with no router separation -- breaking it into
  feature routers would halve debugging time" is a finding.

- Positive washing: If something is genuinely broken, say so. The goal is
  actionable intelligence, not a report that makes the developer feel good.
