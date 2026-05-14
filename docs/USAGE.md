# IT Team Automation - Usage Guide

## Quick Start

### 1. Prerequisites

```bash
# Install gh CLI
# macOS:
brew install gh

# Ubuntu/Debian:
sudo apt install gh

# Other Linux: https://github.com/cli/cli/blob/trunk/docs/install_linux.md

# Authenticate
gh auth login
gh auth status   # confirm: ✓ Logged in to github.com

# Install jq (required by the issue screening script)
brew install jq        # macOS
sudo apt install jq    # Ubuntu/Debian
```

Your project must also have a git remote named `origin` pointing to GitHub — the skill auto-detects your repo owner and name from it. No manual config needed.

### 2. Install the skill

```bash
# From this repo's directory
mkdir -p ~/.claude/skills/it-team
mkdir -p ~/.claude/agents

# Copy skill files
cp -r skill/* ~/.claude/skills/it-team/

# Copy agent definitions
cp agents/* ~/.claude/agents/

# Create your config from the template
cp skill/config.yaml ~/.claude/skills/it-team/config.yaml

# Edit with your repository details
nano ~/.claude/skills/it-team/config.yaml
```

### 3. Configure

Edit `~/.claude/skills/it-team/config.yaml` — all defaults are safe out of the box. The only field you may want to change:

```yaml
issues:
  ready_label: "ready"   # change if you use a different label

# Repository owner/name are auto-detected from your project's git remote.
# No manual entry needed.
```

### 4. Label your issues

In GitHub, add the `ready` label to the issues you want automated.

```bash
# Create the label if it doesn't exist
gh label create ready --description "Ready for IT team automation" --color "0075ca"

# Apply to issues
gh issue edit 42 --add-label ready
gh issue edit 43 --add-label ready
```

### 5. Run your first session

```bash
# Navigate to your project directory (NOT this repo)
cd ~/your-project

# Start Claude Code
claude

# Invoke the skill
> Start IT team session
```

The session will pause for your confirmation at two points:
- After issue screening (shows quarantined issues)
- After architecture planning (shows which files each agent will touch)

Type `yes` at each prompt to proceed.

---

## Workflow

### What happens during a session

```
1. Issue screening   — fetches issues, quarantines suspicious ones, shows you the list
2. Architecture      — maps issues to files, validates all paths, shows you the plan
3. Branch creation   — creates feature/{prefix}{session-id} from your base branch
4. Implementation    — parallel dev agents write the code, scan for secrets, commit
5. QA               — runs your test suite (if detected)
6. PR creation      — opens a PR with full context: issues, agents, test results
7. Final report     — summary of processed + quarantined issues
```

### What you'll see — Issue screening

```
Found issues with label 'ready'
├─ Clean:       8
└─ Quarantined: 2

⚠  2 issue(s) quarantined — check: .it-sessions/issue_parsing/quarantine_issues.json

Processing 8 clean issues. Continue? (yes/no)
```

Quarantined issue content is never shown to the LLM. To inspect what was flagged and why, read the quarantine file directly:

```bash
cat .it-sessions/issue_parsing/quarantine_issues.json
```

### What you'll see — Architecture plan

```
Architecture Analysis Complete
──────────────────────────────
Work units: 3   Total files: 12   Batches: 1

Dev Agent 1:
  Issues:     #15, #23
  Files:      src/auth.py, src/login.py
  Complexity: medium

Dev Agent 2:
  Issues:     #34
  Files:      src/api.py, src/routes.py
  Complexity: low

Dev Agent 3:
  Issues:     #45
  Files:      src/models.py
  Complexity: low

Continue? (yes/no)
```

### What you'll see — Final report

```
═══════════════════════════════════════════════════════════════
IT Team Session Complete — 20240115-1430
═══════════════════════════════════════════════════════════════

✓ Processed: 8 issues
  • #15 — Add OAuth support
  • #23 — Implement rate limiting
  • #34 — Update API endpoints
  • #45 — Fix model validation
  (+ 4 more)

⚠  Quarantined: 2 issue(s) — check: .it-sessions/issue_parsing/quarantine_issues.json

📊 Statistics
  Commits made:     3
  Files modified:   12
  Dev agents used:  3
  Session duration: 8 minutes

🔗 Pull Request: https://github.com/you/repo/pull/123

🧪 Tests: 42 passed / 1 failed
═══════════════════════════════════════════════════════════════
```

---

## Handling Quarantined Issues

Quarantined issues require manual review — they are never processed automatically, and their content is never shown to the LLM.

```bash
# See which issues were quarantined and why
cat .it-sessions/issue_parsing/quarantine_issues.json

# View the full issue content in GitHub (human review)
gh issue view 42

# If the issue is legitimate, implement it manually
claude "Implement the OAuth feature from issue 42"

# If it looks malicious, close it with a note
gh issue close 42 --comment "Closed: security concern flagged by IT team automation"
```

---

## Session State

Each session creates a directory inside your project:

```
your-project/
└── .it-sessions/
    ├── issue_parsing/                     ← shared across sessions, created once
    │   ├── screen_issues.sh               ← screening script (auto-generated)
    │   ├── injection-patterns.yaml        ← blocklist (auto-translated from skill MD)
    │   ├── clean_issues.json              ← issues that passed screening (full JSON)
    │   └── quarantine_issues.json         ← quarantined: number + reason only
    └── 20240115-1430/                     ← session ID (timestamp)
        ├── config.json                    ← snapshot of loaded config
        ├── architecture.json              ← work unit assignments
        ├── dev-results/
        │   ├── dev-1.json                 ← per-agent results + commit hashes
        │   └── dev-2.json
        ├── test-results.json              ← QA output
        ├── review-results.json            ← code review findings
        ├── review-classifications.json    ← immediate vs deferred per finding
        └── session.log                    ← full audit trail
```

`.it-sessions/` is added to your project's `.gitignore` automatically on first run — session data is never committed.

**Cleanup:** By default, sessions older than 7 days are removed automatically. To keep all sessions:

```yaml
advanced:
  keep_session_data: true
```

---

## Configuration Reference

### Issue filtering

```yaml
issues:
  ready_label: "ready"

  # Also require one of these (AND logic with ready_label)
  # additional_labels: ["bug", "enhancement"]

  # Skip issues with any of these labels
  # exclude_labels: ["wip", "blocked"]
```

### Security mode

```yaml
security:
  # strict  — quarantine on any pattern match (recommended)
  # lenient — only quarantine high-confidence patterns
  mode: "strict"

  # Quarantine issues containing likely secret material
  quarantine_secrets: true

  # Pause before creating PR (useful during initial rollout)
  require_manual_approval: false
```

### Resource limits

```yaml
limits:
  max_issues: 20           # per session
  max_parallel_agents: 10
  max_files_per_issue: 50  # issues claiming more are quarantined
  session_timeout_hours: 2
  agent_timeout_minutes: 30
```

### File scope

```yaml
files:
  allowed_extensions:
    - .py
    - .js
    - .ts
    # add your project's types here

  forbidden_directories:
    - .git/
    - .github/workflows/
    - node_modules/
    # add custom protected directories here
    # - deployment/
    # - config/prod/
```

### Branch naming

```yaml
branches:
  prefix: "feature/it-session-"
  base: "main"
```

### Dry run

```yaml
advanced:
  dry_run: true   # generates plan but does not commit or create PR
```

---

## Code Review Workflow

**Always review the PR before merging:**

```bash
# Check out the branch locally
gh pr checkout 123

# Review all changes
git diff main

# Run your tests
pytest           # or: npm test, go test ./..., etc.

# If everything looks good
gh pr review 123 --approve
gh pr merge 123 --squash
```

---

## Security Hygiene (After Each Session)

```bash
# Confirm no files outside project scope were modified
git diff --name-only main..HEAD | grep "^\.\."

# Scan commits for accidental secrets
git log -p main..HEAD | grep -iE "api[_-]?key|secret|password|token"

# Review quarantined issues (number + reason only)
cat .it-sessions/issue_parsing/quarantine_issues.json

# Check audit trail
tail -50 .it-sessions/$(ls -t .it-sessions | grep -v issue_parsing | head -1)/session.log
```

---

## Troubleshooting

### `gh: command not found`

```bash
brew install gh          # macOS
sudo apt install gh      # Ubuntu/Debian
# Other: https://cli.github.com/
```

### `authentication failed`

```bash
gh auth login
gh auth status   # verify after login
```

### No issues found with label

```bash
# Check if the label exists in your repo
gh label list --repo owner/repo

# Create it if missing
gh label create ready --description "Ready for automation"

# Apply to an issue
gh issue edit 42 --add-label ready
```

### `Configuration not found`

```bash
cp ~/.claude/skills/it-team/config.yaml.template \
   ~/.claude/skills/it-team/config.yaml
nano ~/.claude/skills/it-team/config.yaml
```

### All issues quarantined

Review what triggered the quarantine:

```bash
cat .it-sessions/issue_parsing/quarantine_issues.json
```

If they look legitimate, implement the safe ones manually:

```bash
claude "Implement the feature from issue 42"
```

### Dev agent failed to commit

Check the agent result:

```bash
cat .it-sessions/{session-id}/dev-results/dev-1.json | jq '{status, error, details}'
```

Common causes: CRITICAL code review finding, secret detected in file, git conflict.

---

## Advanced Usage

### Custom branch prefix

```yaml
branches:
  prefix: "automated/"
  base: "develop"
```

### Go / Rust / other languages

Add extensions to the allowed list:

```yaml
files:
  allowed_extensions:
    - .go
    - .rs
    - .java
    - .rb
```

### Protect custom directories

```yaml
files:
  forbidden_directories:
    - .git/
    - .github/workflows/
    - deployment/         # ← custom
    - config/prod/        # ← custom
```

### Progressive rollout

Recommended approach for teams adopting this for the first time:

| Week | Config | Goal |
|------|--------|------|
| 1 | `max_issues: 3`, `require_manual_approval: true` | Learn the workflow |
| 2 | `max_issues: 10`, `require_manual_approval: true` | Build confidence |
| 3+ | Default settings | Full operation |

---

## Project Structure After First Run

After the first session, your project will contain:

```
your-project/
├── .git/
├── .gitignore              ← updated: .it-sessions/ added automatically
├── .it-sessions/           ← session data (git-ignored)
│   └── 20240115-1430/      ← example session directory
├── src/
│   └── ...
└── ...
```

The `.it-sessions/` directory lives inside your project so session data stays co-located with the code it produced. It is never committed to git.

---

## FAQ

**Q: Can I use this with GitLab or Bitbucket?**
A: GitHub only for now. The skill uses `gh` CLI which is GitHub-specific.

**Q: What if I have more than 100 ready issues?**
A: The GitHub API returns at most 100 per call. The session will process the first 20 (configurable). Split into multiple sessions using `additional_labels` to filter subsets.

**Q: Can agents install npm/pip packages?**
A: No — this is explicitly forbidden in agent instructions. Install dependencies manually before running a session that needs them.

**Q: What about issues referencing more than 50 files?**
A: Quarantined automatically (`limits.max_files_per_issue`). Too broad to safely automate — implement manually.

**Q: How do I update the skill after a new release?**
A: Pull the latest from this repo, then re-copy:

```bash
cp -r skill/* ~/.claude/skills/it-team/
cp agents/* ~/.claude/agents/
# Do NOT overwrite your customized config.yaml
```

**Q: Can I add custom quarantine patterns?**
A: Yes. Edit `~/.claude/skills/it-team/reference/injection-patterns.md` and add entries under the relevant category. The skill auto-regenerates `.it-sessions/issue_parsing/injection-patterns.yaml` on the next run when it detects the MD file is newer.

**Q: The PR was created but tests failed — should I merge?**
A: Your call. Failed tests are included in the PR body for visibility. The skill deliberately doesn't block merging on test failure. Review the failures and decide.

---

## Getting Help

- **Threat model:** [docs/SECURITY.md](SECURITY.md)
- **Session logs:** `.it-sessions/{session-id}/session.log`
- **Quarantine details:** `.it-sessions/issue_parsing/quarantine_issues.json`
- **Injection patterns:** `skill/reference/injection-patterns.md`
- **Screening script:** `.it-sessions/issue_parsing/screen_issues.sh`
