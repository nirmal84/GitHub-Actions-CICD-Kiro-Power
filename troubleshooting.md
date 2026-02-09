# Troubleshooting Guide

## Common Errors and Fixes

### Error: "Resource not accessible by integration"

**Cause:** The `GITHUB_TOKEN` doesn't have required permissions.

**Fix:** Add explicit permissions to your workflow:

```yaml
permissions:
  contents: read
  pull-requests: write    # Add the permission you need
```

---

### Error: "No such file or directory" in run step

**Cause:** Usually a missing `actions/checkout` step, or incorrect working directory.

**Fix:**

```yaml
steps:
  - uses: actions/checkout@v4    # Don't forget this!
  - run: npm ci
    working-directory: ./packages/api    # If your code is in a subdirectory
```

---

### Error: "Process completed with exit code 1" (generic failure)

**Debugging steps:**

```yaml
steps:
  - name: Debug info
    run: |
      echo "OS: $(uname -a)"
      echo "Node: $(node --version 2>/dev/null || echo 'not installed')"
      echo "Working dir: $(pwd)"
      echo "Files: $(ls -la)"

  - name: Run with verbose output
    run: npm test -- --verbose
    env:
      DEBUG: '*'               # Enable debug logging
```

---

### Error: "Workflow is not valid" / YAML syntax errors

**Common YAML mistakes:**

```yaml
# BAD: Incorrect indentation
jobs:
  build:
  runs-on: ubuntu-latest    # Wrong! Should be indented under build

# GOOD:
jobs:
  build:
    runs-on: ubuntu-latest

# BAD: Using tab characters (YAML requires spaces)
# GOOD: Always use 2-space indentation

# BAD: Missing quotes on special values
on:
  push:
    branches: [main, 3.x]    # 3.x might be interpreted as number

# GOOD:
on:
  push:
    branches: [main, '3.x']
```

---

### Error: Cache not being restored

**Common causes and fixes:**

```yaml
# 1. Key mismatch — make sure hash input file is correct
- uses: actions/cache@v4
  with:
    path: node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    #                                      ^ Check this file exists!

# 2. Path mismatch — cache path must match where deps are installed
    path: ~/.npm               # This caches the npm download cache
    path: node_modules         # This caches installed packages

# 3. Different OS — caches are OS-specific by default
    key: ${{ runner.os }}-...  # Make sure this matches
```

---

### Error: "This request has been automatically failed because it uses a deprecated version of `actions/upload-artifact`"

**Fix:** Update to v4:

```yaml
# OLD (deprecated):
- uses: actions/upload-artifact@v3

# NEW:
- uses: actions/upload-artifact@v4
```

**Breaking changes in v4:**
- Artifact names must be unique within a workflow run
- Artifacts are immutable (can't overwrite)
- Use `merge-multiple: true` in download to combine artifacts

---

### Error: Workflow not triggering

**Checklist:**

1. **File location:** Workflow must be in `.github/workflows/` directory
2. **File extension:** Must be `.yml` or `.yaml`
3. **Branch:** Workflow file must exist on the default branch for certain triggers
4. **Path filters:** Check if `paths` or `paths-ignore` is filtering out your changes
5. **Disabled workflows:** Check in the Actions tab if the workflow is disabled
6. **Fork restrictions:** Fork PRs don't trigger `pull_request_target` by default

```yaml
# Debug: Add workflow_dispatch to manually trigger
on:
  push:
    branches: [main]
  workflow_dispatch:    # Allows manual triggering from Actions tab
```

---

### Error: "The job was not started because recent account payments have failed"

**Cause:** Billing issue on the GitHub account.

**Fix:** Check billing settings at `https://github.com/settings/billing`

---

### Error: Secrets not available in PR from fork

**This is expected behavior** — GitHub doesn't expose secrets to fork PRs for security.

**Workaround:** Use the `pull_request_target` event cautiously, or implement an approval workflow:

```yaml
on:
  pull_request_target:
    types: [labeled]

jobs:
  deploy-preview:
    if: contains(github.event.pull_request.labels.*.name, 'safe-to-test')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}
```

---

## Debugging Techniques

### Enable debug logging

Set these repository secrets to enable verbose output:

| Secret | Value | Purpose |
|--------|-------|---------|
| `ACTIONS_RUNNER_DEBUG` | `true` | Runner diagnostic logs |
| `ACTIONS_STEP_DEBUG` | `true` | Step debug logging |

### Debugging with tmate (SSH into runner)

```yaml
- name: Setup tmate session
  if: failure()              # Only on failure
  uses: mxschmitt/action-tmate@v3
  timeout-minutes: 15        # Auto-terminate after 15 min
```

### Dump all GitHub context

```yaml
- name: Dump contexts
  run: |
    echo "github context:"
    echo '${{ toJson(github) }}'
    echo "env context:"
    echo '${{ toJson(env) }}'
    echo "job context:"
    echo '${{ toJson(job) }}'
```

### Check action versions

```yaml
- name: Check action versions
  run: |
    echo "Runner: ${{ runner.os }} ${{ runner.arch }}"
    echo "Node: $(node --version)"
    echo "npm: $(npm --version)"
    echo "Python: $(python3 --version)"
    echo "Docker: $(docker --version)"
    echo "Git: $(git --version)"
```

---

## Workflow Validation

### Validate locally before pushing

Use `actionlint` to check workflows locally:

```bash
# Install
brew install actionlint     # macOS
go install github.com/rhysd/actionlint/cmd/actionlint@latest  # Go

# Run
actionlint .github/workflows/*.yml
```

### Validate with GitHub CLI

```bash
# Check workflow syntax
gh workflow list
gh run list --workflow=ci.yml
gh run view <run-id> --log-failed
```

---

## Performance Debugging

### Identify slow steps

Check the workflow run in the Actions tab — each step shows its duration. Common slow spots:

1. **Dependency installation** → Add caching
2. **Docker builds** → Use BuildKit cache (`type=gha`)
3. **E2E tests** → Parallelize with matrix or sharding
4. **Large checkouts** → Use `fetch-depth: 1` for shallow clone

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 1         # Shallow clone (faster)
    # Use fetch-depth: 0 only if you need git history
```

### Monitor workflow usage

```bash
# Check billable minutes
gh api /repos/{owner}/{repo}/actions/billing
```
