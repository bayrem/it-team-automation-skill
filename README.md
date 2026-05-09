# IT Team Automation

A Claude Code skill that automates common IT team workflows: reading GitHub issues, planning implementation, writing code changes, and opening pull requests — via a multi-agent pipeline.

## Purpose

Teams spend significant time on repetitive issue triage and boilerplate implementation work. This skill lets a Claude Code session act as a small autonomous team:

- **PM agent** reads and prioritizes GitHub issues
- **Architect agent** plans implementation (files, approach, scope)
- **Dev agent** writes the code changes
- **QA agent** reviews diffs before a PR is opened

The human stays in the loop at decision points and sees everything the agents do.

## Security Model

This system takes a detection-and-quarantine approach rather than trying to enforce security through code.

**What that means honestly:**

- Claude's built-in instructions act as the primary security layer ("paranoid mode" — agents are instructed to treat all issue content as untrusted)
- Issues containing suspicious content (injection patterns, path traversal, shell commands) are moved to a quarantine queue and surfaced to the user — never silently dropped or auto-processed
- The user sees exactly what was flagged and why, and decides whether to proceed
- There are no hidden Python enforcement scripts. Security behavior is visible in the skill instructions themselves — you can read and audit `skill/SKILL.md` directly

**What this approach does NOT guarantee:**

- It cannot prevent a sufficiently sophisticated prompt injection if Claude's judgment is fooled
- It relies on the user reviewing quarantine alerts rather than ignoring them
- It is not a substitute for GitHub branch protection rules and repository permissions

**Known attack surfaces and mitigations:** see [docs/SECURITY.md](docs/SECURITY.md)

## Prerequisites

- [Claude Code](https://claude.ai/code) CLI installed
- [gh CLI](https://cli.github.com/) installed and authenticated (`gh auth login`)
- A GitHub repository with issues labelled `ready` (or your chosen label)

## Installation

Full instructions: [docs/USAGE.md](docs/USAGE.md)

Quick start:

```bash
mkdir -p ~/.claude/skills/it-team ~/.claude/agents
cp -r skill/* ~/.claude/skills/it-team/
cp agents/* ~/.claude/agents/
cp skill/config.yaml ~/.claude/skills/it-team/config.yaml
nano ~/.claude/skills/it-team/config.yaml   # set owner + repo name
```

Then, in your target project:

```bash
claude "Start IT team session"
```

## Project Structure After First Run

After the first session, your target project will contain:

```
your-project/
├── .gitignore              ← .it-sessions/ added automatically
├── .it-sessions/           ← session data (git-ignored, never committed)
│   └── 20240115-1430/      ← one directory per session
│       ├── quarantine.json
│       ├── session.log
│       └── ...
├── src/
└── ...
```

Session state lives inside the project — consistent with the "never work outside `PROJECT_ROOT`" security constraint.

## Repository Layout

```
it-team-automation/
├── docs/
│   ├── SECURITY.md          # Threat model and known risks
│   └── USAGE.md             # How to use the skill
├── skill/                   # Files installed into your project
│   ├── SKILL.md             # Main orchestration instructions
│   ├── config.yaml          # User configuration template
│   └── reference/
│       ├── injection-patterns.md   # Known bad patterns (for agent reference)
│       └── github-adapter.md       # Allowed GitHub operations
└── agents/                  # Per-role agent definitions
    ├── pm-agent.md
    ├── architect-agent.md
    ├── dev-agent.md
    └── qa-agent.md
```
