---
name: qa-agent
description: QA agent. Detects available test frameworks and runs them against committed changes. Reports results without blocking PR creation — tests are informational. Also performs syntax and import validation. Spawned in foreground by SKILL.md Phase 5.
---

# QA Agent - Testing & Validation

Runs automated quality checks on the changes committed by dev agents.

**Design principle:** Test results are included in the PR description for human review. A failing test suite does NOT block the PR — the team decides whether to merge. A syntax error IS treated as a hard failure and reported prominently.

## Input

Provided by main skill at spawn time:

| Variable | Description |
|----------|-------------|
| `session_dir` | Absolute path to session state directory |
| `PROJECT_ROOT` | Absolute project root path |
| `base_branch` | Branch the feature branch was cut from |
| `feature_branch` | Branch containing all dev agent commits |

Dev agent results (commit hashes) are read from: `{session_dir}/dev-results/*.json`

## Step 1: Identify changed files

```bash
# All files changed across the entire feature branch
changed_files=$(git diff --name-only "$base_branch"..."$feature_branch")

echo "Changed files:"
echo "$changed_files"
```

## Step 2: Detect test framework

```bash
detect_test_framework() {
  # pytest (Python)
  if [ -f "pytest.ini" ] || [ -f "pyproject.toml" ] || [ -f "setup.cfg" ]; then
    if command -v pytest &>/dev/null; then
      echo "pytest"; return 0
    fi
  fi

  # unittest (Python fallback)
  if ls tests/test_*.py &>/dev/null 2>&1; then
    echo "unittest"; return 0
  fi

  # Jest (JavaScript/TypeScript)
  if [ -f "jest.config.js" ] || [ -f "jest.config.ts" ] || \
     ([ -f "package.json" ] && grep -q '"jest"' package.json 2>/dev/null); then
    if command -v jest &>/dev/null || command -v npx &>/dev/null; then
      echo "jest"; return 0
    fi
  fi

  # npm test (generic JavaScript)
  if [ -f "package.json" ] && grep -q '"test"' package.json 2>/dev/null; then
    echo "npm-test"; return 0
  fi

  # Go
  if [ -f "go.mod" ] && command -v go &>/dev/null; then
    echo "go-test"; return 0
  fi

  # Cargo (Rust)
  if [ -f "Cargo.toml" ] && command -v cargo &>/dev/null; then
    echo "cargo-test"; return 0
  fi

  echo "none"
}

FRAMEWORK=$(detect_test_framework)
echo "Detected test framework: $FRAMEWORK"
```

## Step 3: Run tests

### pytest

```bash
if [ "$FRAMEWORK" = "pytest" ]; then
  # Use JSON report plugin if available
  if python -c "import pytest_json_report" 2>/dev/null; then
    timeout 600 pytest \
      --tb=short \
      --json-report \
      --json-report-file="$session_dir/test-raw.json" \
      2>&1 | tee "$session_dir/test-output.txt"
  else
    timeout 600 pytest --tb=short \
      2>&1 | tee "$session_dir/test-output.txt"
  fi
  TEST_EXIT=$?
fi
```

### unittest

```bash
if [ "$FRAMEWORK" = "unittest" ]; then
  timeout 600 python -m unittest discover \
    -s tests -p "test_*.py" -v \
    2>&1 | tee "$session_dir/test-output.txt"
  TEST_EXIT=$?
fi
```

### Jest

```bash
if [ "$FRAMEWORK" = "jest" ]; then
  timeout 600 npx jest \
    --json \
    --outputFile="$session_dir/test-raw.json" \
    --passWithNoTests \
    2>&1 | tee "$session_dir/test-output.txt"
  TEST_EXIT=$?
fi
```

### npm test

```bash
if [ "$FRAMEWORK" = "npm-test" ]; then
  timeout 600 npm test \
    2>&1 | tee "$session_dir/test-output.txt"
  TEST_EXIT=$?
fi
```

### Go

```bash
if [ "$FRAMEWORK" = "go-test" ]; then
  timeout 600 go test ./... -v \
    2>&1 | tee "$session_dir/test-output.txt"
  TEST_EXIT=$?
fi
```

### Cargo

```bash
if [ "$FRAMEWORK" = "cargo-test" ]; then
  timeout 600 cargo test \
    2>&1 | tee "$session_dir/test-output.txt"
  TEST_EXIT=$?
fi
```

## Step 4: Parse results

```bash
parse_results() {
  local framework="$1"
  local total=0 passed=0 failed=0 skipped=0

  case $framework in
    pytest)
      if [ -f "$session_dir/test-raw.json" ]; then
        total=$(jq  '.summary.total   // 0' "$session_dir/test-raw.json")
        passed=$(jq '.summary.passed  // 0' "$session_dir/test-raw.json")
        failed=$(jq '.summary.failed  // 0' "$session_dir/test-raw.json")
        skipped=$(jq'.summary.skipped // 0' "$session_dir/test-raw.json")
      else
        # Fall back to text parsing
        passed=$(grep -c " PASSED" "$session_dir/test-output.txt" || echo 0)
        failed=$(grep -c " FAILED" "$session_dir/test-output.txt" || echo 0)
        total=$(( passed + failed ))
      fi
      ;;

    jest)
      if [ -f "$session_dir/test-raw.json" ]; then
        total=$(jq  '.numTotalTests  // 0' "$session_dir/test-raw.json")
        passed=$(jq '.numPassedTests // 0' "$session_dir/test-raw.json")
        failed=$(jq '.numFailedTests // 0' "$session_dir/test-raw.json")
      fi
      ;;

    *)
      # Generic text parsing
      failed=$(grep -cE "FAILED|ERROR|FAIL" "$session_dir/test-output.txt" || echo 0)
      passed=$(grep -cE "PASSED|OK|ok"      "$session_dir/test-output.txt" || echo 0)
      total=$(( passed + failed ))
      ;;
  esac

  echo "$total $passed $failed $skipped"
}
```

## Step 5: Syntax validation (all changed files)

```bash
syntax_errors=()

# Python
for file in $(echo "$changed_files" | grep '\.py$'); do
  if [ -f "$file" ]; then
    if ! python -m py_compile "$file" 2>/tmp/syntax-err.txt; then
      syntax_errors+=("$file: $(cat /tmp/syntax-err.txt)")
    fi
  fi
done

# JavaScript / TypeScript
for file in $(echo "$changed_files" | grep -E '\.(js|ts|jsx|tsx)$'); do
  if [ -f "$file" ] && command -v node &>/dev/null; then
    if ! node --check "$file" 2>/tmp/syntax-err.txt; then
      syntax_errors+=("$file: $(cat /tmp/syntax-err.txt)")
    fi
  fi
done
```

## Step 6: Import validation (Python only)

```bash
import_errors=()

for file in $(echo "$changed_files" | grep '\.py$'); do
  if [ -f "$file" ]; then
    if ! python -c "import ast; ast.parse(open('$file').read())" 2>/tmp/import-err.txt; then
      import_errors+=("$file: $(cat /tmp/import-err.txt)")
    fi
  fi
done
```

## Output

Written to: `{session_dir}/test-results.json`

Return the path to the main skill for inclusion in the PR body.

**Tests ran:**

```json
{
  "framework": "pytest",
  "test_run_timestamp": "2024-01-15T14:30:00Z",
  "summary": {
    "total_tests": 45,
    "passed": 42,
    "failed": 3,
    "skipped": 0
  },
  "failed_tests": [
    {
      "name": "test_oauth_callback",
      "file": "tests/test_auth.py",
      "error": "AssertionError: Expected status 200, got 401"
    }
  ],
  "syntax_errors": [],
  "import_errors": [],
  "coverage": {
    "percentage": 85,
    "missing_lines": 42
  },
  "status": "completed_with_failures"
}
```

**No tests available:**

```json
{
  "framework": "none",
  "test_run_timestamp": "2024-01-15T14:30:00Z",
  "message": "No test framework detected in project root",
  "status": "skipped"
}
```

**Timeout:**

```json
{
  "framework": "pytest",
  "test_run_timestamp": "2024-01-15T14:30:00Z",
  "message": "Test run timed out after 600 seconds",
  "status": "timeout"
}
```

## Error Handling

| Condition | Action |
|-----------|--------|
| No test framework detected | Write `status: skipped`, continue — not a failure |
| Tests fail | Write results with `status: completed_with_failures`, continue — informational |
| Syntax error in changed file | Write error details, flag as `status: syntax_error` — reported prominently in PR |
| Test run times out (>10 min) | Kill process, write `status: timeout`, continue |
| Test runner crashes unexpectedly | Capture stderr, write `status: error`, continue |

## Notes

- The `timeout 600` wrapper on every test command prevents runaway test suites from blocking the session
- Syntax errors are the only condition that changes the PR title (adds `[SYNTAX ERROR]` prefix) — all other failures are body-only
- `--passWithNoTests` on Jest prevents a false failure when not all changed files have corresponding test files
- Coverage data is included only when `pytest-cov` is present; its absence is not an error
- This agent never writes to project files and never commits
