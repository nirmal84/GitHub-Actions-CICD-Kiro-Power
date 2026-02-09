# Deployment Patterns Guide

## GitHub Environments

### Basic environment setup

```yaml
jobs:
  deploy-staging:
    environment:
      name: staging
      url: https://staging.myapp.com
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./deploy.sh staging
        env:
          DEPLOY_TOKEN: ${{ secrets.STAGING_DEPLOY_TOKEN }}

  deploy-production:
    needs: deploy-staging
    environment:
      name: production
      url: https://myapp.com
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./deploy.sh production
        env:
          DEPLOY_TOKEN: ${{ secrets.PROD_DEPLOY_TOKEN }}
```

### Environment protection rules (configure in GitHub UI)

For production environments, always enable:
1. Required reviewers (at least 1-2 team members)
2. Wait timer (optional, e.g., 5 minutes for staging validation)
3. Deployment branches (restrict to `main` only)

---

## Deployment Strategies

### Strategy 1: Push to main → Deploy

Simplest approach for smaller projects:

```yaml
name: Deploy

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm test

  deploy:
    needs: test
    environment: production
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - run: ./deploy.sh
```

### Strategy 2: Tag-based releases

Best for versioned libraries and applications:

```yaml
name: Release

on:
  push:
    tags:
      - 'v*.*.*'

permissions:
  contents: write
  packages: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: npm ci && npm run build

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
          files: |
            dist/*.tar.gz
            dist/*.zip
```

### Strategy 3: Staged rollout (staging → production)

Best for teams needing validation between environments:

```yaml
name: Deploy Pipeline

on:
  push:
    branches: [main]

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

  deploy-staging:
    needs: build
    environment:
      name: staging
      url: https://staging.myapp.com
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build
          path: dist/
      - run: ./deploy.sh staging

  smoke-test:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run test:smoke -- --base-url=https://staging.myapp.com

  deploy-production:
    needs: smoke-test
    environment:
      name: production
      url: https://myapp.com
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build
          path: dist/
      - run: ./deploy.sh production
```

### Strategy 4: Manual dispatch for production

Best for teams wanting explicit control:

```yaml
name: Deploy to Production

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version tag to deploy (e.g., v1.2.3)'
        required: true
        type: string
      confirm:
        description: 'Type "deploy" to confirm'
        required: true
        type: string

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Validate confirmation
        if: github.event.inputs.confirm != 'deploy'
        run: |
          echo "::error::Confirmation failed. Type 'deploy' to confirm."
          exit 1

  deploy:
    needs: validate
    environment: production
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.inputs.version }}
      - run: ./deploy.sh production
```

---

## Platform-Specific Deployment

### Deploy to AWS (ECS)

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    aws-region: us-east-1

- uses: aws-actions/amazon-ecr-login@v2
  id: ecr-login

- uses: docker/build-push-action@v6
  with:
    push: true
    tags: ${{ steps.ecr-login.outputs.registry }}/myapp:${{ github.sha }}

- name: Deploy to ECS
  run: |
    aws ecs update-service \
      --cluster my-cluster \
      --service my-service \
      --force-new-deployment
```

### Deploy to Vercel

```yaml
- name: Deploy to Vercel
  uses: amondnet/vercel-action@v25
  with:
    vercel-token: ${{ secrets.VERCEL_TOKEN }}
    vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
    vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
    vercel-args: '--prod'
```

### Deploy to Netlify

```yaml
- name: Deploy to Netlify
  uses: netlify/actions/cli@master
  with:
    args: deploy --prod --dir=dist
  env:
    NETLIFY_SITE_ID: ${{ secrets.NETLIFY_SITE_ID }}
    NETLIFY_AUTH_TOKEN: ${{ secrets.NETLIFY_AUTH_TOKEN }}
```

### Deploy to GitHub Pages

```yaml
permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build

      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: dist/
      - uses: actions/deploy-pages@v4
        id: deployment
```

---

## Rollback Patterns

### Automatic rollback on smoke test failure

```yaml
deploy:
  runs-on: ubuntu-latest
  steps:
    - name: Deploy new version
      id: deploy
      run: ./deploy.sh ${{ github.sha }}

    - name: Smoke test
      id: smoke
      continue-on-error: true
      run: ./smoke-test.sh

    - name: Rollback on failure
      if: steps.smoke.outcome == 'failure'
      run: |
        echo "::error::Smoke tests failed, rolling back..."
        ./deploy.sh ${{ env.PREVIOUS_SHA }}
        exit 1
```

---

## Notifications

### Slack notification on deploy

```yaml
- name: Notify Slack
  if: always()
  uses: slackapi/slack-github-action@v2
  with:
    webhook: ${{ secrets.SLACK_WEBHOOK_URL }}
    webhook-type: incoming-webhook
    payload: |
      {
        "text": "Deploy ${{ job.status }}: ${{ github.repository }}@${{ github.sha }}"
      }
```
