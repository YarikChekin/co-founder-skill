# Co-Founder Skill Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code skill that acts as a technical co-founder — tracking milestones, pushing back on scope creep, and telling solo founders when to ship.

**Architecture:** Three-layer system: a global skill (onboarding bootstrapper), a local skill template (daily driver copied into each project), and an analyst agent template (subagent for deep analysis). All three are markdown files — this project produces no application code. The global skill generates per-project local files during an interactive onboarding flow.

**Tech Stack:** Claude Code skills (`.md` files), Claude Code agent definitions (`.md` files), JSON for structured data templates.

**Design spec:** `docs/specs/2026-03-12-co-founder-skill-design.md`

**Skill creation:** All skill and agent `.md` files (Tasks 1-3) MUST be created by invoking `/skill-creator` via the Skill tool. Do not write these files directly.

---

## Chunk 1: Analyst Agent Template

**Why first:** The analyst agent is the foundational building block. Both the global skill (for the first assessment after onboarding) and the local skill (for on-demand checks) dispatch this agent. Writing it first establishes the assessment format that the other two files reference.

### Task 1: Create the analyst agent

**Files:**
- Create: `agents/co-founder-analyst.md`

**What this file does:** This is a Claude Code agent definition that gets copied into each project at `.claude/agents/co-founder-analyst.md` during onboarding. When dispatched as a subagent, it reads all co-founder memory files + project state in its own context window and returns a compact 20-30 line structured assessment.

**Key requirements from the spec (Section 5):**

The agent must:
1. Read these files: `milestones.json`, `journal.md`, `snapshots/`, `config.json`, the PRD, git history, sprint/task tracking (if detected), test results (if detected)
2. Compare current PRD against the milestone-lock snapshot to detect scope changes
3. Produce a structured assessment with: current milestone, time in milestone, exit criteria status, scope health, risk signals, and recommendation
4. Use one of four recommendation levels: KEEP BUILDING, PREPARE TO ADVANCE, ADVANCE NOW, URGENT: SHIP
5. Detect anti-patterns: features outside milestone scope, multiple scope expansions, long time in milestone with criteria met, never advancing to Internal Alpha

**Assessment output format (from spec):**
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

**Recommendation trigger logic (from spec Section 5):**
- KEEP BUILDING: Exit criteria mostly unmet, active sprint in progress
- PREPARE TO ADVANCE: Most exit criteria met, 1-2 remaining items
- ADVANCE NOW: All exit criteria met but founder hasn't acted
- URGENT: SHIP: Exit criteria met for 2+ sprints, founder adding features or polish instead of advancing

**Steps:**

- [ ] **Step 1: Write the analyst agent file**

**REQUIRED:** Invoke `/skill-creator` (via the Skill tool) to create `agents/co-founder-analyst.md`. Do NOT write the file directly — the skill-creator skill knows Claude Code skill/agent format conventions and must be used.

Provide skill-creator with these requirements for the agent definition:

1. **Frontmatter/header:** Agent name, description, what triggers it (dispatched by the local co-founder skill)
2. **Data gathering instructions:** Explicit list of files to read (using paths from `config.json`) and what to extract from each:
   - `docs/co-founder/config.json` → PRD location, tracking system, core journey
   - `docs/co-founder/milestones.json` → current milestone, exit criteria, met/unmet status
   - `docs/co-founder/journal.md` → history of scope changes, past recommendations, patterns
   - `docs/co-founder/snapshots/` → PRD snapshot at milestone lock (for diff against current PRD)
   - The PRD itself (path from config) → current feature list
   - Git history → commit frequency, time since last milestone change, recent activity patterns
   - Sprint/task tracking (if `trackingSystem` in config is not "none") → sprint completions, velocity
   - Test results (if `testSuites` in config is non-empty) → pass/fail status
3. **Analysis logic:** How to compare current state against exit criteria, how to detect scope drift (diff PRD vs snapshot), how to calculate time in milestone
4. **Anti-pattern detection rules:** The four anti-patterns from spec Section 5
5. **Output format:** The exact structured assessment format shown above
6. **Tone:** Data-driven, concise, no filler. This is a report, not a conversation.

The agent should use the `Agent` tool's `subagent_type: "Explore"` pattern — it's a read-only analysis agent that reads files and returns findings. It should NOT use Edit/Write tools.

- [ ] **Step 2: Review the agent file**

Read the created file and verify:
- It includes all data sources from the spec
- The output format matches the spec exactly
- Recommendation levels and trigger logic are correct
- Anti-pattern detection is included
- It references `docs/co-founder/` paths correctly

- [ ] **Step 3: Commit**

```bash
git add agents/co-founder-analyst.md
git commit -m "Add co-founder analyst agent template"
```

---

## Chunk 2: Local Skill Template (Daily Driver)

**Why second:** The local skill is the interface that founders interact with daily. It dispatches the analyst agent (written in Chunk 1) and handles conversations. Writing it second means we can reference the analyst's output format correctly.

### Task 2: Create the local skill template

**Files:**
- Create: `skills/co-founder/local/SKILL.md`

**What this file does:** This is a Claude Code skill that gets copied into each project at `.claude/skills/co-founder/SKILL.md` during onboarding. It's the main interface — the founder types `/co-founder` and this skill handles everything: dispatching analysis, presenting results, managing scope gates, and conversational follow-up.

**Key requirements from the spec (Sections 4-8):**

The skill must handle these modes:

**Mode 1: On-demand assessment (`/co-founder`)**
1. Dispatch the analyst agent (`.claude/agents/co-founder-analyst.md`)
2. Receive the structured assessment
3. Present it to the founder with partner personality (spec Section 8)
4. Be ready for conversational follow-up ("why do you think we should ship now?")

**Mode 2: Scope gate (`/co-founder` with scope change context)**
1. Detect that new scope is being proposed
2. Compare against current milestone's locked scope
3. Three responses based on whether feature is: in current scope, in future milestone, or entirely new
4. Log the decision (approved/deferred/parked) to journal with founder's reasoning
5. If approved expansion: update milestones.json, take new PRD snapshot, log warning if Nth expansion

**Mode 3: Milestone advancement**
1. When founder agrees to advance, update milestones.json (mark current as completed, next as active)
2. Take new PRD snapshot for the new milestone
3. Log advancement to journal with duration and sprint count
4. Dispatch analyst for fresh assessment at new milestone

**Mode 4: Session start nudge (lightweight)**
1. Read last entry in journal.md (no agent dispatch)
2. If last recommendation was ADVANCE NOW or URGENT: SHIP, surface a nudge
3. If KEEP BUILDING or PREPARE TO ADVANCE, stay silent

**Personality rules from spec Section 8:**
- Never use filler praise
- Use data, not opinions
- Ask the uncomfortable question when founder is stalling
- Remember and reference past decisions from journal
- Acknowledge legitimate scope expansion
- Escalate gradually through the four levels

**Journal entry formats (from spec Section 7):**
The skill must append entries using exact formats for: Assessment, Scope Change, Scope Expansion, and Milestone Advance (formats defined in spec Section 7).

**Workflow hook documentation (from spec Section 6):**
The skill must include a section documenting how project workflows can integrate hooks:
- Sprint completion → dispatch analyst
- Scope addition → invoke scope gate
- Session start → check journal for nudge

**Steps:**

- [ ] **Step 1: Write the local skill file**

**REQUIRED:** Invoke `/skill-creator` (via the Skill tool) to create `skills/co-founder/local/SKILL.md`. Do NOT write the file directly — the skill-creator skill knows Claude Code skill format conventions and must be used.

Provide skill-creator with these requirements for the local skill:

1. **Frontmatter:** Skill name (`co-founder`), description, trigger conditions (when user says `/co-founder`, or scope change detected, or session start)
2. **Detection logic:** How to determine which mode to enter:
   - If invoked with scope change context → Mode 2 (scope gate)
   - If session just started and last journal entry has ADVANCE NOW/URGENT: SHIP → Mode 4 (nudge)
   - Default → Mode 1 (full assessment)
3. **Agent dispatch instructions:** How to dispatch `.claude/agents/co-founder-analyst.md` using the Agent tool
4. **Assessment presentation:** How to present the analyst's output with partner personality, adjusted by recommendation level (tone examples from spec Section 8)
5. **Scope gate conversation flow:** The three-branch logic for scope changes (in scope / future milestone / entirely new) with journal logging
6. **Milestone advancement flow:** Update milestones.json, snapshot, journal entry, fresh assessment
7. **Session nudge logic:** Lightweight journal read, conditional nudge
8. **Journal append instructions:** Exact formats for each entry type, with explicit "APPEND ONLY — never edit or delete past entries" rule
9. **Workflow hook documentation:** Block documenting how other skills/workflows can integrate co-founder hooks

The skill must reference these project paths:
- `docs/co-founder/config.json`
- `docs/co-founder/milestones.json`
- `docs/co-founder/journal.md`
- `docs/co-founder/snapshots/`
- `.claude/agents/co-founder-analyst.md`

- [ ] **Step 2: Review the local skill file**

Read the created file and verify:
- All four modes are covered
- Agent dispatch instructions are correct
- Journal entry formats match the spec exactly
- Partner personality rules are encoded
- Scope gate logic has all three branches
- Workflow hook documentation is included

- [ ] **Step 3: Commit**

```bash
git add skills/co-founder/local/SKILL.md
git commit -m "Add co-founder local skill template (daily driver)"
```

---

## Chunk 3: Global Skill (Onboarding Bootstrapper)

**Why third:** The global skill generates the local files from Chunks 1 and 2. Writing it last means we know exactly what files it needs to produce and can embed the correct templates.

### Task 3: Create the global skill

**Files:**
- Create: `skills/co-founder/SKILL.md`

**What this file does:** This is the Claude Code skill installed at `~/.claude/skills/co-founder/SKILL.md`. When `/co-founder` is invoked, it checks if the current project already has co-founder files. If not, it runs the interactive onboarding flow. If yes, it defers to the project-local skill.

**Key requirements from the spec (Sections 3-4):**

**Detection logic:**
- Check if `docs/co-founder/config.json` exists
- If YES → tell the user the project already has co-founder set up, local skill handles it
- If NO → run onboarding flow

**Onboarding flow (5 steps from spec Section 4):**

**Step 1: Prerequisite check (silent scan)**
- Scan for PRD: check `docs/`, `README.md`, root-level `PRD.md`, `docs/PRD.md`, `docs/prd/`, etc.
- Verify `.git/` exists
- Detect sprint/task tracking: `prd.json` (Ralph), `.github/` (GitHub Issues), TODO files
- Detect test suites: `jest.config*`, `.maestro/`, `cypress/`, `pytest.ini`, `vitest.config*`, etc.
- Check for `CLAUDE.md` or `AGENTS.md`
- If no PRD found → stop with message asking founder to create/locate one

**Step 2: Project context questions**
- Confirm PRD location
- Ask for core user journey (the one thing that defines value)
- Ask about tracking system integration (if detected)
- Ask about alpha testers
- Ask how long they've been building
- Skip questions that can be answered from files already read

**Step 3: Milestone definition**
- Propose milestone definitions with specific exit criteria based on PRD and answers
- Founder reviews and adjusts each milestone
- Lock milestones after approval
- Default progression: Pre-Alpha → Internal Alpha → Alpha → Beta → Launch (but not all projects need all stages)

**Step 4: Generate local files**
Create these files:
1. `docs/co-founder/config.json` — from template, filled with onboarding answers
2. `docs/co-founder/milestones.json` — from agreed milestones
3. `docs/co-founder/journal.md` — first entry (onboarding record)
4. `docs/co-founder/snapshots/pre-alpha-lock.md` — copy of PRD at milestone lock
5. `.claude/skills/co-founder/SKILL.md` — copy local skill template (from `skills/co-founder/local/SKILL.md` in this repo, but the global skill embeds the template content)
6. `.claude/agents/co-founder-analyst.md` — copy analyst agent template (from `agents/co-founder-analyst.md` in this repo, but the global skill embeds the template content)

**Step 5: First assessment**
- Dispatch the analyst agent immediately
- Show "you are here" reading with exit criteria status and estimate

**Important architectural note:** The global skill must EMBED the full content of the local skill and analyst agent as templates within itself. When installed globally at `~/.claude/skills/co-founder/SKILL.md`, it won't have access to the repo files. So the templates must be inline.

**Steps:**

- [ ] **Step 1: Write the global skill file**

**REQUIRED:** Invoke `/skill-creator` (via the Skill tool) to create `skills/co-founder/SKILL.md`. Do NOT write the file directly — the skill-creator skill knows Claude Code skill format conventions and must be used.

Provide skill-creator with these requirements for the global skill:

1. **Frontmatter:** Skill name (`co-founder`), description, trigger condition (`/co-founder`)
2. **Detection logic:** Check for `docs/co-founder/config.json` to determine new vs existing project
3. **Existing project path:** Message that local skill handles this project, defer to it
4. **Onboarding Step 1:** Silent prerequisite scan with file detection patterns
5. **Onboarding Step 2:** Context questions (with skip logic for already-detected info)
6. **Onboarding Step 3:** Milestone proposal and interactive approval loop
7. **Onboarding Step 4:** File generation instructions with:
   - `config.json` template (from spec Section 7)
   - `milestones.json` template (from spec Section 7)
   - `journal.md` first entry format (from spec Section 7)
   - PRD snapshot instructions
   - **Embedded local skill template** (full content of the local skill from Chunk 2)
   - **Embedded analyst agent template** (full content of the analyst agent from Chunk 1)
8. **Onboarding Step 5:** First assessment dispatch instructions

- [ ] **Step 2: Review the global skill file**

Read the created file and verify:
- Detection logic correctly checks for existing co-founder files
- All 5 onboarding steps are present
- Prerequisite scan covers the file patterns from the spec
- Context questions match the spec
- File generation includes ALL 6 files
- Local skill and analyst agent templates are embedded inline
- First assessment dispatch is included
- config.json template matches spec Section 7 format
- milestones.json template matches spec Section 7 format

- [ ] **Step 3: Commit**

```bash
git add skills/co-founder/SKILL.md
git commit -m "Add co-founder global skill (onboarding bootstrapper)"
```

---

## Chunk 4: README

### Task 4: Write README.md

**Files:**
- Create: `README.md`

**What this file does:** Public-facing documentation explaining what the co-founder skill is, how to install it, and how it works.

**Sections needed:**
1. **Header + one-line description** — what it is
2. **The problem** — why solo founders need this (from spec Section 1)
3. **How it works** — the three layers explained simply
4. **Milestones** — the progression and what each stage means
5. **Installation** — global install instructions (copy to `~/.claude/skills/co-founder/`)
6. **Usage** — `/co-founder` to onboard, then daily use
7. **What it creates** — the file structure in your project
8. **Personality** — brief description of the partner tone
9. **Workflow integration** — how to wire hooks into existing workflows

**Steps:**

- [ ] **Step 1: Write README.md**

Create the README covering all sections above. Keep it concise and practical — this is a tool for developers, not a marketing page.

- [ ] **Step 2: Review README**

Verify installation instructions are accurate, file paths match the actual deliverables, and the tone matches the project's personality.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "Add README with installation and usage docs"
```

---

## Execution Notes

**Order matters:** Tasks must be executed in order (1 → 2 → 3 → 4) because:
- Task 2 (local skill) references Task 1's output format (analyst agent)
- Task 3 (global skill) embeds the content of Tasks 1 and 2
- Task 4 (README) describes what all three files do

**Invoke `/skill-creator` for Tasks 1-3:** These are Claude Code skill/agent files. You MUST invoke the `/skill-creator` skill (via the Skill tool) for each file creation — do not write skill/agent `.md` files directly. The skill-creator skill knows the correct format, frontmatter conventions, and best practices for Claude Code skills and agents.

**No TDD for this project:** These are markdown skill files, not application code. There's nothing to unit test. Quality verification is done by reading and reviewing the files against the spec.

**Template embedding in Task 3:** The global skill must contain the full text of the local skill and analyst agent as inline templates. This is the trickiest part — the global skill is a single file that, when installed at `~/.claude/skills/co-founder/SKILL.md`, must be able to generate all project-local files without access to any other repo files.
