# Co-Founder Skill

A Claude Code skill that acts as a technical co-founder for solo developers — tracks milestones, pushes back on scope creep, and tells you when to ship.

## Project Type

This is a **Claude Code skill project** — it produces `.md` skill and agent files, not application code. The deliverables are:

1. A **global skill** (`SKILL.md`) — the onboarding bootstrapper
2. A **local skill** (`SKILL.md`) — the daily driver installed per-project
3. An **analyst agent** (`co-founder-analyst.md`) — subagent for deep analysis
4. **Persistent memory files** — `config.json`, `milestones.json`, `journal.md`, snapshots

## Repo Structure

```
co-founder-skill/
├── CLAUDE.md                          <- You are here
├── docs/
│   └── specs/
│       └── 2026-03-12-co-founder-skill-design.md  <- Full design spec
├── skills/
│   └── co-founder/
│       └── SKILL.md                   <- Global skill (bootstrapper + onboarding)
├── agents/
│   └── co-founder-analyst.md          <- Analyst subagent template
└── README.md                          <- Public-facing docs
```

## Design Spec

The full design is at `docs/specs/2026-03-12-co-founder-skill-design.md`. Read this before making any implementation decisions.

## Key Architecture Decisions

- **Global skill creates local files** — the global skill onboards a project, then generates project-local skill + agent + memory files
- **Subagent for analysis** — heavy file reading happens in `co-founder-analyst.md`'s own context window, returns a compact 20-30 line assessment to avoid bloating the founder's session
- **Append-only journal** — `docs/co-founder/journal.md` is never edited, only appended to. This is the agent's persistent memory across sessions
- **Project-agnostic** — detects project structure (Ralph, GitHub Issues, etc.) during onboarding and adapts. No hard dependencies on any specific workflow
- **Partner personality** — blunt peer tone, not parental. Data-driven arguments. Escalates gradually: KEEP BUILDING -> PREPARE TO ADVANCE -> ADVANCE NOW -> URGENT: SHIP

## Milestones

Pre-Alpha -> Internal Alpha -> Alpha -> Beta -> Launch

The most important transition is **Pre-Alpha -> Internal Alpha** — getting the founder to stop building and start using their own product.

## Git Commits

- Do NOT include co-author tags or AI attribution in git commits
