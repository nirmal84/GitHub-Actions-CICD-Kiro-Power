# Performance Optimization Guide

## Caching Strategies

### Dependency caching with setup actions (simplest)

Most `actions/setup-*` actions have built-in caching:

```yaml
# Node.js
- uses: actions/setup-node@v4
  with:
    node-version: 20
    cache: 'npm'          # Also supports 'yarn', 'pnpm'

# Python
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'           # Also supports 'pipenv', 'poetry'

# Go
- uses: actions/setup-go@v5
  with:
    go-version: '1.22'
    cache: true

# Rust
- uses: actions/cache@v4
  with:
    path: |
      ~/.cargo/registry
      ~/.cargo/git
      target/
    key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
    restore-keys: |
      ${{ runner.os }}-cargo-
```

### Advanced caching with actions/cache

```yaml
- uses: actions/cache@v4
  id: cache-deps
  with:
    path: node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

- name: Install dependencies
  if: steps.cache-deps.outputs.cache-hit != 'true'
  run: npm ci
```

### Docker layer caching

```yaml
- uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: myapp:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

---

## Concurrency Control

### Cancel redundant runs

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

### Protect production deploys (never cancel)

```yaml
concurrency:
  group: deploy-production
  cancel-in-progress: false    # Queue instead of cancel
```

### Per-PR concurrency

```yaml
concurrency:
  group: pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true
```

---

## Matrix Strategies

### Efficient matrix with fail-fast

```yaml
strategy:
  fail-fast: true          # Stop all jobs if one fails
  max-parallel: 3          # Limit concurrent jobs
  matrix:
    node-version: [18, 20, 22]
```

### Dynamic matrix (advanced)

```yaml
jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - id: set-matrix
        run: |
          echo 'matrix={"include":[{"project":"api","path":"packages/api"},{"project":"web","path":"packages/web"}]}' >> "$GITHUB_OUTPUT"

  build:
    needs: prepare
    strategy:
      matrix: ${{ fromJson(needs.prepare.outputs.matrix) }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cd ${{ matrix.path }} && npm ci && npm test
```

---

## Job Parallelization

### Run independent jobs in parallel

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run typecheck

  unit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test

  e2e-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run test:e2e

  # Only deploy when ALL parallel jobs pass
  deploy:
    needs: [lint, typecheck, unit-test, e2e-test]
    runs-on: ubuntu-latest
    steps:
      - run: echo "All checks passed, deploying..."
```

---

## Reducing Billable Minutes

### Use path filters to skip unnecessary runs

```yaml
on:
  push:
    paths-ignore:
      - '**.md'
      - 'docs/**'
      - '.vscode/**'
      - 'LICENSE'
```

### Use `actions/checkout` with sparse checkout

```yaml
- uses: actions/checkout@v4
  with:
    sparse-checkout: |
      src
      tests
      package.json
    sparse-checkout-cone-mode: false
```

### Skip CI with commit message

```yaml
jobs:
  build:
    if: "!contains(github.event.head_commit.message, '[skip ci]')"
```

### Use smaller runners when possible

```yaml
# For simple tasks like linting, labeling, notifications
runs-on: ubuntu-latest     # 2-core is the default and cheapest
```

---

## Artifact Optimization

### Compress before uploading

```yaml
- name: Compress build output
  run: tar -czf build.tar.gz dist/

- uses: actions/upload-artifact@v4
  with:
    name: build
    path: build.tar.gz
    retention-days: 3        # Reduce from default 90 days
    compression-level: 6     # 0 (no compression) to 9 (max)
```

### Use retention-days wisely

| Artifact Type | Recommended Retention |
|--------------|----------------------|
| PR build artifacts | 1-3 days |
| Test reports | 7 days |
| Release binaries | 30-90 days |
| Compliance/audit logs | 90 days |

---

## Timeout Configuration

### Always set timeouts

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 15        # Job-level timeout

    steps:
      - name: Long-running test
        timeout-minutes: 10    # Step-level timeout
        run: npm run test:integration
```

### Recommended timeouts

| Job Type | Timeout |
|----------|---------|
| Lint / typecheck | 5-10 min |
| Unit tests | 10-15 min |
| E2E tests | 20-30 min |
| Docker build | 15-20 min |
| Full deploy | 20-30 min |
