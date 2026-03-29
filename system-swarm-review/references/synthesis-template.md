# Synthesis Template

Follow this structure exactly. Every section is required. Do not skip any section
even if content is thin. Do not reorder sections.

---

# System Swarm Review -- [Product Name]

Generated: [date]
Agents run: [list of agents that executed]
Run mode: [sub-agents / agent team]
Codebase read: [key files reviewed during discovery]

---

## Critical Findings -- Fix Before Shipping

Issues that block users, create security risk, or cause trust loss. Every CRITICAL
finding from any agent belongs here, with the source agent noted.

| Finding | Agent | Severity | Effort |
|---------|-------|----------|--------|
| [specific finding] | [agent name] | CRITICAL | [S/M/L] |

---

## Cross-Agent Patterns -- Systemic Issues

Problems that appeared independently across multiple agents. These are architectural,
not isolated bugs. The fact that multiple lenses found the same thing means it's real.

| Pattern | Agents Who Found It | What It Means |
|---------|-------------------|---------------|
| [issue] | [names] | [why it matters systemically] |

---

## UX Findings Summary

Top findings from Newcomer, Pressured User, Expert User, and Edge Case User combined.
Focus on patterns -- what did multiple user personas hit?

Newcomer: [2-3 most important findings]
Pressured User: [2-3 most important findings]
Expert User: [2-3 most important findings]
Edge Case User: [2-3 most important findings]

---

## Specialist Findings Summary

Top findings from each specialist agent that ran.

Security: [top 2-3 findings]
Mobile: [top 2-3 findings -- omit section if agent didn't run]
Performance: [top 2-3 findings]
Accessibility: [top 2-3 findings -- omit section if agent didn't run]
Code Quality: [top 2-3 findings -- omit section if agent didn't run]
Standards: [top 2-3 findings -- omit section if agent didn't run]
Product Strategy: [top 2-3 findings -- omit section if agent didn't run]

---

## Features Requested Across Multiple Agents

Features that appeared independently in multiple agent reports. These are the
strongest product roadmap candidates -- multiple lenses arrived at the same gap.

| Feature | Requested By | What It Would Do |
|---------|-------------|-----------------|
| [feature] | [agent names] | [description] |

---

## Features Requested by Single Agent

High-value feature ideas from individual agents. Niche but worth tracking.

| Feature | Agent | Why It Matters |
|---------|-------|---------------|
| [feature] | [agent] | [rationale] |

---

## Top 10 Recommended Actions -- Ranked by Impact (UX-Weighted)

Opinionated ranking across all agents. Mix of fixes and new features. Justify ordering.

UX is the primary evaluation pillar. At least 4 of the top 10 should be UX findings
unless genuinely no UX issues were found. All pillars matter -- but user experience
is weighted highest because it determines whether people use the product at all.

| Rank | Action | Type | Agent(s) | Why This Rank |
|------|--------|------|----------|--------------|
| 1 | [action] | Fix/Feature | [agents] | [reason] |
[through 10]

After writing this section, also write the Top 10 to Swarm.Sync.md (see Phase 5
of SKILL.md for the Swarm.Sync.md format).

---

## Moments Worth Preserving

Specific elements that worked well -- identified by agents as trust-building,
delightful, or technically sound. These should not be changed in future iterations.

If nothing stood out positively: "No clear delight moments identified. This is
a baseline -- not a verdict."

---

## One-Sentence Summary

Complete honestly: "[Product] feels like _____ right now, and the most important
thing to fix is _____."

Both halves must be honest. The first reflects the overall experience. The second
is the single highest-leverage action.

---

## Validation Checklist

Before the synthesis agent finalizes, verify:

- All CRITICAL findings from every agent report appear in the Critical section
- Cross-agent patterns have evidence from at least 2 agent reports
- Top 10 list is ordered by real impact, not by agent execution order
- At least 4 of the top 10 are UX findings (unless genuinely none exist)
- Delight/preserve section is short and genuine -- not manufactured positivity
- One-sentence summary has both halves filled in honestly
- No finding was invented that isn't in the source agent reports
- Sections for agents that didn't run are omitted (not left blank)
- Swarm.Sync.md is written with the finding tally and top 10
