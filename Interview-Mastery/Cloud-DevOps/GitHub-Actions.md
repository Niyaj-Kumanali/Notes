# GitHub Actions

---

## Overview

- **Definition:** GitHub Actions is a CI/CD and automation platform built into GitHub repositories, enabling automated workflows for build, test, deploy, and any GitHub event.
- **Why It Exists:** It provides deep GitHub integration (PRs, issues, releases), a marketplace of 10,000+ pre-built actions, free minutes for public repos, and supports Linux/macOS/Windows runners with automatic scaling.
- **Key Concepts:** **Workflow** (YAML-defined automated process in `.github/workflows/*.yml`), **Job** (set of steps on one runner), **Step** (shell command or reusable action), **Action** (reusable unit — JavaScript or Docker), **Runner** (GitHub-hosted or self-hosted execution environment), **Event** (trigger — push, pull_request, schedule, workflow_dispatch)

---

## Core Concepts

- **Workflow Structure:** YAML with `name`, `on` (triggers), `env`, `jobs`. Jobs run in parallel by default; use `needs` for dependencies. Each job specifies `runs-on` and contains `steps`.
- **Expressions and Contexts:** Use `${{ }}` syntax. Contexts: `github` (sha, ref, event), `env`, `secrets`, `vars`, `runner`, `steps`, `job`. Functions: `contains()`, `startsWith()`, `hashFiles()`, `fromJSON()`, `toJSON()`.
- **Runner Types:** GitHub-hosted (Ubuntu 2-4 vCPU, Windows, macOS 3-12 vCPU). Self-hosted (custom hardware, network access, cached tools). Ephemeral VMs that auto-terminate after job.
- **Matrix Builds:** Test across multiple OS, language versions, or configs using `strategy.matrix`. Use `include`/`exclude` for fine-grained control. `fail-fast: false` to run all even if one fails.
- **Artifacts vs Cache:** Artifacts pass files between jobs in one run (90-day retention, 10 GB limit). Cache shares dependencies across runs (7-day TTL, key-based, 10 GB per repo).

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    strategy:
      matrix:
        node-version: [18.x, 20.x]
        os: [ubuntu-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci && npm test

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying..."
```

---

## Common Mistakes

- **Using `pull_request_target` without care** — Security risk (runs in base branch context, can expose secrets to PRs)
- **Not pinning action versions** — Supply chain risk; pin to SHA or major version tag
- **Hardcoding credentials in workflow** — Leaks secrets; use GitHub Secrets
- **No concurrency control** — Wasted runs on quick pushes; use `concurrency.group`
- **Running full test matrix always** — Slow and expensive; use path filters
- **Ignoring self-hosted runner security** — Code execution risk; isolate runners
- **Large artifact uploads** — Slow pipelines, storage costs; compress, limit retention
- **Missing error handling in shell scripts** — Silent failures; use `set -euo pipefail`
- **No conditional deployment** — Accidental production deploys; add branch/event conditions
- **Every action on `push` instead of `pull_request`** — Wasted CI runs; use `pull_request` for PR checks

---

## Key Design Considerations

- **Pipeline Optimization** — Concurrency groups to cancel redundant runs, path filters for monorepo, dependency caching (`actions/cache`), Docker layer caching (`type=gha`), matrix limited to necessary combinations
- **Security** — Use OIDC for cloud auth instead of static credentials, least privilege permissions per workflow, pin action versions to SHA, prefer `pull_request` over `pull_request_target`, use `actions/dependency-review-action` on PRs
- **Reusable Workflows** — Define standard pipelines in central `.github/workflows` called via `workflow_call`. Parameterize with `inputs` and `secrets`. Enables consistent CI/CD across org without duplication
- **Cost Management** — Path filters to skip CI for docs/README, limit matrix size, concurrency to cancel stale runs, short artifact retention, self-hosted runners for heavy workloads
- **Enterprise Governance** — Org-level workflow templates, required status checks, OIDC trust policies, environment protection rules (required reviewers, wait timer), branch protection rules
- **Maturity Model** — Level 1: Basic CI (build+test). Level 2: CD (deploy to staging). Level 3: Multi-env CD with approvals. Level 4: Reusable workflows + composite actions + matrix. Level 5: Full automation + OIDC + policy as code + auto-rollback

---

## Real-World Scenarios

**Scenario 1: Multi-Environment CD with OIDC and Environment Protection**
A FinTech company deploys microservices to AWS ECS across dev, staging, and production. Use OIDC for AWS authentication — no static AWS keys stored as secrets. Each environment has protection rules: dev (auto-deploy from main), staging (requires successful CI), production (requires approval from two approvers + Azure Monitor gate checking error rates < 0.5%).

```yaml
name: Deploy to ECS
on:
  push:
    branches: [main]
permissions:
  id-token: write
  contents: read
jobs:
  deploy-dev:
    environment: dev
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-dev
          aws-region: us-east-1
      - run: aws ecs update-service --cluster myapp --service dev --force-new-deployment
  deploy-prod:
    needs: deploy-dev
    environment: prod
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-prod
          aws-region: us-east-1
```

**Scenario 2: Monorepo CI with Dynamic Matrix for Changed Services**
A team manages 30 microservices in a single monorepo. Running all service CI on every commit wastes hours. Use path filtering and dynamic matrix generation.

```yaml
jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.filter.outputs.changes }}
    steps:
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            service-a: 'services/service-a/**'
            service-b: 'services/service-b/**'
  build:
    needs: changes
    strategy:
      matrix:
        service: ${{ fromJSON(needs.changes.outputs.services) }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cd services/${{ matrix.service }} && npm ci && npm test
```

**Scenario 3: Self-Hosted Runner Auto-Scaling with ARC**
A 200-developer organization spends $15K/month on GitHub-hosted runners. Migrate to self-hosted runners using actions-runner-controller on Kubernetes. Ephemeral runners per job (clean state, no cross-job contamination). Auto-scale based on queue depth. Scale to zero when idle. Custom VM image with pre-cached tools reduces job setup time from 60s to 5s.

---

## Scenario-Based Questions

1. **Q: Design a secure CI/CD pipeline for a financial services company using GitHub Actions.**
   A: Use OIDC for cloud auth (no static secrets), environment protection rules for prod (required reviewers), signed commits, dependency review action, CodeQL scanning, SBOM generation, artifact signing with Cosign, immutable deployment records.

2. **Q: How would you implement monorepo CI with 100+ services that only builds changed services?**
   A: Use `dorny/paths-filter` to detect changed paths. Matrix build with `fromJSON(needs.changes.outputs.services)`. Each service runs its own build/test commands. Shared library extracted and cached. Dependencies built only once.

3. **Q: Design a multi-region deployment pipeline with canary analysis and auto-rollback.**
   A: Deploy to Region 1 canary (10% traffic), monitor latency/error rates for 5 min, if healthy deploy 100%, then proceed to Region 2. CloudWatch/Datadog metrics trigger rollback if thresholds exceeded.

4. **Q: How do you manage secrets across multiple environments and cloud providers?**
   A: Use GitHub Environments with different secrets per environment. OIDC for cloud provider auth (AWS IAM roles, Azure Managed Identities). Organization secrets for shared values. Never reuse secrets across environments.

5. **Q: Design a self-hosted runner auto-scaling solution.**
   A: Use `actions-runner-controller` (ARC) on Kubernetes. Auto-scales based on job queue size. Ephemeral runners — each job gets a fresh runner pod. Cache Docker layers and dependencies in persistent volume. Scale to zero when idle.

6. **Q: How would you implement blue-green deployment with traffic shifting?**
   A: Deploy new version as separate "green" deployment. Run smoke tests. Update load balancer/service selector from "blue" to "green". Keep blue running for rollback. Use GitHub Actions deployment environment for tracking.

7. **Q: Design a database migration pipeline with rollback capability.**
   A: Migration pipeline triggered by push to migrations folder. Run migrations against staging DB. Validate schema integrity. Deploy app. Run migrations against prod DB (backward-compatible only). Rollback script in same PR. Flyway/Liquibase for version control.

8. **Q: How do you optimize GitHub Actions costs for a 200-developer organization?**
   A: Use self-hosted runners for primary CI (Linux spot VMs), path filters to skip unnecessary runs, concurrency to cancel stale runs, limit matrix to LTS versions only, short artifact retention (7 days), cache dependencies aggressively.

9. **Q: Design a pipeline that enforces SLSA Level 3 compliance.**
   A: Hermetic builds (no network access during build), provenance attestation (generate SLSA provenance), signed artifacts (Cosign), verifiable build steps, immutable build logs, dependency review, reproducible builds.

10. **Q: How would you implement feature flag deployments using GitHub Actions?**
    A: Deploy code with feature flag system (LaunchDarkly, Flagsmith). Pipeline runs A/B tests. Metrics evaluated post-deployment. Rollback via flag toggle without redeployment. Canary users via targeting rules. Gradual ramp-up automated.


---

## Interview Questions

1. **What is the difference between a workflow, job, and step in GitHub Actions?**
   A: A workflow is an automated process defined in YAML (.github/workflows/*.yml). A job is a set of steps running on a single runner (can be parallel or sequential via needs). A step is an individual task (shell command or reusable action). Hierarchy: Workflow -> Jobs -> Steps -> Actions.

2. **What is the difference between `push`, `pull_request`, and `workflow_dispatch` triggers?**
   A: `push` triggers on code push to a branch. `pull_request` triggers on PR events (opened, synchronized). `workflow_dispatch` allows manual triggering from the GitHub UI. Use push for CI on main, pull_request for PR validation, and workflow_dispatch for ad-hoc runs.

3. **What is a matrix strategy and when would you use it?**
   A: A matrix runs a job across multiple combinations of variables (OS, language version, environment). Example: test Node.js 18 and 20 on Ubuntu and Windows. Use for cross-platform testing, multi-version compatibility, and environment-specific deployments. Limit matrix size to avoid excessive runs.

4. **What is the difference between GitHub-hosted and self-hosted runners?**
   A: GitHub-hosted: zero maintenance, pre-installed tools, automatic scaling, limited to 360 min/job (Linux), no VNet access, costs per minute. Self-hosted: full control, network access, unlimited execution time, cost-effective at scale, requires maintenance. Choose GitHub-hosted for simplicity, self-hosted for custom needs or cost savings.

5. **How does OIDC work in GitHub Actions?**
   A: GitHub issues an OIDC token per job. The cloud provider (AWS/Azure/GCP) trusts GitHub's OIDC endpoint. The workflow exchanges the token for short-lived cloud credentials using a specific action (configure-aws-credentials, azure/login, google-github-actions/auth). No static secrets stored in GitHub. The token includes claims (repo, environment, branch) that cloud providers can validate.

6. **What is the `actions/cache` action and how does it improve build times?**
   A: `actions/cache` caches dependencies and build outputs to speed up subsequent runs. Uses a key (typically based on lock file hash) to restore and save cache. Common caches: npm node_modules, Maven .m2, Go module cache, Docker layers. Cache is immutable once saved (new key = new cache). Reduces build times by 50-80%.

7. **What is a reusable workflow vs a composite action?**
   A: Reusable workflow (`.github/workflows/*.yml` with `on: workflow_call`) defines complete multi-job workflows called from other workflows. Composite action (`action.yml`) defines a single job's steps as a reusable unit. Use reusable workflows for sharing complete CI/CD patterns, composite actions for sharing step-level logic.

8. **How do environment protection rules work?**
   A: Environments (dev, staging, prod) have protection rules: required reviewers (one or more approvers), wait timer (delay before deploy), and deployment branches (restrict which branches can deploy). Combined with secret scoping (each environment has its own secrets), this prevents unauthorized deployments to sensitive environments.

9. **What is the difference between `needs` and `if` conditions in workflows?**
   A: `needs` creates a dependency — job B waits for job A to complete. `if` conditionally runs a job based on an expression (branch name, previous job status, inputs). Use needs for sequential execution, if for conditional execution. Combine: `needs: [build, test]` and `if: success()`.

10. **How do you handle concurrency in GitHub Actions?**
   A: Use `concurrency` group to limit concurrent runs. Common patterns: cancel-in-progress on same branch (`group: ${{ github.workflow }}-${{ github.ref }}, cancel-in-progress: true`). Or queue deployments per environment (`group: deploy-production`). Prevents wasted runs on rapid pushes and serializes deployments to sensitive environments.

---

## Developer Recommendations

- **Use OIDC over static secrets for all cloud authentication** — Static AWS keys in GitHub Secrets expire, leak, and require rotation. OIDC provides short-lived credentials per job, no secret management, and auditable access. Configure once per cloud provider, every workflow benefits. The IAM trust policy is your security boundary — lock it to specific repos, branches, and environments.

- **Pin all actions to commit SHAs, not version tags** — A tag like `v4` can be reassigned to malicious code if the action repository is compromised. Pinning to a SHA (`@e21f271`) ensures the exact code is used. Use Dependabot to automatically update pinned SHAs. The risk is low for official actions but critical for third-party actions.

- **Use path filters to avoid unnecessary CI runs** — Running the full CI suite on a README change wastes time, money, and runner capacity. Use `paths-ignore` for docs, config, and non-code changes. Use `paths` to run only specific service CI in monorepos. Combined with path-based matrix generation, you can reduce CI costs by 40-60%.

- **Cache everything, but cache wisely** — Without caching, every workflow reinstalls all dependencies. Use `actions/cache` with restore keys based on lock files. Cache npm, Maven, Go modules, Docker layers, and compiled binaries. Restore key: hit restores full cache, partial key restores and stores new cache. Set a TTL for stale caches.

- **Use environments for production deployments, not manual conditions** — `if: github.ref == 'refs/heads/main'` is a weak guard. Environments provide: scoped secrets, required reviewers, deploy branch restrictions, and audit trail. Every production deployment should go through an environment with at least one required approver. The deployment record is visible in the GitHub UI.

- **Prefer self-hosted runners at scale, but with caution** — At 50+ active developers, GitHub-hosted runner costs ($0.008/min for Linux 2-core) exceed self-hosted costs. Use ARC on Kubernetes for auto-scaling ephemeral runners. Security: isolate self-hosted runners from your cluster (dedicated node pool, no persistent storage). Never run untrusted PR workflows on self-hosted runners.