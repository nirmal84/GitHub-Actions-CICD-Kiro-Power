# Reusable Workflows and Custom Actions Guide

## Reusable Workflows

### Creating a reusable workflow

File: `.github/workflows/reusable-build.yml`

```yaml
name: Reusable Build

on:
  workflow_call:
    inputs:
      node-version:
        description: 'Node.js version'
        required: false
        type: string
        default: '20'
      environment:
        description: 'Target environment'
        required: true
        type: string
    secrets:
      DEPLOY_TOKEN:
        description: 'Deployment token'
        required: true
    outputs:
      artifact-name:
        description: 'Name of the build artifact'
        value: ${{ jobs.build.outputs.artifact-name }}

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.meta.outputs.name }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - name: Set artifact name
        id: meta
        run: echo "name=build-${{ inputs.environment }}-${{ github.sha }}" >> "$GITHUB_OUTPUT"
      - uses: actions/upload-artifact@v4
        with:
          name: ${{ steps.meta.outputs.name }}
          path: dist/
```

### Calling a reusable workflow

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]

jobs:
  build-staging:
    uses: ./.github/workflows/reusable-build.yml
    with:
      node-version: '20'
      environment: staging
    secrets:
      DEPLOY_TOKEN: ${{ secrets.STAGING_TOKEN }}

  build-production:
    needs: build-staging
    uses: ./.github/workflows/reusable-build.yml
    with:
      node-version: '20'
      environment: production
    secrets:
      DEPLOY_TOKEN: ${{ secrets.PROD_TOKEN }}
```

### Cross-repository reusable workflows

```yaml
jobs:
  shared-ci:
    uses: my-org/shared-workflows/.github/workflows/node-ci.yml@v1
    with:
      node-version: '20'
    secrets: inherit    # Pass all secrets from caller
```

### Reusable workflow limitations

- Maximum 4 levels of nesting (workflow → reusable → reusable → reusable)
- Cannot call another reusable workflow from within a reusable workflow's `uses` step
- Environment variables set at the caller level are not propagated
- The `env` context is not available in reusable workflow `with` inputs
- Maximum 20 unique reusable workflows per workflow file

---

## Composite Actions

### Creating a composite action

File: `.github/actions/setup-and-test/action.yml`

```yaml
name: 'Setup and Test'
description: 'Install dependencies and run tests'

inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '20'
  test-command:
    description: 'Test command to run'
    required: false
    default: 'npm test'

outputs:
  coverage:
    description: 'Test coverage percentage'
    value: ${{ steps.coverage.outputs.percentage }}

runs:
  using: 'composite'
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'

    - name: Install dependencies
      shell: bash
      run: npm ci

    - name: Run tests
      shell: bash
      run: ${{ inputs.test-command }}

    - name: Extract coverage
      id: coverage
      shell: bash
      run: |
        COVERAGE=$(npm run test:coverage -- --json | jq '.total.lines.pct')
        echo "percentage=$COVERAGE" >> "$GITHUB_OUTPUT"
```

### Using a composite action

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-and-test
        id: test
        with:
          node-version: '20'
      - run: echo "Coverage is ${{ steps.test.outputs.coverage }}%"
```

### Composite action best practices

1. **Always specify `shell`** in every `run` step — composite actions don't inherit the default shell
2. **Use `$GITHUB_OUTPUT`** for outputs, not the deprecated `::set-output` command
3. **Keep actions focused** — one action per responsibility
4. **Document inputs/outputs** clearly in `action.yml`
5. **Version with tags** if sharing across repos (`@v1`, `@v2`)

---

## JavaScript/TypeScript Actions

### When to use JavaScript actions

- Need to interact with GitHub APIs extensively
- Complex logic that's hard to express in shell
- Want to publish to the GitHub Marketplace
- Need cross-platform support (Windows, macOS, Linux)

### Basic JavaScript action structure

```
my-action/
├── action.yml
├── index.js
├── package.json
└── node_modules/     # Must be committed
```

### action.yml for JavaScript action

```yaml
name: 'My Custom Action'
description: 'Does something useful'

inputs:
  github-token:
    description: 'GitHub token'
    required: true
  label:
    description: 'Label to apply'
    required: false
    default: 'automated'

outputs:
  result:
    description: 'The action result'

runs:
  using: 'node20'
  main: 'index.js'
```

---

## Docker Container Actions

### When to use Docker actions

- Need specific tools or dependencies not available on runners
- Want consistent environments across all platforms
- Complex build environments

### Docker action structure

```yaml
# action.yml
name: 'Security Scan'
description: 'Run security scanning tools'

inputs:
  scan-path:
    description: 'Path to scan'
    default: '.'

runs:
  using: 'docker'
  image: 'Dockerfile'
  args:
    - ${{ inputs.scan-path }}
```

---

## Workflow Composition Patterns

### Pattern 1: Shared CI + environment-specific deploy

```
.github/
├── workflows/
│   ├── ci.yml                    # Calls reusable-test.yml
│   ├── deploy-staging.yml        # Calls reusable-deploy.yml
│   ├── deploy-production.yml     # Calls reusable-deploy.yml
│   ├── reusable-test.yml         # Shared test logic
│   └── reusable-deploy.yml       # Shared deploy logic
├── actions/
│   ├── setup-project/            # Composite: install deps
│   │   └── action.yml
│   └── notify/                   # Composite: send notifications
│       └── action.yml
```

### Pattern 2: Monorepo with shared actions

```
.github/
├── workflows/
│   ├── api-ci.yml
│   ├── web-ci.yml
│   └── shared-ci.yml             # Reusable workflow
├── actions/
│   ├── detect-changes/           # Which packages changed
│   │   └── action.yml
│   └── setup-workspace/          # Install all deps
│       └── action.yml
packages/
├── api/
└── web/
```

### Pattern 3: Organization-wide shared workflows

```
# In my-org/shared-workflows repo:
.github/
└── workflows/
    ├── node-ci.yml
    ├── docker-build.yml
    ├── security-scan.yml
    └── deploy-ecs.yml

# In any org repo:
jobs:
  ci:
    uses: my-org/shared-workflows/.github/workflows/node-ci.yml@v1
```
