---
name: "github-actions"
displayName: "GitHub Actions CI/CD"
description: "Build, test, and deploy with GitHub Actions. Author production-ready workflows, reusable actions, matrix strategies, and security-hardened CI/CD pipelines following GitHub's official best practices."
keywords: ["github actions", "ci", "cd", "cicd", "ci/cd", "pipeline", "workflow", "deploy", "build", "test", "automation", "github", "actions", "yaml", "runner", "matrix", "artifact", "release", "continuous integration", "continuous deployment"]
author: "Community"
---

# GitHub Actions CI/CD Power

Build production-ready CI/CD pipelines with GitHub Actions. This power gives your Kiro agent expert-level knowledge of workflow authoring, security hardening, performance optimization, and deployment strategies — so you get correct, secure pipelines on the first try instead of debugging YAML through trial and error.

## MCP Servers

This power uses the following MCP server:

- **github**: Provides access to GitHub repositories, pull requests, issues, and Actions workflow runs via the official GitHub MCP server.

## MCP Config Placeholders

After installation, you'll need to configure:

- `GITHUB_PERSONAL_ACCESS_TOKEN`: A GitHub Personal Access Token (classic or fine-grained) with the following scopes:
  - `repo` — full repository access (needed to read/write workflows)
  - `workflow` — update GitHub Actions workflows
  - `actions` — manage Actions runs and artifacts

  Generate one at: https://github.com/settings/tokens

---

# Onboarding

## Step 1: Validate GitHub CLI (optional but recommended)

The GitHub CLI (`gh`) enhances the workflow development experience. Check if it's available:

- Verify with: `gh --version`
- If not installed, workflows can still be authored — the MCP server handles GitHub API access directly.
- If installed, verify authentication: `gh auth status`

## Step 2: Verify repository structure

Before authoring workflows, confirm:

- The project has a `.github/workflows/` directory (create it if missing)
- The repository is hosted on GitHub (check with `git remote -v`)

## Step 3: Understand the project context

Before writing any workflow:

1. Identify the project language/framework (Node.js, Python, Go, Rust, Java, etc.)
2. Check for existing workflows in `.github/workflows/`
3. Look for a `package.json`, `Makefile`, `Dockerfile`, `pyproject.toml`, or similar build config
4. Identify the deployment target (Netlify, Vercel, AWS, GCP, Azure, Docker registry, etc.)

---

# Steering Instructions

This power includes workflow-specific steering files that load on-demand based on what you're working on.

## Steering File Map

| When the user is working on... | Load this steering file |
|-------------------------------|------------------------|
| Creating or editing CI/CD workflows, YAML syntax, triggers, jobs | `workflow-authoring.md` |
| Security hardening, secrets, permissions, supply chain | `security-hardening.md` |
| Performance optimization, caching, matrix strategies, concurrency | `performance-optimization.md` |
| Deployment workflows, environments, release strategies | `deployment-patterns.md` |
| Reusable workflows, composite actions, custom actions | `reusable-actions.md` |
| Debugging failed runs, troubleshooting, error resolution | `troubleshooting.md` |

---

# Quick Reference: Workflow Anatomy

Every GitHub Actions workflow follows this structure:

```yaml
name: CI                          # Workflow display name

on:                               # Trigger events
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:                      # IMPORTANT: always set explicitly
  contents: read

jobs:
  build:                          # Job ID
    runs-on: ubuntu-latest        # Runner
    steps:
      - uses: actions/checkout@v4 # Always pin to major version
      - name: Run tests
        run: npm test
```

## Key Principles (Always Follow)

1. **Pin actions to major versions**: Use `actions/checkout@v4`, not `@main` or `@latest`
2. **Set explicit permissions**: Always include a top-level `permissions` block — default to `contents: read`
3. **Use secrets for sensitive data**: Never hardcode tokens, keys, or passwords in workflow files
4. **Use concurrency groups**: Prevent duplicate runs with `concurrency` blocks
5. **Cache dependencies**: Use `actions/cache@v4` or built-in caching in setup actions
6. **Fail fast by default**: Set `fail-fast: true` in matrix strategies unless you need all results
7. **Use environment protection rules**: For production deployments, always use GitHub Environments with required reviewers

---

# Common Workflow Templates

## Basic CI for Node.js

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm test
      - run: npm run lint
```

## Basic CI for Python

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
      - run: pip install -r requirements.txt
      - run: pytest
      - run: ruff check .
```

## Docker Build and Push

```yaml
name: Docker

on:
  push:
    branches: [main]
    tags: ['v*']

permissions:
  contents: read
  packages: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```
