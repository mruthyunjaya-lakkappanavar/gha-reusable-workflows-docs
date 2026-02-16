# Advanced GHA Patterns

Each pattern below is implemented in the reusable workflows and demonstrates depth of GitHub Actions expertise.

---

## 1. Dynamic Matrix Builds with `fromJSON()`

**Pattern:** Accept matrix axes as JSON-string inputs, expand at runtime with `fromJSON()`.

**Why it matters:** Consumers control the matrix dimensions without modifying the shared workflow. This gives consumers full control over test dimensions without modifying the shared workflow.

```yaml
# In the reusable workflow:
strategy:
  fail-fast: ${{ inputs.fail_fast }}
  matrix:
    version: ${{ fromJSON(inputs.language_versions) }}
    os: ${{ fromJSON(inputs.os_matrix) }}
    test_type: ${{ fromJSON(inputs.test_types) }}

# Consumer passes:
with:
  language_versions: '["18", "20", "22"]'
  os_matrix: '["ubuntu-latest", "macos-latest", "windows-latest"]'
  test_types: '["unit", "integration"]'
```

**Result:** 3 × 3 × 2 = 18 parallel jobs, fully controlled by the consumer.

---

## 2. Service Containers

**Pattern:** Provision database and cache services alongside test jobs using GHA's native `services:` block.

**Why it matters:** No Docker-in-Docker required. The runner manages container lifecycle, networking (localhost ports), and health checks automatically.

```yaml
services:
  postgres:
    image: postgres:16
    env:
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
      POSTGRES_DB: testdb
    ports: ["5432:5432"]
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    options: --health-cmd "redis-cli ping"
```

The application connects via `DATABASE_URL=postgresql://testuser:testpass@localhost:5432/testdb`.

---

## 3. Environment Approval Gates

**Pattern:** Use GitHub Environments with required reviewers for staged deployments.

**Why it matters:** Provides a UI-based approval system with deployment history, environment-specific secrets, and wait timers — all native to GHA.

```yaml
jobs:
  publish-production:
    environment: production    # ← Requires reviewer approval
    runs-on: ubuntu-latest
    steps:
      - name: Publish to production
        run: npm publish --tag latest
```

**Setup:**
1. Create environment in repo Settings → Environments
2. Add required reviewers (team leads)
3. Optionally add wait timers and deployment branch restrictions

---

## 4. Concurrency Controls

**Pattern:** Prevent redundant builds and auto-cancel superseded runs.

**Why it matters:** Built into GHA at the workflow level — no plugins or external infrastructure required.

```yaml
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

**Behavior:** If a developer pushes 3 commits in quick succession, only the latest run continues — the first two are automatically cancelled.

---

## 5. Path-Based Triggering

**Pattern:** Only trigger CI when relevant files change.

**Why it matters:** Saves runner minutes and provides faster feedback. Changes to README or docs don't trigger the full test suite.

```yaml
on:
  push:
    branches: [main]
    paths:
      - "src/**"
      - "tests/**"
      - "requirements*.txt"
      - "Dockerfile"
      - ".github/workflows/ci.yml"
```

---

## 6. Parameterized Builds

**Pattern:** Use `workflow_dispatch.inputs` with typed controls (choice, boolean, string).

**Why it matters:** Provides UI-driven dropdown and checkbox controls for manual workflow runs.

```yaml
on:
  workflow_dispatch:
    inputs:
      python_version:
        description: "Primary Python version"
        type: choice
        default: "3.12"
        options: ["3.11", "3.12", "3.13"]
      enable_performance:
        description: "Run performance tests"
        type: boolean
        default: true
```

---

## 7. Artifact Passing Between Jobs

**Pattern:** Upload artifacts in one job, download them in dependent jobs. Used for build-once-deploy-many patterns.

```yaml
jobs:
  build:
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: package-${{ steps.version.outputs.version }}
          path: dist/

  publish-staging:
    needs: [build]
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: package-${{ needs.build.outputs.package_version }}
```

---

## 8. Docker Build with GHA Layer Cache

**Pattern:** Use BuildX with GHA cache backend for fast Docker builds.

```yaml
- uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ghcr.io/${{ github.repository }}:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

**Why it matters:** Layer cache is shared across workflow runs. Subsequent builds only rebuild changed layers, dramatically reducing build time.

---

## 9. Composite Action Patterns

**Pattern:** Encapsulate multi-step logic into reusable actions with conditional branching.

```yaml
runs:
  using: "composite"
  steps:
    - name: Setup Python
      if: inputs.language == 'python'
      uses: actions/setup-python@v5
      with:
        python-version: ${{ inputs.language_version }}
        cache: ${{ inputs.cache == 'true' && 'pip' || '' }}

    - name: Setup Node.js
      if: inputs.language == 'node'
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.language_version }}
        cache: ${{ inputs.cache == 'true' && 'npm' || '' }}
```

This single action handles 3 languages via conditional steps — clean, maintainable, and testable.

---

## 10. Job Summaries

**Pattern:** Write rich markdown content to `$GITHUB_STEP_SUMMARY` for inline CI reports.

```bash
{
  echo "## Test Results"
  echo "| Metric | Value |"
  echo "|---|---|"
  echo "| Total | $TOTAL |"
  echo "| Passed | $PASSED |"
  echo "| Failed | $FAILED |"
  echo "| Coverage | ${COVERAGE}% |"
} >> "$GITHUB_STEP_SUMMARY"
```

This creates a formatted summary visible directly in the GitHub Actions UI — no external dashboards needed for per-run details.

---

## 11. Dependabot (GHA-Exclusive)

**Pattern:** Automated dependency update PRs with zero configuration.

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
```

**Why it matters:** Dependabot automatically creates PRs to update dependencies, which then trigger the CI workflow — creating a fully automated update-test-merge loop.
