---
name: pm-agent
description: Project Manager agent. Fetches GitHub issues with specified label. Returns unscreened issue list for main skill to validate. Spawned in foreground by SKILL.md Phase 1.
---

# PM Agent - Issue Fetcher

Fetches issues from GitHub. Does NOT screen for security — the main skill (SKILL.md) handles all quarantine logic after this agent returns.

## Input

Provided by the main skill at spawn time:

| Variable | Description | Example |
|----------|-------------|---------|
| `owner` | GitHub username or org | `brm` |
| `repo` | Repository name | `my-project` |
| `ready_label` | Label that marks issues as ready | `ready` |
| `additional_labels` | Optional: AND filter with additional labels | `bug,enhancement` |
| `exclude_labels` | Optional: skip issues carrying these labels | `wip,blocked` |
| `session_dir` | Path to session state directory inside PROJECT_ROOT | `{PROJECT_ROOT}/.it-sessions/20240115-1430` |

## Operations

### Step 1: Validate inputs

```bash
# Validate owner — alphanumeric + hyphens only
if ! [[ "$owner" =~ ^[a-zA-Z0-9-]+$ ]]; then
  echo "ERROR: Invalid repository owner: '$owner'"
  exit 1
fi

# Validate repo — alphanumeric + hyphens + underscores
if ! [[ "$repo" =~ ^[a-zA-Z0-9_-]+$ ]]; then
  echo "ERROR: Invalid repository name: '$repo'"
  exit 1
fi

# Sanitize label — strip shell metacharacters
ready_label=$(echo "$ready_label" | tr -d ';|&$`<>()')
if [[ -z "$ready_label" ]]; then
  echo "ERROR: ready_label is empty after sanitization"
  exit 1
fi
```

### Step 2: Confirm gh authentication

```bash
if ! gh auth status &>/dev/null; then
  echo "ERROR: gh CLI not authenticated. Run: gh auth login"
  exit 1
fi
```

### Step 3: Fetch issues

```bash
gh issue list \
  --repo "$owner/$repo" \
  --label "$ready_label" \
  --state open \
  --json number,title,body,labels,assignees,createdAt,url \
  --limit 100 \
  > "$session_dir/issues-raw.json"

fetch_exit=$?

if [ $fetch_exit -ne 0 ]; then
  echo "ERROR: gh issue list failed (exit $fetch_exit)"
  exit $fetch_exit
fi

issue_count=$(jq 'length' "$session_dir/issues-raw.json")
echo "Fetched $issue_count issues with label '$ready_label'"
```

### Step 4: Apply additional label filter (if configured)

```bash
if [ -n "$additional_labels" ]; then
  # Build jq expression to require ANY of the additional labels
  jq --argjson labels "$(echo "$additional_labels" | tr ',' '\n' | jq -Rsc 'split("\n")[:-1]')" \
    '[.[] | select(.labels | map(.name) | any(. as $l | $labels[] | . == $l))]' \
    "$session_dir/issues-raw.json" > "$session_dir/issues-filtered.json"
  mv "$session_dir/issues-filtered.json" "$session_dir/issues-raw.json"
  echo "After additional_labels filter: $(jq 'length' "$session_dir/issues-raw.json") issues"
fi
```

### Step 5: Apply exclusion label filter (if configured)

```bash
if [ -n "$exclude_labels" ]; then
  jq --argjson labels "$(echo "$exclude_labels" | tr ',' '\n' | jq -Rsc 'split("\n")[:-1]')" \
    '[.[] | select(.labels | map(.name) | all(. as $l | $labels[] | . != $l))]' \
    "$session_dir/issues-raw.json" > "$session_dir/issues-filtered.json"
  mv "$session_dir/issues-filtered.json" "$session_dir/issues-raw.json"
  echo "After exclude_labels filter: $(jq 'length' "$session_dir/issues-raw.json") issues"
fi
```

## Output

Written to: `{session_dir}/issues-raw.json`

Return the path to the main skill. The main skill performs all security screening before any other agent sees this data.

**Output format:**

```json
[
  {
    "number": 42,
    "title": "Add OAuth authentication",
    "body": "Please implement OAuth login via GitHub provider...",
    "labels": [{"name": "ready"}, {"name": "feature"}],
    "assignees": [],
    "createdAt": "2024-01-15T10:00:00Z",
    "url": "https://github.com/owner/repo/issues/42"
  }
]
```

## Error Handling

| Condition | Output | Action |
|-----------|--------|--------|
| `gh` not installed | `ERROR: gh CLI not found` | Exit 1 |
| Not authenticated | `ERROR: gh CLI not authenticated` | Exit 1 |
| Repository not found | `ERROR: Repository not found` | Exit 1 |
| Zero issues returned | `WARN: No issues with label '{label}'` | Write `[]` to output, exit 0 |
| Rate limit hit | `ERROR: GitHub rate limit exceeded` | Exit 1 |
| Network failure | `ERROR: gh CLI network error` | Exit 1 |

## Notes

- Fetches up to 100 issues (GitHub API `--limit` cap per call)
- Returns raw, unscreened data — the main skill is the security boundary
- Does not read, write, or analyse any project files
- All output is written to `session_dir`, never to the project tree
