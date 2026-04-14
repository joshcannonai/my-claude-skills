# Example Output — Worked Sample

This reference shows what a real swarm-review run produces end-to-end. Read it
if you're running the skill for the first time and want to know what "done"
looks like before you commit to a run.

The example below is synthetic — it's based on a plausible student project
("CampusNova", a React + Supabase note-taking app for college students) but
the findings are illustrative, not from a real audit.

---

## Sample `project-brief.md` (from Phase 1)

```markdown
# Project Brief — CampusNova

## What it does
A note-taking app for college students. Organize notes by class, tag by
topic, share with classmates. React frontend, Supabase backend (Postgres + Auth + Storage).

## Who it's for
Students enrolled in traditional college courses. The product assumes a
semester-based academic calendar and a limited number of courses (4-6 per
semester).

## Tech stack
- Frontend: React 18 + Vite + Tailwind + shadcn/ui
- Backend: Supabase (Postgres + RLS + Auth + Storage)
- Deployment: Vercel
- Auth: Email + Google OAuth via Supabase Auth

## Core user flows
1. Sign up → create first class → create first note (<2 min to value)
2. Take a note during lecture → tag it → save
3. Search across all notes in a class
4. Share a note with a classmate
5. Export a class's notes as markdown

## Project size
- 47 files (React components, hooks, Supabase queries)
- 8,200 LOC
- Classification: small (1.0x cost multiplier)

## Immediate red flags from code reading
- No RLS policies visible in `supabase/migrations/` — needs a specialist check
- Search is client-side only, loads ALL notes into memory
- No mobile-specific styling despite students using phones heavily

## Persona Candidates

1. **Freshman in Week 1** (pre-selected)
   - Context: just arrived on campus, downloaded the app from a friend, setting it up on their phone during orientation
   - Unique lens: no existing note organization habits, easily confused by unexplained jargon, judging the app by first 60 seconds
   - Device: mobile, data-constrained

2. **Senior managing 4 majors worth of notes** (pre-selected)
   - Context: double major + minor, 4 years of accumulated notes, needs cross-class search and export for thesis
   - Unique lens: cares about search quality, bulk export, performance with 1000+ notes
   - Device: laptop, fast network

3. **TA running 3 sections**
   - Context: grad student TAing multiple sections, wants to share notes with students
   - Unique lens: sharing/collaboration flows, distinguishing students from each other
   - Device: laptop

4. **Student with ADHD**
   - Context: uses the app as a working memory aid, needs distraction-free capture
   - Unique lens: friction points, interruption handling, forgiveness on mistyping
   - Device: mixed

## Specialist Candidates (filtered for Web app)
1. Security — RLS policies, auth flows, secret handling, OAuth config
2. Performance — query count, bundle size, client-side search scale, image loading
3. Accessibility — WCAG 2.2 AA, keyboard nav, screen reader paths, color contrast
4. Mobile — responsive layout, touch targets, viewport issues, network-constrained flows
```

---

## Sample `personas/freshman-week-1.md` (from Phase 4)

```markdown
# Freshman in Week 1 — Review

**Persona:** Just arrived on campus, downloaded the app from a friend, setting
it up on their phone during orientation. No existing note organization habits.
Judging the app by the first 60 seconds.

---

## Findings

### 1. Onboarding asks for 8 fields before any value
**Severity:** high | **Type:** UX | **Files:** `src/components/Onboarding.tsx:42-88`

Sign-up flow asks for: email, password, full name, school, year, major, graduation year, and timezone before showing the home screen. As a freshman fidgeting with their phone during orientation, I almost bounced. 8 fields is too many to commit to a brand-new app I haven't even seen yet.

**What I'd do instead:** ask for email + password only. Everything else can be deferred to the first time they create a note (e.g., "what class is this for?" auto-suggests creating the class). Let the value show up BEFORE the friction.

### 2. The word "syllabus" isn't explained
**Severity:** medium | **Type:** UX | **Files:** `src/components/ClassSetup.tsx:15`

The class setup screen says "Upload your syllabus to enable smart tagging." A freshman in week 1 doesn't know what a syllabus is yet — that's a word professors start using in week 2 when they hand you one. The button reads like upsell jargon and I skipped it.

**What I'd do instead:** add a tooltip or a "what's this?" link that explains "A syllabus is the class outline your professor gives you. You can skip this and add it later." Also consider renaming the CTA from "Upload your syllabus" to "Add class details (optional)."

### 3. No keyboard for "Continue" button on mobile
**Severity:** high | **Type:** Mobile/UX | **Files:** `src/components/Onboarding.tsx:120`

On my phone, after filling in the email field, the keyboard's "Next" button doesn't advance to the password field — it submits the form. I typed my email, hit Next, and got a "password required" error. Confusing.

**What I'd do instead:** wire up `tabIndex` correctly on all form fields and ensure the Enter/Next key on mobile keyboards advances to the next field, not submits.

### 4. "Share with classmates" implies I already have classmates
**Severity:** low | **Type:** UX | **Files:** `src/components/NoteActions.tsx:60`

The share button on a note says "Share with classmates" — but in week 1, I don't know anyone yet. The button feels irrelevant and vaguely shameful.

**What I'd do instead:** default the share button to "Share (link or email)" and only swap to "Share with classmates" once the user has added 2+ contacts or joined a class Slack.

---

<!-- SWARM_AGENT_COMPLETE -->
```

---

## Sample `specialists/security.md` (from Phase 4)

```markdown
# Security Specialist — Review

---

## Findings

### 1. RLS policies missing on `notes` table
**Severity:** critical | **Type:** Security | **Files:** `supabase/migrations/003_create_notes.sql`

The `notes` table has no row-level security policies. Any authenticated user
can SELECT any other user's notes via the Supabase client. Tested by hitting:
```sql
SELECT * FROM notes WHERE user_id != auth.uid();
```
Returns all rows. This is a hard data leak.

**Fix:** Add a policy:
```sql
CREATE POLICY "Users see only their own notes"
ON notes FOR ALL
USING (auth.uid() = user_id);
```

### 2. Supabase service role key exposed to client
**Severity:** critical | **Type:** Security | **Files:** `src/lib/supabase.ts:8`

The file imports `process.env.SUPABASE_SERVICE_ROLE_KEY` and uses it in a
client-side context (Vite exposes env vars starting with `VITE_` to the
browser bundle — even if this is in `.env`, any `SUPABASE_*` var getting
prefixed with `VITE_` ends up in the built JS). Service role bypasses RLS
entirely; leaking it to the client defeats all authorization.

**Fix:** Use the public `anon` key on the client. Service role belongs only on
the server (edge functions, API routes). Audit `.env` for any `VITE_SUPABASE_SERVICE_*`
prefixes and rename.

### 3. Google OAuth redirect URL not validated
**Severity:** medium | **Type:** Security | **Files:** `src/routes/auth/callback.tsx:12`

The post-auth redirect uses `window.location.origin + searchParams.get('next')`
without validating that `next` is an internal path. An attacker can craft a
link like `/auth/callback?next=https://evil.com` and phish users through the
app's own auth flow.

**Fix:** Whitelist allowed redirect paths, or at minimum check that `next`
starts with `/` and doesn't contain `:` or `//`.

---

<!-- SWARM_AGENT_COMPLETE -->
```

---

## Sample `synthesis.md` (from Phase 5)

```markdown
# Swarm Review Synthesis — CampusNova

**Run:** 20260414-1530
**Agents:** freshman-week-1, senior-managing-notes, security, performance
**Model:** sonnet | **Mode:** full | **Cost:** ~$0.42

---

## Top 10 Actions (UX-Weighted)

| Rank | Action | Type | Agent(s) | Why This Rank |
|------|--------|------|----------|---------------|
| 1 | Add RLS policies to `notes` table | Security | security | Critical data leak — any user can read any other user's notes |
| 2 | Remove service role key from client bundle | Security | security | Critical — bypasses all authorization if leaked |
| 3 | Cut onboarding from 8 fields to email+password | UX | freshman-week-1 | Quoted verbatim: "I almost bounced" — this kills activation |
| 4 | Paginate client-side search (currently loads ALL notes) | Performance | senior-managing-notes, performance | Senior with 1000+ notes says "it takes 8 seconds to open the app" |
| 5 | Validate OAuth redirect URL | Security | security | Phishing vector through own auth flow |
| 6 | Add tooltip explaining "syllabus" for freshmen | UX | freshman-week-1 | Week-1 jargon barrier — users skip a feature they'd benefit from |
| 7 | Wire up mobile keyboard "Next" to advance fields | Mobile/UX | freshman-week-1 | Form submission bug, blocks completion on phones |
| 8 | Rename "Share with classmates" → "Share" for users with 0 contacts | UX | freshman-week-1 | Button implies a social graph the user doesn't have yet |
| 9 | Lazy-load note attachments (currently eager on mount) | Performance | performance | 2.5s initial page load on a 50-note class |
| 10 | Add `<main>` landmark + skip-to-content link | Accessibility | (would need accessibility agent, not run) |

## Finding Tally

| Finding | Flagged by | Severity |
|---------|-----------|----------|
| RLS missing on notes | security | critical |
| Service key in client | security | critical |
| Onboarding friction | freshman-week-1 | high |
| Client-side search scale | senior-managing-notes, performance | high |
| OAuth redirect unchecked | security | medium |
| Syllabus jargon | freshman-week-1 | medium |
| Mobile keyboard tab order | freshman-week-1 | high |
| Share button presumption | freshman-week-1 | low |
| Attachment eager-load | performance | medium |

## Themes

**Security is the most urgent lane.** Two critical findings (RLS + service key)
are effectively data leaks waiting to happen. These should be fixed before
anything else ships.

**UX issues cluster around first-run experience.** The freshman persona
surfaced 4 findings in onboarding + jargon + mobile keyboard + share button
— all specific to "user has been on the app for less than 5 minutes." Good
activation fixes pay off more than late-funnel polish at this stage.

**Performance is a scaling concern, not an acute issue.** Senior persona with
1000+ notes hit slowness. A new user with 10 notes won't notice. Schedule for
M2, not M1.

---
```

---

## Sample `synthesis.json` (from Phase 5, abbreviated)

```json
{
  "run_id": "20260414-1530",
  "project": "CampusNova",
  "project_size": "small",
  "started_at": "2026-04-14T15:30:00Z",
  "completed_at": "2026-04-14T15:41:00Z",
  "synthesis_completed_at": "2026-04-14T15:41:45Z",
  "synthesis_status": "complete",
  "model": "sonnet",
  "mode": "full",
  "agents_requested": ["freshman-week-1", "senior-managing-notes", "security", "performance"],
  "agents_completed": ["freshman-week-1", "senior-managing-notes", "security", "performance"],
  "agents_failed": [],
  "total_findings": 14,
  "top_actions": [
    {
      "rank": 1,
      "finding": "Add RLS policies to notes table",
      "type": "Security",
      "severity": "critical",
      "files": ["supabase/migrations/003_create_notes.sql"],
      "agents": ["security"],
      "why_this_rank": "Data leak — any authed user can SELECT any other user's notes"
    },
    {
      "rank": 3,
      "finding": "Cut onboarding from 8 fields to email+password",
      "type": "UX",
      "severity": "high",
      "files": ["src/components/Onboarding.tsx"],
      "agents": ["freshman-week-1"],
      "why_this_rank": "Quoted verbatim: 'I almost bounced' — activation killer"
    }
  ],
  "findings_by_agent": {
    "freshman-week-1": [
      { "finding": "Onboarding asks for 8 fields before any value", "severity": "high", "files": ["src/components/Onboarding.tsx"], "line_refs": ["src/components/Onboarding.tsx:42-88"] },
      { "finding": "The word syllabus isn't explained", "severity": "medium", "files": ["src/components/ClassSetup.tsx"], "line_refs": ["src/components/ClassSetup.tsx:15"] },
      { "finding": "Mobile keyboard Next button doesn't advance form", "severity": "high", "files": ["src/components/Onboarding.tsx"], "line_refs": ["src/components/Onboarding.tsx:120"] },
      { "finding": "Share with classmates button implies existing social graph", "severity": "low", "files": ["src/components/NoteActions.tsx"], "line_refs": ["src/components/NoteActions.tsx:60"] }
    ],
    "security": [
      { "finding": "RLS policies missing on notes table", "severity": "critical", "files": ["supabase/migrations/003_create_notes.sql"], "line_refs": [] },
      { "finding": "Service role key exposed to client", "severity": "critical", "files": ["src/lib/supabase.ts"], "line_refs": ["src/lib/supabase.ts:8"] },
      { "finding": "OAuth redirect URL not validated", "severity": "medium", "files": ["src/routes/auth/callback.tsx"], "line_refs": ["src/routes/auth/callback.tsx:12"] }
    ]
  }
}
```

---

## What this example demonstrates

1. **Phase 1 produces a project brief that's actually useful** — not just a file list, but actual observations, red flags, and classified size. The brief is what every sub-agent reads.

2. **Personas produce findings grounded in the persona's lens.** The freshman doesn't flag RLS policies (that's security's job) — they flag jargon and onboarding friction. Personas are vertical depth on user experience, not horizontal coverage of everything.

3. **Specialists produce technical findings with file refs.** Every security finding has a `files` array and often `line_refs` — this makes the JSON output pipeable into CI gates that can auto-comment on PRs.

4. **The markdown synthesis is selective; the JSON is complete.** The top 10 in the markdown report is UX-weighted for humans reading; the JSON has every finding from every agent for machines.

5. **Completion markers** (`<!-- SWARM_AGENT_COMPLETE -->`) appear at the end of each agent file. Resume detection looks for these before deciding an agent finished successfully.

6. **Cost is transparent and small.** A full review of a 47-file project on Sonnet ran ~$0.42 — cheap enough to do before every major PR if you want.

---

Use this example as a calibration target: your runs should look structurally
similar. If your findings don't have file references, severities, or specific
"what I'd do instead" suggestions, the agents are being too vague and you
should push them harder in their focus areas.
