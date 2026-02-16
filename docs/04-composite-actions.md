# Composite Actions Reference

Composite actions are lightweight, reusable step sequences that run within an existing job. They complement reusable workflows by encapsulating common atomic operations.

---

## 1. `setup-toolchain` — Language Setup + Caching

**Path:** `actions/setup-toolchain/action.yml`

**Purpose:** Unified setup for Python, Node.js, or Go with automatic dependency caching. Eliminates boilerplate that would otherwise appear in every workflow.

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `language` | Yes | — | `python`, `node`, or `go` |
| `language_version` | Yes | — | Version string (e.g., `3.11`, `20`, `1.24`) |
| `cache` | No | `true` | Enable dependency caching |

### What It Does

```
language = python?                    language = node?                    language = go?
  │                                      │                                   │
  ├→ actions/setup-python@v5              ├→ actions/setup-node@v4            ├→ actions/setup-go@v5
  ├→ pip cache enabled                    ├→ npm cache enabled                ├→ Go module cache
  └→ Verify: python --version            └→ Verify: node --version           └→ Verify: go version
```

### Implementation

```yaml
name: "Setup Toolchain"
description: "Setup Python, Node.js, or Go with dependency caching"

inputs:
  language:
    description: "Programming language: python, node, or go"
    required: true
  language_version:
    description: "Language version (e.g., 3.11, 20, 1.22)"
    required: true
  cache:
    description: "Enable dependency caching"
    required: false
    default: "true"

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

    - name: Setup Go
      if: inputs.language == 'go'
      uses: actions/setup-go@v5
      with:
        go-version: ${{ inputs.language_version }}
        cache: ${{ inputs.cache == 'true' }}
```

### Usage in Reusable Workflows

```yaml
- name: Setup toolchain
  uses: mruthyunjaya-lakkappanavar/github-shared-workflows/actions/setup-toolchain@main
  with:
    language: ${{ inputs.language }}
    language_version: ${{ inputs.language_version }}
```

---

## 2. `slack-notify` — CI/CD Notifications

**Path:** `actions/slack-notify/action.yml`

**Purpose:** Send color-coded Slack notifications for CI/CD events. Uses Slack's Block Kit for rich formatting.

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `status` | Yes | — | `success`, `failure`, or `release` |
| `webhook_url` | Yes | — | Slack incoming webhook URL |
| `repo_name` | Yes | — | Repository name |
| `run_url` | Yes | — | Link to the workflow run |
| `message` | No | `""` | Optional custom message |

### Notification Styles

| Status | Color | Icon | Title |
|---|---|---|---|
| `success` | Green `#2eb886` | ✅ | CI Passed |
| `failure` | Red `#ff0000` | 🔴 | CI Failed |
| `release` | Blue `#0076D7` | 🚀 | New Release |

### Message Format

Each notification includes:
- **Icon + Title** — Visual status at a glance
- **Repo name** — Which repository
- **Branch** — Which branch triggered the build
- **Commit SHA** — Which commit
- **Author** — Who triggered it
- **Details** — Custom message (e.g., release notes)
- **View Run button** — Direct link to the GitHub Actions run

### Slack Block Kit Payload

```json
{
  "attachments": [{
    "color": "#ff0000",
    "blocks": [
      {
        "type": "section",
        "text": {
          "type": "mrkdwn",
          "text": "🔴 *CI Failed*\n*Repo:* sample-app-python\n*Branch:* main\n*Commit:* `abc1234`\n*Author:* @developer"
        }
      },
      {
        "type": "actions",
        "elements": [{
          "type": "button",
          "text": { "type": "plain_text", "text": "View Run" },
          "url": "https://github.com/..."
        }]
      }
    ]
  }]
}
```

### Usage in Reusable Workflows

```yaml
- name: Notify Slack on failure
  if: failure() && env.SLACK_WEBHOOK_URL != ''
  uses: mruthyunjaya-lakkappanavar/github-shared-workflows/actions/slack-notify@main
  with:
    status: failure
    webhook_url: ${{ secrets.SLACK_WEBHOOK_URL }}
    repo_name: ${{ github.repository }}
    run_url: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
    message: "Lint failed for ${{ inputs.language }}"
```
