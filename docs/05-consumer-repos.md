# Consumer Repos & Real-World Usage

This section documents each consumer repository that integrates with the shared workflows, demonstrating real-world usage across different languages, frameworks, and CI/CD patterns.

---

## Overview

| Repo | Language | Framework | Workflows Used | CI Complexity | LOC in ci.yml |
|---|---|---|---|---|---|
| **sample-app-python** | Python 3.12 | FastAPI + SQLAlchemy | CI + Integration CI + Release | High — services, Docker, deploy stages | ~40 lines |
| **sample-app-node** | Node.js 20 | Express + TypeScript | CI + Release | Standard — lint, test, security | ~15 lines |
| **sample-app-go** | Go 1.24 | gorilla/mux | CI + Release | Standard — lint, test, security | ~15 lines |
| **sample-lib-node** | Node.js 18/20/22 | Zero-dep HTTP client | Matrix CI + Publish + Release | High — 18 parallel jobs, package publishing | ~30 lines |

---

## 1. sample-app-python

**Type:** Web Application  
**Stack:** FastAPI + SQLAlchemy + PostgreSQL  
**Workflows:** `reusable-ci` + `reusable-integration-ci` + `reusable-release`

### Application

A FastAPI REST API with CRUD endpoints and database integration:

| Endpoint | Description |
|---|---|
| `GET /health` | Health check — `{"status": "ok", "version": "x.y.z"}` |
| `GET /api/greet?name=X` | Greeting — `{"message": "Hello, X!"}` |
| `POST /api/items` | Create item (DB-backed) |
| `GET /api/items` | List items with pagination |
| `GET /api/items/{id}` | Get item by ID |
| `PUT /api/items/{id}` | Update item |
| `DELETE /api/items/{id}` | Delete item |

### Database Strategy

- **Local development:** SQLite (`sqlite:///./app.db`)
- **CI environment:** PostgreSQL 16 (via service containers)
- Controlled by `DATABASE_URL` environment variable

### Test Categories

Tests are organized by pytest markers for selective execution in CI:

```python
@pytest.mark.sanity       # Quick smoke tests — run first
@pytest.mark.regression   # Full CRUD coverage, edge cases
@pytest.mark.performance  # Response time, throughput validation
```

| Test File | Marker | Tests | What It Covers |
|---|---|---|---|
| `test_sanity.py` | `sanity` | 8 tests | Health, greet, basic CRUD reachability |
| `test_regression.py` | `regression` | 20+ tests | All CRUD ops, pagination, validation, edge cases |
| `test_performance.py` | `performance` | 6 tests | Response time limits, sequential throughput |
| `test_app.py` | (none) | 4 tests | Original test suite, backward-compatible |

### CI Configuration (ci.yml)

This is the most advanced consumer — it calls **two** reusable workflows:

```yaml
name: CI
on:
  push:
    branches: [main]
    paths: ["src/**", "tests/**", "requirements*.txt", "Dockerfile"]
  pull_request_target:
    branches: [main]
  schedule:
    - cron: "0 3 * * 1-5"    # Nightly regression
  workflow_dispatch:
    inputs:
      python_version:
        type: choice
        default: "3.12"
        options: ["3.11", "3.12", "3.13"]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # Standard CI
  ci:
    uses: .../reusable-ci.yml@main
    with:
      language: python
      language_version: "3.12"

  # Integration CI (service containers + Docker + deploy)
  integration:
    uses: .../reusable-integration-ci.yml@main
    with:
      language: python
      language_version: "3.12"
      language_versions: '["3.11", "3.12", "3.13"]'
      enable_sanity: true
      enable_regression: true
      enable_performance: true
      enable_docker_build: true
      docker_image_name: "sample-app-python"
```

### GHA Features Demonstrated

- Path-based triggering (`on.push.paths`)
- Scheduled builds (nightly cron)
- Parameterized builds (`workflow_dispatch.inputs` with type: choice)
- Concurrency controls (`cancel-in-progress`)
- Service containers (PostgreSQL + Redis)
- Docker build & push to GHCR
- Parallel test stages (sanity, regression, performance)
- Environment deployment gates (staging → production)

---

## 2. sample-app-node

**Type:** Web Application  
**Stack:** Express + TypeScript  
**Workflows:** `reusable-ci` + `reusable-release`

### Application

A minimal Express + TypeScript app:

```typescript
app.get("/health", (_req, res) => {
  res.json({ status: "ok", version: VERSION });
});

app.get("/api/greet", (req, res) => {
  const name = (req.query.name as string) || "World";
  res.json({ message: `Hello, ${name}!` });
});
```

### CI Configuration

The simplest consumer — just **15 lines** of workflow YAML:

```yaml
name: CI
on: [push, pull_request]

jobs:
  ci:
    uses: .../reusable-ci.yml@main
    with:
      language: node
      language_version: "20"
      enable_lint: true
      enable_test: true
      enable_security_scan: true
    secrets:
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 3. sample-app-go

**Type:** Web Application  
**Stack:** Go + gorilla/mux  
**Workflows:** `reusable-ci` + `reusable-release`

### Application

A minimal Go HTTP server:

```go
func healthHandler(w http.ResponseWriter, r *http.Request) {
    json.NewEncoder(w).Encode(HealthResponse{Status: "ok", Version: version})
}

func greetHandler(w http.ResponseWriter, r *http.Request) {
    name := r.URL.Query().Get("name")
    if name == "" { name = "World" }
    json.NewEncoder(w).Encode(GreetResponse{Message: fmt.Sprintf("Hello, %s!", name)})
}
```

### CI Configuration

Same minimal pattern — demonstrates Go language support:

```yaml
jobs:
  ci:
    uses: .../reusable-ci.yml@main
    with:
      language: go
      language_version: "1.24"
      enable_lint: true
      enable_test: true
      enable_security_scan: true
```

---

## 4. sample-lib-node

**Type:** Library (npm package)  
**Stack:** TypeScript, zero runtime dependencies  
**Workflows:** `reusable-matrix-ci` + `reusable-publish` + `reusable-release`

### Library — `@httpkit/client`

A typed HTTP client with built-in retry:

```typescript
const client = new HttpClient({
  baseURL: 'https://api.example.com',
  timeout: 5000,
  retries: 3,
  retryDelay: 1000,
});

const response = await client.get<User[]>('/users');
// response.data: User[]
// response.status: 200
```

**Features:** Generic types, exponential backoff with jitter, per-request overrides, cross-platform (Node 18/20/22 × Linux/macOS/Windows).

### CI Configuration — Matrix Build

```yaml
jobs:
  ci:
    uses: .../reusable-matrix-ci.yml@main
    with:
      language: node
      language_versions: '["18", "20", "22"]'
      os_matrix: '["ubuntu-latest", "macos-latest", "windows-latest"]'
      test_types: '["unit", "integration"]'
      enable_build: true
      build_script: "npm run build"
      enable_security: true
      enable_lint: true
      coverage_threshold: 80
      fail_fast: false
```

This generates **18 parallel test jobs** (3 versions × 3 OSes × 2 test types).

### Publish Configuration

```yaml
jobs:
  publish:
    uses: .../reusable-publish.yml@main
    with:
      language: node
      language_version: "20"
      registry: github
      publish_staging: true      # @next tag
      publish_production: true   # @latest tag (manual approval)
      build_script: "npm run build"
```

### GHA Features Demonstrated

- Dynamic matrix via `fromJSON()` (3 axes)
- Cross-platform testing (Ubuntu, macOS, Windows)
- Build verification step
- Package publishing with staged rollout
- GitHub Environments with approval gates
- Path-based triggering
- Parameterized builds with choice dropdowns

---

## Consumer Adoption Summary

Adding shared workflows to a new repository takes **5 minutes**:

1. Create `.github/workflows/ci.yml` (~15 lines)
2. Optionally add `release.yml` (~15 lines)
3. Add `SLACK_WEBHOOK_URL` secret
4. Push — CI runs automatically

No install, no configuration files, no dependencies. The central repo owns all CI/CD logic.
