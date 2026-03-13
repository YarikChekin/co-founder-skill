# Co-Founder Skill

A Claude Code skill that acts as a technical co-founder for solo developers — tracks milestones, pushes back on scope creep, and tells you when to ship.

## The Problem

Solo founders get stuck in the build loop: adding features, polishing UI, fixing edge cases — without ever shipping. Research shows co-founded startups outperform solo ventures not because of more output, but because of complementary judgment — the willingness to say "this is good enough, let's learn from real users."

This skill fills that role. It doesn't write code. It makes sure you stop writing code at the right time.

## How It Works

Three layers:

1. **Global skill** — Installed once on your machine. Runs the onboarding flow when you invoke `/co-founder` on a new project: reads your PRD, asks about your goals, defines milestones with specific exit criteria, and generates project-local files.

2. **Local skill** — Generated per-project during onboarding. This is your daily driver. Handles on-demand assessments, scope gates (challenges you when you add features), milestone advancement, and session-start nudges.

3. **Analyst agent** — A subagent that runs in its own context window. Reads your PRD, milestones, journal, git history, and test results, then returns a compact assessment with a recommendation: KEEP BUILDING, PREPARE TO ADVANCE, ADVANCE NOW, or URGENT: SHIP.

## Milestones

The skill uses standard release lifecycle milestones. Not every project needs all five.

| Milestone | Who's Testing | Goal |
|-----------|--------------|------|
| **Pre-Alpha** | Just you | Build core features |
| **Internal Alpha** | You, with real data | Use it yourself |
| **Alpha** | 5-10 trusted testers | Does it work for others? |
| **Beta** | 50+ people | Do people want this? |
| **Launch** | Public | We're live |

The most important transition is **Pre-Alpha → Internal Alpha** — getting you to stop building and start using your own product with real data.

## Installation

Copy the skill to your Claude Code global skills directory:

```bash
mkdir -p ~/.claude/skills/co-founder
cp skills/co-founder/SKILL.md ~/.claude/skills/co-founder/SKILL.md
```

That's it. The global skill handles everything else per-project.

## Usage

### First time on a project

```
/co-founder
```

The skill runs the onboarding flow:
1. Scans for your PRD, git repo, test suites, and task tracking
2. Asks about your core user journey, timeline, and alpha testers
3. Proposes milestones with specific exit criteria for your review
4. Generates all local tracking files
5. Gives you an initial "you are here" assessment

### Daily use

```
/co-founder
```

Gets a full status assessment: where you are against your milestones, scope health, risk signals, and a recommendation.

The skill also integrates at key moments:
- **Sprint completion** — checks if you've hit a milestone boundary
- **Scope addition** — challenges new features against your agreed milestones
- **Session start** — nudges you if a previous assessment said it's time to advance

## What It Creates

After onboarding, your project gets these files:

```
your-project/
├── .claude/
│   ├── skills/co-founder/SKILL.md       # Local daily driver
│   └── agents/co-founder-analyst.md     # Analyst subagent
└── docs/co-founder/
    ├── config.json                      # Project metadata
    ├── milestones.json                  # Milestone definitions + exit criteria
    ├── journal.md                       # Append-only decision log
    └── snapshots/
        └── pre-alpha-lock-YYYY-MM-DD.md # PRD snapshot at milestone lock
```

All files are committed to git and persist across sessions.

## Personality

The co-founder is a blunt peer, not a project manager or cheerleader:

- Uses data, not opinions ("You've been in Pre-Alpha for 8 weeks")
- No filler praise — only honest assessment
- Asks the uncomfortable question ("What are you actually worried will happen if a friend uses this?")
- Remembers past decisions and holds you to them
- Acknowledges when scope changes are legitimate
- Escalates gradually: supportive → encouraging → direct → blunt

## Workflow Integration

The local skill documents how other tools can integrate co-founder hooks:

- **At sprint completion:** Invoke `/co-founder` before planning the next sprint
- **At scope addition:** Invoke `/co-founder` when proposing new features
- **At session start:** Check `docs/co-founder/journal.md` for pending ADVANCE NOW or URGENT: SHIP nudges

## Prerequisites

| Requirement | Required? | Why |
|-------------|-----------|-----|
| A PRD or product spec | Yes | Defines what you're building and enables milestone tracking |
| Git repository | Yes | Used for history analysis and file persistence |
| Claude Code | Yes | The skill runs within Claude Code |
| Sprint/task tracking | No | Gives richer signals about progress |
| Test suite | No | Used in readiness assessments |

## License

MIT
