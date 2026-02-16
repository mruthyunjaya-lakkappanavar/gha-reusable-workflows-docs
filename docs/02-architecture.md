# Architecture

## System Overview

The platform follows a **hub-and-spoke** architecture. The central `github-shared-workflows` repository is the hub, containing all reusable workflows, composite actions, and the live dashboard. Consumer repositories (the spokes) call these workflows using GitHub's native `workflow_call` trigger.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    github-shared-workflows (Hub)                        │
│                                                                         │
│   .github/workflows/                    actions/                        │
│   ┌────────────────────────┐            ┌─────────────────────┐        │
│   │ reusable-ci.yml        │            │ setup-toolchain/    │        │
│   │  Lint → Test → Scan    │            │  Python/Node/Go     │        │
│   ├────────────────────────┤            │  + dep caching      │        │
│   │ reusable-matrix-ci.yml │            ├─────────────────────┤        │
│   │  Version × OS × Tests  │            │ slack-notify/       │        │
│   ├────────────────────────┤            │  Color-coded msgs   │        │
│   │ reusable-integration-ci│            └─────────────────────┘        │
│   │  Services + Docker +   │                                            │
│   │  Staged Deploy         │            dashboard/                      │
│   ├────────────────────────┤            ┌─────────────────────┐        │
│   │ reusable-publish.yml   │            │ GitHub Pages        │        │
│   │  Staging → Production  │            │ Cross-repo status   │        │
│   ├────────────────────────┤            │ Health metrics      │        │
│   │ reusable-release.yml   │            │ Activity timeline   │        │
│   │  Semver + Changelog    │            └─────────────────────┘        │
│   └────────────────────────┘                                            │
└───────────┬───────────────┬────────────────┬───────────────┬────────────┘
            │               │                │               │
     workflow_call    workflow_call    workflow_call    workflow_call
            │               │                │               │
  ┌─────────▼──────┐ ┌─────▼────────┐ ┌─────▼──────┐ ┌─────▼──────────┐
  │ sample-app-    │ │ sample-app-  │ │ sample-app-│ │ sample-lib-    │
  │ python         │ │ node         │ │ go         │ │ node           │
  │                │ │              │ │            │ │                │
  │ FastAPI +      │ │ Express + TS │ │ Go + Mux   │ │ HTTP Client    │
  │ SQLAlchemy     │ │ Jest         │ │ go test    │ │ Library        │
  │ PostgreSQL     │ │ ESLint       │ │ golangci   │ │ Zero deps      │
  │ Docker         │ │              │ │            │ │                │
  │                │ │ Uses:        │ │ Uses:      │ │ Uses:          │
  │ Uses:          │ │ • CI         │ │ • CI       │ │ • Matrix CI    │
  │ • CI           │ │ • Release    │ │ • Release  │ │ • Publish      │
  │ • Integration  │ │              │ │            │ │ • Release      │
  │ • Release      │ │              │ │            │ │                │
  └────────────────┘ └──────────────┘ └────────────┘ └────────────────┘
         │                  │                │               │
         └──────────────────┴────────────────┴───────────────┘
                                     │
                              ┌──────▼──────┐
                              │    Slack     │
                              │  #builds    │
                              │  #releases  │
                              └─────────────┘
```

## Repository Structure

```
github-shared-workflows/
├── .github/
│   ├── workflows/
│   │   ├── reusable-ci.yml              # Standard CI pipeline
│   │   ├── reusable-matrix-ci.yml       # Matrix CI (multi-version × OS × test type)
│   │   ├── reusable-integration-ci.yml  # Services + Docker + Deploy
│   │   ├── reusable-publish.yml         # Package publishing with env gates
│   │   ├── reusable-release.yml         # Semantic release pipeline
│   │   └── update-dashboard.yml         # Dashboard data updater (scheduled)
│   └── dependabot.yml                   # Automated dependency updates
├── actions/
│   ├── setup-toolchain/action.yml       # Composite: Python/Node/Go + caching
│   └── slack-notify/action.yml          # Composite: Slack notifications
├── dashboard/                           # GitHub Pages dashboard
│   ├── index.html
│   ├── style.css
│   ├── app.js
│   └── data/                            # Pre-generated static JSON data
└── docs/
    ├── ARCHITECTURE.md
    └── USAGE.md
```

## Design Principles

### 1. Reusable Workflows vs Composite Actions

We use **both**, each for its ideal purpose:

| Type | Use Case | Example |
|---|---|---|
| **Reusable Workflow** (`workflow_call`) | Full CI/CD pipelines with multiple jobs | `reusable-ci.yml`, `reusable-publish.yml` |
| **Composite Action** (`uses: ./actions/...`) | Atomic, reusable steps within a job | `setup-toolchain`, `slack-notify` |

**Why?** Reusable workflows run on their own runners with full `jobs` context, secret forwarding, and conditional job execution. Composite actions are lightweight steps that run within an existing job.

### 2. Public Repository

The central repo is **public** because GitHub requires callers to have access to the workflow file. Options:
- **Public** — any GitHub repo can call (chosen for maximum portability)
- **Internal** — same GitHub Enterprise org only
- **Private** — requires GitHub Enterprise (paid)

### 3. Zero-Dependency Dashboard

The live dashboard uses **vanilla HTML/CSS/JS** — no framework, no build tools, no bundler. This means:
- GitHub Pages hosts it directly
- Anyone can read and modify the code
- Zero maintenance burden from dependency updates

### 4. Static Data over Live API

The dashboard uses **pre-generated static JSON** instead of live GitHub API calls. A scheduled workflow refreshes the data every 6 hours. This eliminates API rate-limiting for visitors and ensures the dashboard always loads fast.

## Data Flow Diagrams

### CI Pipeline Flow

```
Developer pushes code
  └→ Consumer ci.yml triggers (on: push/pull_request)
      └→ Calls reusable-ci.yml via workflow_call
          ├→ setup-toolchain composite action
          │   └→ Installs Python/Node/Go + configures dep cache
          ├→ Install dependencies (pip install / npm ci)
          ├→ Lint (flake8 / ESLint / golangci-lint)
          ├→ Test (pytest / jest / go test)
          ├→ Security scan (Trivy filesystem scan)
          ├→ Upload test results as artifacts
          └→ Slack notification (success/failure)
```

### Release Flow

```
PR merged with conventional commit → main branch
  └→ Consumer release.yml triggers
      └→ Calls reusable-release.yml
          ├→ Release Please analyzes commits
          ├→ Creates/updates release PR (version bump + changelog)
          └→ On release PR merge:
              ├→ Creates GitHub Release with release notes
              └→ Slack notification to #releases
```

### Matrix CI Flow

```
Developer pushes code
  └→ Consumer ci.yml triggers
      └→ Calls reusable-matrix-ci.yml
          ├→ Lint job (single runner)
          ├→ Matrix test job expands via fromJSON():
          │   ┌─ Node 18 × ubuntu  × unit
          │   ├─ Node 18 × macos   × unit
          │   ├─ Node 20 × ubuntu  × integration
          │   ├─ ... (N × M × K parallel jobs)
          │   └─ Node 22 × windows × integration
          ├→ Security scan job
          ├→ Build verification job (optional)
          └→ Summary job aggregates results → PR comment
```

### Integration CI Flow

```
Developer pushes code (sample-app-python)
  └→ Consumer ci.yml triggers
      └→ Calls reusable-integration-ci.yml
          ├→ Parallel test stages:
          │   ├─ Sanity tests (PostgreSQL + Redis service containers)
          │   ├─ Regression tests (version matrix × PostgreSQL)
          │   └─ Performance tests (throughput + response time)
          ├→ Docker build job
          │   └→ docker/build-push-action → push to GHCR
          ├→ Deploy staging (environment: staging)
          └→ Deploy production (environment: production, manual approval)
```

### Publish Flow

```
Manual dispatch or release tag
  └→ Consumer publish.yml triggers
      └→ Calls reusable-publish.yml
          ├→ Build package (npm pack / python -m build)
          ├→ Publish staging (@next tag / test.pypi.org)
          │   └→ Environment: staging
          └→ Publish production (@latest tag / pypi.org)
              └→ Environment: production (manual approval required)
```

## Security Considerations

| Aspect | Implementation |
|---|---|
| **Dependency scanning** | Trivy filesystem scan on every CI run |
| **Secret management** | GitHub Secrets with `secrets:` passthrough (never exposed in logs) |
| **Dependency updates** | Dependabot configured for automated PR-based updates |
| **Immutable logs** | GitHub-hosted, tamper-proof audit trail |
| **Scoped tokens** | `GITHUB_TOKEN` auto-scoped per-job with minimal permissions |
