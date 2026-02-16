# GHA Platform Capabilities

This document maps every GitHub Actions feature exercised in this project — demonstrating deep, hands-on coverage across the platform.

---

## Core GHA Features

| GHA Capability | What We Demonstrated |
|---|---|
| **Reusable Workflows** (`workflow_call`) | Central repo with 5 shared workflows |
| **Composite Actions** (`action.yml`) | `setup-toolchain`, `slack-notify` |
| **Workflow files** (`.github/workflows/`) | Consumer repo CI configs (~15 lines each) |
| **Typed Inputs** | All reusable workflows accept parameterized inputs |
| **Secrets passthrough** | Secure credential handling across workflows |
| **GitHub Pages** | Live dashboard with cross-repo metrics |
| **Jobs + Steps** | Multi-stage pipelines with DAG dependencies |

---

## Build & Test Capabilities

| GHA Feature | How We Used It |
|---|---|
| `strategy.matrix` + `fromJSON()` | Dynamic multi-axis testing (18 parallel jobs) |
| Multiple jobs with `needs:` DAG | Parallel + sequential stages |
| `services:` block | PostgreSQL + Redis as native job services |
| `strategy.fail-fast: false` | Complete all matrix jobs even if one fails |
| `actions/upload-artifact` / `download-artifact` | Share build outputs between jobs |
| `if: always()` + artifact upload | Always collect test results |
| `timeout-minutes` | Prevent runaway jobs |
| `setup-*` actions with matrix versions | Multi-version testing (Python 3.12, Node 18/20/22, Go 1.24) |

---

## Triggers & Pipeline Control

| GHA Feature | How We Used It |
|---|---|
| `schedule` with cron syntax | Nightly CI runs |
| `workflow_dispatch` with typed inputs | Manual runs with parameters |
| `on.push.paths` | Skip CI for irrelevant changes (docs-only, etc.) |
| `concurrency` groups | Auto-cancel redundant runs |
| `on.push.branches` | Branch-specific triggers |
| `$GITHUB_STEP_SUMMARY` | Rich CI summaries in Actions UI |

---

## Deployment & Release

| GHA Feature | How We Used It |
|---|---|
| `environment:` with protection rules | Approval gates for production deploys |
| GitHub Environments (staging → production) | Staged promotions with separate secrets |
| `docker/build-push-action` with BuildX | Container build + push to GHCR |
| npm publish / GitHub Packages | Package publishing with staging tags (@next → @latest) |
| `concurrency:` groups | Deployment locking (one deploy at a time) |

---

## Security & Credentials

| GHA Feature | How We Used It |
|---|---|
| **GitHub Secrets** (repo/org/environment-level) | Secure credential storage and scoping |
| **`secrets:` passthrough** | Forward secrets to reusable workflows securely |
| **Environment-scoped secrets** | Different credentials for staging vs production |
| **`GITHUB_TOKEN`** | Auto-scoped, auto-rotated per-job permissions |
| **Trivy scanning** | Automated vulnerability scanning on every CI run |
| **Dependabot** | Automated dependency update PRs |

---

## GHA-Native Advantages

These features are available in GitHub Actions **natively** — no plugins, no extra infrastructure:

| Feature | What It Does |
|---|---|
| **`cancel-in-progress`** | Auto-cancel redundant runs when a new push arrives |
| **Dependabot** | Native dependency update PRs — zero configuration |
| **Hosted runners** | Free ubuntu/macos/windows — zero infrastructure |
| **`GITHUB_TOKEN`** | Auto-scoped, auto-rotated, per-job permissions |
| **GitHub Environments** | UI approvals + deployment history + wait timers + env-specific secrets |
| **Docker layer cache (GHA)** | `cache-from: type=gha` — shared across workflow runs |
| **Path-based triggers** | `on.push.paths` — skip CI when irrelevant files change |
| **Dynamic matrix `fromJSON()`** | Generate matrix axes at runtime from inputs |
| **Immutable logs** | GitHub-hosted, tamper-proof audit trail |
| **Starter workflows** | Org-wide templates in `.github` repo |

---

## Onboarding Example

A new repository gets full CI/CD by adding a single workflow file:

- Create `.github/workflows/ci.yml` (~15 lines)
- Add release workflow if needed (~15 lines)
- Full CI/CD pipeline in under 5 minutes

### Example: Consumer CI Configuration

```yaml
name: CI
on: [push, pull_request]

jobs:
  ci:
    uses: org/shared-workflows/.github/workflows/reusable-ci.yml@main
    with:
      language: python
      language_version: "3.12"
      enable_lint: true
      enable_security_scan: true
    secrets:
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

**Result:** ~15 lines of YAML gives you lint, test, security scan, Slack notification, and CI summary — all from the shared workflow.
