# GitHub Actions CI/CD — Kiro Power

Build, test, and deploy with GitHub Actions. This power gives your Kiro agent expert-level knowledge of workflow authoring, security hardening, performance optimization, and deployment strategies.

## What's Included

| Component | Description |
|-----------|-------------|
| `POWER.md` | Core power documentation with quick reference, templates, and key principles |
| `mcp.json` | GitHub MCP server configuration for repository and Actions integration |
| **Steering Files** | On-demand guides loaded based on your current task |

### Steering Files

| File | Covers |
|------|--------|
| `workflow-authoring.md` | Triggers, jobs, matrix strategies, expressions, services, artifacts |
| `security-hardening.md` | Permissions, secrets, OIDC, supply chain, script injection prevention |
| `performance-optimization.md` | Caching, concurrency, parallelization, reducing billable minutes |
| `deployment-patterns.md` | Environments, staged rollouts, platform-specific deploys, rollback |
| `reusable-actions.md` | Reusable workflows, composite actions, JavaScript/Docker actions |
| `troubleshooting.md` | Common errors, debugging techniques, validation, performance debugging |

## Installation

### Option 1: From Kiro IDE

1. Open the Powers panel in Kiro
2. Click **Import from GitHub**
3. Paste this repository URL

### Option 2: Local install

1. Clone this repository
2. In Kiro, import from the local directory path

## Setup

After installation, you'll need a **GitHub Personal Access Token** with these scopes:

- `repo` — repository access
- `workflow` — update Actions workflows
- `actions` — manage runs and artifacts

Generate one at: https://github.com/settings/tokens

## Usage

Just start talking about CI/CD in Kiro! Keywords like "github actions", "workflow", "pipeline", "deploy", "ci", "cd" will automatically activate this power.

**Example prompts:**

- "Create a CI workflow for my Node.js project"
- "Add Docker build and push to GHCR"
- "Set up a staging → production deployment pipeline"
- "Help me debug why my workflow isn't triggering"
- "Harden the security of my existing workflow"
- "Create a reusable workflow for our org"

## License

MIT
