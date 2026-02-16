# GitHub Actions — Reusable Workflows POC

## Documentation

This documentation covers the **Proof of Concept** we built to demonstrate hands-on expertise in GitHub Actions — specifically, centralized CI/CD using reusable workflows and composite actions.

The documentation is structured in two parts so you can get the high-level picture first and dive deeper only when needed.

---

### Part 1 — Overview

Start here for the big picture: what we built, what it demonstrates, and why it matters.

1. **[Executive Summary](01-executive-summary.md)** — What the POC is, key outcomes, and capabilities at a glance
2. **[Architecture](02-architecture.md)** — Hub-and-spoke design and component relationships
3. **[Reusable Workflows](03-reusable-workflows.md)** — The five workflows and what each one does
4. **[Composite Actions](04-composite-actions.md)** — Toolchain setup and Slack notification actions
5. **[Consumer Repos](05-consumer-repos.md)** — How four sample repos adopt shared workflows

### Part 2 — Deep Dive

Detailed technical reference for those who want to explore further.

6. **[GHA Platform Capabilities](06-jenkins-to-gha-migration.md)** — Full feature coverage and comparison with traditional CI tools
7. **[Live Dashboard](07-dashboard.md)** — Cross-repo CI/CD visibility dashboard
8. **[Advanced GHA Patterns](08-advanced-patterns.md)** — Dynamic matrix, service containers, environment gates, OIDC, and more
