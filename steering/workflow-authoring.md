# Workflow Authoring Guide

## Trigger Events

### Push and Pull Request (most common)

```yaml
on:
  push:
    branches: [main, develop]
    paths:
      - 'src/**'
      - 'package.json'
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]
```

### Scheduled (cron)

```yaml
on:
  schedule:
    - cron: '0 6 * * 1'  # Every Monday at 6am UTC
```

### Manual dispatch with inputs

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - staging
          - production
      dry_run:
        description: 'Dry run mode'
        required: false
        type: boolean
        default: true
```

### Release triggers

```yaml
on:
  release:
    types: [published]
  push:
    tags:
      - 'v*.*.*'
```

### Path filtering (monorepo pattern)

```yaml
on:
  push:
    paths:
      - 'packages/api/**'
      - '.github/workflows/api-ci.yml'
    paths-ignore:
      - '**.md'
      - 'docs/**'
```

---

## Job Configuration

### Runner selection

| Runner | Use Case |
|--------|----------|
| `ubuntu-latest` | Default for most workloads |
| `ubuntu-24.04` | Pin specific OS version for reproducibility |
| `windows-latest` | .NET, Windows-specific builds |
| `macos-latest` | iOS, macOS builds, Swift |
| Self-hosted | Custom hardware, GPU, private network access |

### Job dependencies

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test

  deploy:
    needs: [lint, test]        # Runs only after both complete
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying..."
```

### Conditional execution

```yaml
steps:
  - name: Deploy to production
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    run: ./deploy.sh

  - name: Comment on PR
    if: github.event_name == 'pull_request'
    run: echo "This is a PR build"

  - name: Run only on tag
    if: startsWith(github.ref, 'refs/tags/v')
    run: ./release.sh
```

### Matrix strategies

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: true
      matrix:
        node-version: [18, 20, 22]
        os: [ubuntu-latest, windows-latest]
        exclude:
          - node-version: 18
            os: windows-latest
        include:
          - node-version: 20
            os: ubuntu-latest
            experimental: true
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci
      - run: npm test
```

---

## Expressions and Contexts

### Commonly used contexts

| Context | Purpose | Example |
|---------|---------|---------|
| `github.sha` | Commit SHA | Image tagging |
| `github.ref` | Branch/tag ref | Conditional deploy |
| `github.ref_name` | Short branch name | Environment naming |
| `github.actor` | Who triggered | Notifications |
| `github.repository` | `owner/repo` | Registry paths |
| `github.event_name` | Trigger type | Conditional logic |
| `github.run_id` | Unique run ID | Artifact naming |
| `secrets.GITHUB_TOKEN` | Auto-generated token | API auth |

### String functions

```yaml
if: contains(github.event.head_commit.message, '[skip ci]')
if: startsWith(github.ref, 'refs/tags/')
if: endsWith(github.repository, '-template')
if: github.event.pull_request.title != ''
```

### Status check functions

```yaml
steps:
  - name: Always run cleanup
    if: always()
    run: ./cleanup.sh

  - name: Run on failure only
    if: failure()
    run: ./notify-failure.sh

  - name: Run on success only
    if: success()
    run: ./celebrate.sh
```

---

## Outputs and Job Communication

### Passing data between steps

```yaml
steps:
  - name: Set version
    id: version
    run: echo "value=$(cat VERSION)" >> "$GITHUB_OUTPUT"

  - name: Use version
    run: echo "Version is ${{ steps.version.outputs.value }}"
```

### Passing data between jobs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
    steps:
      - name: Build metadata
        id: meta
        run: echo "tags=myapp:${{ github.sha }}" >> "$GITHUB_OUTPUT"

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.image_tag }}"
```

---

## Artifacts

### Upload artifacts

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/
    retention-days: 7
    if-no-files-found: error
```

### Download artifacts

```yaml
- uses: actions/download-artifact@v4
  with:
    name: build-output
    path: ./dist
```

---

## Environment Variables

### Setting env vars at different levels

```yaml
env:                              # Workflow-level
  CI: true
  NODE_ENV: test

jobs:
  build:
    env:                          # Job-level
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
    runs-on: ubuntu-latest
    steps:
      - name: Build
        env:                      # Step-level (highest precedence)
          API_KEY: ${{ secrets.API_KEY }}
        run: npm run build
```

### Dynamic environment variables

```yaml
- name: Set dynamic env
  run: |
    echo "BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)" >> "$GITHUB_ENV"
    echo "SHORT_SHA=${GITHUB_SHA::7}" >> "$GITHUB_ENV"

- name: Use dynamic env
  run: echo "Built on $BUILD_DATE at commit $SHORT_SHA"
```

---

## Services (sidecar containers)

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        ports:
          - 6379:6379
    steps:
      - uses: actions/checkout@v4
      - run: npm test
        env:
          DATABASE_URL: postgres://postgres:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
```
