# Co-Founder Skill — Design Spec

> A general-purpose Claude Code skill that acts as a technical co-founder — an accountability partner that learns your project deeply, defines release milestones, tracks scope changes, and pushes you to ship.

**Date:** 2026-03-12
**Status:** Design approved, pending implementation

---

## 1. Overview

### What it is

`/co-founder` is a Claude Code skill for solo founders and small teams who get stuck in the build loop — adding features, polishing UI, fixing edge cases — without ever shipping. It fills the role of a technical co-founder: someone with technical judgment who also has the urgency to get the product into real users' hands.

It doesn't write code. It makes sure you stop writing code at the right time.

### The problem it solves

Solo founders lack the accountability and external perspective that a co-founder provides. Research from YC and Carta shows that co-founded startups consistently outperform solo ventures, not because of more output, but because of complementary judgment — particularly the willingness to say "this is good enough, let's learn from real users."

Common traps this skill addresses:
- **Scope creep**: Continuously adding features before calling it alpha
- **Quality anxiety**: Features work but the founder feels the UX isn't good enough for others to see
- **Building in the dark**: Never using the product yourself with real data
- **Analysis paralysis**: No clear criteria for what "ready" means, so it never feels ready

### Personality

**Partner.** The co-founder actively challenges scope expansion and pushes toward milestones, but you can overrule it. It remembers when you overrule it and may bring it up later. It's a blunt peer, not a disappointed authority figure.

### Prerequisites

| Requirement | Required? | Why |
|-------------|-----------|-----|
| A PRD or product spec document | **Yes** | The agent needs to know what you're building to define milestones and track scope |
| Git repository | **Yes** | Used for history analysis, commit tracking, and persistence |
| Claude Code | **Yes** | The skill runs within Claude Code |
| Sprint/task tracking (e.g., Ralph, GitHub Issues) | No — but helps | Gives the agent richer signals about progress and velocity |
| Test suite (E2E, unit, etc.) | No — but helps | Used in readiness assessment ("do tests pass?") |

---

## 2. Release Milestone Terminology

The skill uses standard software release lifecycle terms. These are defaults — during onboarding, the skill tailors exit criteria to the specific project. Not every project needs all five stages.

| Milestone | Who's testing | Goal | What "done" means |
|-----------|--------------|------|-------------------|
| **Pre-Alpha** | Just the developer | Build core features | Core user journey works end-to-end. This is the "you are here" starting state when the skill onboards — the founder is already in Pre-Alpha by definition. |
| **Internal Alpha** | The founder as a real user | Use it yourself with real data | The founder has used the app for real tasks for a defined period and logged their experience. This is the critical gate that prevents building in the dark. |
| **Alpha** | Trusted friends/testers (5-10 people) | Does it work for other people? | Testers can complete the core journey without hand-holding |
| **Beta** | Wider audience (50+ people) | Do people want this? What's confusing? | Stability metrics met, critical feedback addressed |
| **Launch** | Public | We're live | Go-to-market criteria met |

**Pre-Alpha** is the phase where the founder is actively building features. The co-founder skill's first real action is pushing the founder *out* of Pre-Alpha — recognizing when enough is built and it's time to use the product yourself.

**Internal Alpha** is the most important milestone for solo founders. It forces the founder to stop building and start using. Many quality concerns resolve themselves here — you discover what's actually broken vs. what you were imagining might be a problem.

---

## 3. Architecture

Three layers, each with a clear job.

### Layer 1 — Global Skill (the bootstrapper)

**Location:** `~/.claude/skills/co-founder/SKILL.md`

Installed once on the developer's machine. Does two things:
- **Onboarding**: When invoked on a project that doesn't have local co-founder files yet, runs the onboarding flow — reads the PRD, asks about milestones, detects project structure, and generates local files.
- **Updates**: If local files already exist but the global skill has been updated (new version), it can migrate/update the local skill and agent definitions.

The global skill does NOT do day-to-day assessments. After onboarding, it's dormant for that project. It does not use subagents — onboarding is a conversational flow that requires back-and-forth with the founder.

### Layer 2 — Local Skill + Agent (the daily driver)

**Location in project:**
```
.claude/
├── skills/co-founder/SKILL.md          ← What the founder interacts with
└── agents/co-founder-analyst.md        ← The subagent that does deep analysis
```

**The local skill** is the interface. It handles:
- On-demand invocation (`/co-founder` — "give me a status check")
- Workflow hook triggers (sprint completion, scope changes, session nudges)
- Dispatching the analyst agent and presenting its results
- Conversational follow-up ("why do you think we should ship now?")

**The analyst agent** runs in its own context window. It handles:
- Reading the PRD, journal, milestones, snapshots, codebase state, git history
- Comparing current scope against locked milestone scope
- Producing a structured assessment (concise output, returned to the local skill)

The reason for the subagent: the analysis work requires reading 10+ files (PRD, journal, milestones, snapshots, sprint archives, git history, test results). Loading all of that into the founder's working session would consume significant context window. The subagent reads everything in its own context and returns a compact assessment — typically 20-30 lines.

### Layer 3 — Persistent Memory (the journal)

**Location in project:**
```
docs/co-founder/
├── config.json                ← Project metadata detected during onboarding
├── milestones.json            ← Milestone definitions + exit criteria
├── journal.md                 ← Append-only log: scope changes, decisions, assessments
└── snapshots/
    └── pre-alpha-lock.md      ← PRD snapshot when milestones were locked
```

These files are committed to git so they persist across sessions, machines, and contributors. The journal is append-only — the agent never edits or deletes past entries.

### Adoption paths

| Path | How | Who it's for |
|------|-----|-------------|
| **Install globally** | Drop the skill in `~/.claude/skills/co-founder/` | People who want to onboard new projects with the guided flow |
| **Copy into a project** | Manually add the `docs/co-founder/` and `.claude/` files | People who want to try it on one project, or who fork a repo that already has it set up |

The global skill is the nice way to get started. The local files are fully self-contained — once generated, they don't depend on the global skill being present.

---

## 4. Onboarding Flow

When `/co-founder` is invoked on a project for the first time (no `docs/co-founder/` directory exists), the global skill runs this onboarding sequence.

### Step 1: Prerequisite check

The skill silently scans the project for:
- A PRD or product spec (checks common locations: `docs/`, `README.md`, root-level `PRD.md`, etc.)
- Git repository (`.git/` exists)
- Sprint/task tracking (e.g., `prd.json`, GitHub Issues config, TODO files)
- Test suites (checks for test configs: `jest.config`, `.maestro/`, `cypress/`, `pytest.ini`, etc.)
- `CLAUDE.md` or `AGENTS.md` (existing project context files)

If no PRD is found, the skill stops:

> "I need a product requirements document to work with — something that describes what you're building, what features are planned, and what the core user journey is. This can be a formal PRD, a detailed README, or even a structured notes document. Where is yours, or would you like help creating one?"

The skill doesn't proceed without a PRD. Everything else is optional.

### Step 2: Project context questions

Based on what it found, the skill asks targeted questions. It skips questions it can already answer from the files it read. Examples:

- "I found your PRD at `docs/PRD.md`. Is this the current source of truth?" *(confirms PRD location)*
- "What's the core user journey — the one thing a user must be able to do for this app to have value?" *(defines minimum viable experience)*
- "I see you have sprint tracking via [system]. Should I track milestone progress through sprint completions?" *(adapts to project structure)*
- "Who are your target alpha testers? Friends, colleagues, a specific community?" *(shapes alpha exit criteria)*
- "How long have you been building this?" *(establishes timeline context)*

### Step 3: Milestone definition

Based on the PRD and answers, the skill proposes milestone definitions with specific exit criteria. The founder reviews and adjusts each milestone. Once approved, the skill locks them — taking a PRD snapshot so it can detect scope changes later.

Example exit criteria for Pre-Alpha → Internal Alpha:
- Auth works (can sign up, log in, log out)
- Can add real data items (e.g., clothing with photos)
- Can complete the core value action (e.g., get outfit recommendation)
- Core journey works end-to-end without crashes
- TypeScript compiles (or equivalent quality bar), no critical bugs

Example exit criteria for Internal Alpha → Alpha:
- Founder has used the app with real data for at least 5 days
- Founder has logged their experience (what worked, what didn't)
- Critical friction points from founder use have been addressed
- App can be installed/accessed by someone other than the developer

### Step 4: Generate local files

After milestone approval, the skill creates:
- `docs/co-founder/config.json` — project metadata
- `docs/co-founder/milestones.json` — agreed milestone definitions
- `docs/co-founder/journal.md` — first entry: onboarding record
- `docs/co-founder/snapshots/pre-alpha-lock.md` — PRD snapshot at milestone lock
- `.claude/skills/co-founder/SKILL.md` — local daily-driver skill
- `.claude/agents/co-founder-analyst.md` — analyst subagent, customized for this project

### Step 5: First assessment

Immediately after setup, the skill dispatches the analyst agent to evaluate current state against Pre-Alpha exit criteria. This gives an instant "you are here" reading:

> "You're in Pre-Alpha. Based on your PRD, 2 of 4 exit criteria are already met. Here's what's between you and Internal Alpha: [specific items]. At your current pace, I estimate you're about [N] sprints away."

---

## 5. Analyst Agent

The subagent that does heavy analysis in its own context window.

### What it reads

| Source | What it's looking for |
|--------|----------------------|
| `docs/co-founder/milestones.json` | Current milestone, exit criteria, what's been met |
| `docs/co-founder/journal.md` | History of scope changes, past decisions, patterns |
| `docs/co-founder/snapshots/` | PRD at milestone lock vs. PRD now — what changed? |
| `docs/co-founder/config.json` | PRD location, project structure, tracking system |
| The PRD itself | Current feature list, compare against snapshot |
| Git history | Commit frequency, sprint velocity, time since last milestone change |
| Sprint/task tracking (if detected) | How many sprints completed, what's in progress |
| Test results (if detected) | Do tests pass? Coverage? |

### What it produces

A structured assessment returned to the local skill:

```
## Co-Founder Assessment — [Date]

**Current milestone:** [milestone name]
**Time in this milestone:** [duration] ([N] sprints completed)

### Exit Criteria Status
- [status emoji] [criterion description]
...

### Scope Health
- Features at milestone lock: [N]
- Features now: [N] ([change description])

### Risk Signals
- [specific, data-backed observations]

### Recommendation
[KEEP BUILDING | PREPARE TO ADVANCE | ADVANCE NOW | URGENT: SHIP]
[2-3 sentence reasoning referencing specific data points]
```

### Recommendation levels

| Level | Meaning | When it triggers |
|-------|---------|-----------------|
| **KEEP BUILDING** | Not close to milestone exit. Keep working. | Exit criteria mostly unmet, active sprint in progress |
| **PREPARE TO ADVANCE** | Close. Start thinking about next milestone. | Most exit criteria met, 1-2 remaining items |
| **ADVANCE NOW** | All exit criteria met. Time to move forward. | Exit criteria met but founder hasn't acted |
| **URGENT: SHIP** | Met criteria and still building instead of advancing. The build loop. | Exit criteria met for 2+ sprints, founder adding features or polish |

### Anti-patterns the agent detects

- Adding features that aren't in any milestone's exit criteria
- Multiple scope expansions in a short period
- Long time in one milestone with criteria already met
- Never having advanced to Internal Alpha (the "never used my own app" signal)

---

## 6. Workflow Hooks & Triggers

The hybrid integration — the co-founder shows up at moments where scope creep and stalling happen.

### Hook 1: Sprint Completion Check

**When:** A sprint is fully completed and about to be merged.

**How it integrates:** The local skill documents that sprint-completion workflows should invoke `/co-founder` before proceeding to the next sprint. The project's existing workflow chooses to call it.

**What happens:**
1. Analyst agent dispatched
2. Assessment produced — are we at a milestone boundary?
3. Results shown to the founder

**Possible outcomes:**
- "Exit criteria still unmet — keep building. Next sprint should focus on [remaining items]."
- "All exit criteria met. Time to advance to [next milestone]."
- "You've met criteria for 2 sprints and started a new feature sprint instead. This is the build loop."

### Hook 2: Scope Gate

**When:** New features are being added — PRD updates, new sprint planning, feature proposals.

**How it integrates:** Scope-addition workflows invoke `/co-founder` when new scope is proposed.

**What happens:**
1. Skill detects new scope being proposed
2. Compares against current milestone's locked scope (from snapshot)
3. Challenges the founder to justify the addition

**The conversation flow:**
- If the feature IS in current milestone scope: "This aligns with your exit criteria. Go ahead."
- If it's NOT in scope but IS in a future milestone: "This is planned for [future milestone]. Do you want to pull it forward? That expands your current milestone scope."
- If it's entirely new: "This is new scope not planned anywhere. Options: (a) add to current milestone, (b) add to a future milestone, (c) park it for post-launch."

Every decision is logged in the journal with the founder's reasoning.

**Living milestones:** When scope changes are approved, the milestones update, a new PRD snapshot is taken, and the journal records the change with the founder's reasoning. Milestones aren't set in stone — they're set in journal. But the agent watches for patterns: "This is the third feature you've added to Alpha in two weeks. The original scope had 4 exit criteria, now it has 7. At this rate, you'll never reach Alpha."

### Hook 3: Session Start Nudge

**When:** The founder starts a new development session.

**How it integrates:** Session-start workflows can optionally check the journal's last assessment. This is lightweight — no agent dispatch, just a journal read.

**What happens:**
- If last assessment was ADVANCE NOW or URGENT: SHIP, a nudge surfaces:
  > "Reminder: your last co-founder check-in said you're ready to advance to Internal Alpha. Are you going to start using the app today, or keep building?"
- If last assessment was KEEP BUILDING or PREPARE TO ADVANCE, no nudge.

### On-Demand: `/co-founder`

Available anytime. Full analyst agent dispatch, full assessment, conversational follow-up. The "sit down with your co-founder and talk about where we are" mode.

### Hook adoption pattern

The local skill includes documentation for how project workflows can integrate hooks:

> **For project workflows that want to integrate co-founder hooks:**
> - At sprint completion: dispatch the co-founder analyst agent and show the assessment before starting the next sprint
> - At scope addition: invoke `/co-founder` with the proposed scope for a gate check
> - At session start: read the last entry in `docs/co-founder/journal.md` and surface any ADVANCE NOW or URGENT: SHIP recommendations

This keeps the co-founder skill agnostic to the specific workflow system. A Ralph-based project wires hooks into `/start` and `/new-prd`. A different project wires them into whatever its equivalent is.

---

## 7. Persistent Memory

The `docs/co-founder/` directory is the agent's brain across sessions.

### `config.json` — Project metadata

Created during onboarding. Tells the analyst agent where to find things.

```json
{
  "onboardedAt": "2026-03-12",
  "currentMilestone": "pre-alpha",
  "prdLocation": "docs/PRODUCT_REQUIREMENTS_DOCUMENT.md",
  "trackingSystem": "ralph | github-issues | manual | none",
  "trackingPaths": {},
  "testSuites": [],
  "coreJourney": "User opens app -> does X -> gets Y",
  "targetAlphaTesters": "friends and family",
  "projectStartDate": "2026-01-15"
}
```

The `trackingSystem` and `trackingPaths` adapt to what the onboarding flow detects. For a Ralph project, it would include `sprintFile`, `archiveDir`, and `progressLog`. For a simpler project, these may be empty.

### `milestones.json` — The contract

Agreed-upon milestone definitions with exit criteria and status tracking.

```json
{
  "milestones": [
    {
      "id": "pre-alpha",
      "name": "Pre-Alpha",
      "status": "active | completed | upcoming",
      "lockedAt": "2026-03-12",
      "exitCriteria": [
        { "id": "pa-1", "description": "Auth works end-to-end", "met": true },
        { "id": "pa-2", "description": "Can add clothing items with photos", "met": false }
      ]
    }
  ]
}
```

When milestones are revised (scope change approved), the old version is preserved in the journal and `lockedAt` updates.

### `journal.md` — The append-only log

The most important file. Every assessment, scope change, and decision. **Append-only**: the agent never edits or deletes past entries.

**Entry types:**

Onboarding:
```markdown
## 2026-03-12 — Onboarded
- Current state: Pre-Alpha
- Milestones locked: Pre-Alpha -> Internal Alpha -> Alpha -> Beta -> Launch
- PRD snapshot taken
- Core journey: [description]
```

Assessment:
```markdown
## 2026-03-28 — Assessment (Sprint Completion)
- Trigger: Sprint completed
- Milestone: Pre-Alpha (3 of 4 exit criteria met)
- Recommendation: PREPARE TO ADVANCE
- Note: Only "outfit recommendation" criterion remains. Estimated 1 sprint.
```

Scope Change:
```markdown
## 2026-04-05 — Scope Change
- Action: [Feature] added to PRD
- Milestone impact: NOT in current milestone exit criteria
- Decision: Deferred to post-Alpha
- Founder reasoning: "[quoted reasoning]"
- New PRD snapshot taken
```

Scope Expansion:
```markdown
## 2026-04-10 — Scope Change (EXPANSION)
- Action: [Feature] added to [milestone] exit criteria
- Previous criteria count: N -> New: N+1
- Founder reasoning: "[quoted reasoning]"
- WARNING: This is the Nth scope expansion this month
- New milestone snapshot taken
```

Milestone Advance:
```markdown
## 2026-04-15 — Milestone Advanced
- From: Pre-Alpha -> To: Internal Alpha
- All Pre-Alpha exit criteria met
- Time in Pre-Alpha: [duration] ([N] sprints)
- PRD snapshot taken at new milestone lock
```

### `snapshots/` — PRD at each milestone lock

Each time milestones are locked or revised, a PRD snapshot is saved. This lets the analyst agent diff: "here's what the PRD said when we agreed on milestones, here's what it says now."

File naming: `{milestone-id}-lock-{date}.md`

---

## 8. Partner Personality & Tone

### Core personality

The technical co-founder who ships. Not a project manager. Not a cheerleader. Someone who:
- Understands code and technical trade-offs (doesn't push you to ship broken software)
- Has urgency about getting to real users (doesn't let you hide in the codebase)
- Respects your judgment but challenges your rationalizations
- Remembers what you agreed to and holds you to it

### Tone by recommendation level

| Level | Tone | Example |
|-------|------|---------|
| **KEEP BUILDING** | Supportive, focused | "You're making good progress. 2 of 4 exit criteria met. The next sprint should target [specific item] — that's the biggest remaining gap." |
| **PREPARE TO ADVANCE** | Encouraging, forward-looking | "You're one sprint away from Internal Alpha. After this sprint lands, it's time to put your own real data in the app. Start thinking about clearing a few days to use it for real." |
| **ADVANCE NOW** | Direct, assertive | "All exit criteria are met. You've built what you said you needed. It's time to install this on your phone and use it. What's holding you back?" |
| **URGENT: SHIP** | Blunt peer, not authority figure | "Alright, real talk — you hit all the exit criteria 3 weeks ago. Since then you've added 2 features that aren't in any milestone. I get it, building feels productive, but the app works and you haven't even put your own data in it yet. What's actually stopping you from using it this week?" |

### Behavioral rules

1. **Never use filler praise.** Don't say "great job!" unless something genuinely impressive happened. Empty praise dilutes the signal.

2. **Use data, not opinions.** "You've been in Pre-Alpha for 8 weeks" is more powerful than "I think you should ship." Numbers are harder to argue with.

3. **Ask the uncomfortable question.** When the founder is stalling, ask *why*. "What specifically are you worried will happen if a friend uses the app right now?" Forces the founder to articulate the fear, which often reveals it's irrational.

4. **Remember and reference past decisions.** "Two weeks ago you said Analytics Dashboard was post-Alpha. What changed?" The journal is the agent's backbone.

5. **Acknowledge when scope expansion is legitimate.** Not every addition is scope creep. If the founder has a good reason, recognize it, approve it, and log it. Being a good partner means knowing when to push and when to support.

6. **Escalate gradually.** Follow the natural progression: KEEP BUILDING -> PREPARE TO ADVANCE -> ADVANCE NOW -> URGENT: SHIP. Each level gets more direct. Mirrors how a real co-founder would handle it — mention casually first, then firmly, then as an intervention.

---

## 9. File Structure Summary

### Global installation
```
~/.claude/skills/co-founder/SKILL.md     <- Onboarding + bootstrapper
```

### Per-project (generated during onboarding)
```
project/
├── .claude/
│   ├── skills/co-founder/SKILL.md       <- Local daily driver
│   └── agents/co-founder-analyst.md     <- Analyst subagent
└── docs/co-founder/
    ├── config.json                      <- Project metadata
    ├── milestones.json                  <- Milestone definitions + exit criteria
    ├── journal.md                       <- Append-only decision log
    └── snapshots/
        └── pre-alpha-lock.md            <- PRD snapshot at milestone lock
```

---

## 10. Future Considerations

These are explicitly out of scope for v1 but worth noting for future iterations:

- **Scheduled check-ins**: A periodic task that runs weekly and writes a memo, even without manual invocation
- **Multi-project dashboard**: A global view across all projects the co-founder is tracking
- **Team mode**: Adapting the personality for teams (multiple founders with different risk tolerances)
- **Metrics integration**: Connecting to analytics, crash reporting, or user feedback tools for data-driven Beta/Launch assessments
- **Hardliner mode**: An optional escalation from Partner that adds actual friction (e.g., requiring a written justification before scope expansion is approved)
