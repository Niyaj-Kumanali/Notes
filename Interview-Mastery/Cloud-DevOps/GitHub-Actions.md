# GitHub Actions Study Guide

## 1. Executive Summary

GitHub Actions is a CI/CD and automation platform integrated directly into GitHub repositories. It enables developers to automate software workflows including build, test, deploy, and any other GitHub repository events. With a marketplace of 10,000+ pre-built actions, support for Linux, macOS, and Windows runners, and deep GitHub integration, it has become one of the most popular CI/CD platforms. Workflows are defined as YAML files in the `.github/workflows` directory.

## 2. Core Theory

### 2.1 GitHub Actions Architecture

```
GitHub Event (push, PR, issue, etc.)
       |
       v
   Workflow (.github/workflows/*.yml)
       |
       v
   Job (runs on runner)
       |
    +--+--+
    |     |
  Step   Step
  (uses) (run)
    |     |
  Action Shell/Command
```

### 2.2 Core Components

- **Workflow**: An automated process defined in YAML, consisting of one or more jobs
- **Job**: A set of steps that execute on the same runner
- **Step**: An individual task (either a shell command or a reusable action)
- **Action**: A reusable unit of code (Docker container or JavaScript)
- **Runner**: A server that executes workflows (GitHub-hosted or self-hosted)
- **Event**: A trigger that starts a workflow (push, pull_request, schedule, etc.)

### 2.3 Execution Model

```yaml
# Default: sequential jobs with implicit dependencies
jobs:
  job1:  # runs first
  job2:  # runs second (depends on job1)

# With explicit dependencies
jobs:
  job1:
  job2:
    needs: job1  # runs after job1
  job3:
    needs: [job1, job2]  # runs after both
  job4:
    if: always()  # runs regardless of previous results
```

## 3. Under-the-Hood Deep Dive

### 3.1 Runner Infrastructure

GitHub Actions runners are ephemeral virtual machines:

| Runner Type | vCPUs | RAM | Storage | OS |
|-------------|-------|-----|---------|-----|
| Ubuntu | 2-4 | 7-16 GB | 14-84 GB | Ubuntu 22.04 |
| Windows | 2-4 | 7-16 GB | 14-84 GB | Windows Server 2022 |
| macOS | 3-12 | 14-24 GB | 14-84 GB | macOS 12/13/14 |

### 3.2 Self-Hosted Runner Setup

```bash
# Register a self-hosted runner
curl -O -L https://github.com/actions/runner/releases/download/v2.314.1/actions-runner-linux-x64-2.314.1.tar.gz
tar xzf actions-runner-linux-x64-2.314.1.tar.gz
./config.sh --url https://github.com/myorg/myrepo --token <TOKEN>
sudo ./svc.sh install
sudo ./svc.sh start
```

### 3.3 Workflow Contexts and Expressions

GitHub Actions provides rich context objects:

```yaml
# Available contexts
github:     # github.sha, github.ref, github.event_name, github.actor
env:        # environment variables
job:        # job.status, job.services
steps:      # steps.<id>.outputs, steps.<id>.conclusion
runner:     # runner.os, runner.arch, runner.name
secrets:    # secrets.MY_SECRET
vars:       # variables set in repository/organization

# Expressions
${{ github.ref == 'refs/heads/main' }}
${{ contains(github.event.pull_request.labels.*.name, 'bug') }}
${{ fromJSON(steps.build.outputs.metadata).version }}
${{ hashFiles('**/package-lock.json') }}
${{ join(github.event.pull_request.labels.*.name, ', ') }}
```

### 3.4 Artifact and Cache Storage

```text
Artifacts: Uploaded per workflow run, stored for 90 days by default
  - Used for passing files between jobs
  - File size limit: 10 GB per artifact (GitHub.com)

Cache: Shared across workflow runs
  - Key-based storage for dependencies
  - Max size: 10 GB per repository
  - TTL: 7 days since last access
  - Restore keys provide fallback matching
```

## 4. Production Code Examples

### 4.1 Production CI Pipeline

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

defaults:
  run:
    shell: bash
    working-directory: ./src

env:
  NODE_VERSION: '20.x'
  PYTHON_VERSION: '3.12'

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      backend: ${{ steps.filter.outputs.backend }}
      frontend: ${{ steps.filter.outputs.frontend }}
      docs: ${{ steps.filter.outputs.docs }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v2
        id: filter
        with:
          filters: |
            backend:
              - 'backend/**'
              - 'shared/**'
            frontend:
              - 'frontend/**'
              - 'shared/**'
            docs:
              - 'docs/**'
              - '*.md'

  lint:
    needs: changes
    if: ${{ needs.changes.outputs.backend == 'true' || needs.changes.outputs.frontend == 'true' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: '**/package-lock.json'
      - run: npm ci
      - run: npm run lint
      - run: npm run format-check

  test:
    needs: changes
    if: ${{ needs.changes.outputs.backend == 'true' }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18.x, 20.x, 21.x]
        python-version: ['3.11', '3.12']
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    env:
      DB_URL: postgres://postgres:postgres@localhost:5432/test
      REDIS_URL: redis://localhost:6379
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
          cache-dependency-path: '**/package-lock.json'
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: npm ci
      - run: npm run test:ci
      - uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
          flags: unittests
          name: codecov-umbrella

  build:
    needs: [lint, test]
    if: ${{ always() && needs.lint.result == 'success' && needs.test.result == 'success' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          retention-days: 7

  security-scan:
    needs: changes
    if: ${{ needs.changes.outputs.backend == 'true' || needs.changes.outputs.frontend == 'true' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
      - uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  dependency-review:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/dependency-review-action@v3
        with:
          fail-on-severity: high
```

### 4.2 Multi-Environment CD Pipeline

```yaml
name: Deploy to Kubernetes

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-,format=short
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=raw,value=latest,enable={{is_default_branch}}
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: docker
    if: |
      github.ref == 'refs/heads/main' && 
      (github.event_name == 'push' || github.event.inputs.environment == 'staging')
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-kubectl@v3
      - uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.KUBE_CONFIG_STAGING }}
      - name: Deploy to Staging
        run: |
          envsubst < k8s/deployment.yaml | kubectl apply -f -
          kubectl set image deployment/myapp \
            myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          kubectl rollout status deployment/myapp --timeout=300s
      - name: Run Smoke Tests
        run: |
          curl -f --retry 5 --retry-delay 10 https://staging.myapp.com/health

  e2e-tests:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:e2e -- --base-url=https://staging.myapp.com
        env:
          E2E_API_KEY: ${{ secrets.E2E_API_KEY }}

  deploy-production:
    needs: e2e-tests
    if: success()
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-kubectl@v3
      - uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.KUBE_CONFIG_PRODUCTION }}
      - name: Canary Deploy
        run: |
          # Deploy 10% canary
          kubectl apply -f k8s/canary.yaml
          sleep 120
          # Check canary health
          CANARY_OK=$(curl -s -o /dev/null -w "%{http_code}" https://myapp.com/health)
          if [ "$CANARY_OK" != "200" ]; then
            echo "Canary failed, rolling back"
            kubectl delete -f k8s/canary.yaml
            exit 1
          fi
          # Full rollout
          kubectl set image deployment/myapp \
            myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          kubectl rollout status deployment/myapp --timeout=300s
```

### 4.3 Reusable Workflow

```yaml
# .github/workflows/build-deploy.yml (reusable)
name: Build and Deploy

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      node-version:
        required: false
        type: string
        default: '20.x'
    secrets:
      KUBE_CONFIG:
        required: true
      DOCKER_USERNAME:
        required: true

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
      - run: npm ci && npm run build
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - run: docker build -t myapp .
      - run: echo "Deploying to ${{ inputs.environment }}"

# Calling workflow
name: Deploy Service
on:
  push:
    branches: [main]
jobs:
  deploy-staging:
    uses: ./.github/workflows/build-deploy.yml
    with:
      environment: staging
    secrets:
      KUBE_CONFIG: ${{ secrets.KUBE_CONFIG_STAGING }}
      DOCKER_USERNAME: ${{ github.actor }}
```

### 4.4 Scheduled Maintenance Workflow

```yaml
name: Scheduled Tasks

on:
  schedule:
    # Run daily at 2 AM UTC
    - cron: '0 2 * * *'
  workflow_dispatch:  # Allow manual trigger

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Clean old artifacts
        run: |
          gh extension install actions/gh-actions-cache
          gh actions-cache list --limit 100 \
            --json id,key --jq '.[].id' | \
            xargs -I {} gh actions-cache delete {} --confirm

  db-backup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Backup database
        env:
          DB_URL: ${{ secrets.DB_URL }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          pg_dump $DB_URL | gzip > backup-$(date +%Y%m%d).sql.gz
          aws s3 cp backup-*.sql.gz s3://myapp-db-backups/

  dependency-update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm audit --json > audit-report.json
      - uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const report = JSON.parse(fs.readFileSync('audit-report.json', 'utf8'));
            if (report.metadata.vulnerabilities.high > 0) {
              github.rest.issues.create({
                owner: context.repo.owner,
                repo: context.repo.repo,
                title: 'Security vulnerabilities found',
                body: 'High severity vulnerabilities found in dependencies'
              });
            }
```

## 5. Real-World Scenarios

### 5.1 Monorepo Filtered CI

```yaml
name: Monorepo CI

on:
  pull_request:

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.changes.outputs.services }}
    steps:
      - uses: actions/checkout@v4
      - id: changes
        uses: dorny/paths-filter@v2
        with:
          filters: |
            services:
              - 'services/**'
              - '!services/*/README.md'
            shared:
              - 'shared/**'
            docs:
              - 'docs/**'

  service-ci:
    needs: detect-changes
    if: ${{ needs.detect-changes.outputs.services != '[]' }}
    strategy:
      matrix:
        service: ${{ fromJSON(needs.detect-changes.outputs.services) }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          cd services/${{ matrix.service }}
          npm ci && npm test && npm run build
```

### 5.2 Matrix Build and Cross-Platform Testing

```yaml
name: Cross-Platform Build

on: [push, pull_request]

jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [18.x, 20.x]
        include:
          - os: ubuntu-latest
            node: 20.x
            coverage: true
        exclude:
          - os: macos-latest
            node: 18.x
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci
      - run: npm test
      - if: ${{ matrix.coverage }}
        run: npm run coverage
```

## 6. Performance

### 6.1 Optimization Strategies

```yaml
# Dependency caching
- uses: actions/cache@v3
  with:
    path: |
      ~/.npm
      ${{ github.workspace }}/.next/cache
    key: ${{ runner.os }}-modules-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-modules-

# Docker layer caching
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### 6.2 Reducing Costs

```yaml
# Skip workflow on docs only
on:
  push:
    paths-ignore:
      - 'docs/**'
      - '*.md'
      - '.gitignore'

# Use concurrency to cancel redundant runs
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

# Limit job matrix size
strategy:
  matrix:
    node-version: [20.x]  # Not every version
  max-parallel: 4
```

## 7. Security

### 7.1 Secrets Management

```yaml
# Good: Use encrypted secrets
env:
  API_KEY: ${{ secrets.API_KEY }}

# Better: Use OpenID Connect (OIDC) for cloud auth
jobs:
  deploy:
    permissions:
      id-token: write  # Need to request OIDC token
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActions
          aws-region: us-east-1
```

### 7.2 Workflow Hardening

```yaml
# Principle of least privilege
permissions:
  contents: read
  issues: none
  pull-requests: read
  packages: write

# Prevent untrusted PRs from accessing secrets
pull_request_target:
  types: [labeled]

# Use pinned action versions (SHA256)
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
```

### 7.3 SECURITY.md for Actions

```yaml
# Preventing supply chain attacks:
# 1. Pin actions to full commit SHA, not tags
# 2. Review marketplace actions before use
# 3. Use Dependabot to keep actions updated
# 4. Run dependency review on PRs
```

## 8. Common Mistakes

| Mistake | Issue | Fix |
|---------|-------|-----|
| Using `pull_request_target` without care | Security risk (runs in base context) | Validate carefully or use `pull_request` |
| Not pinning action versions | Supply chain risk | Pin to SHA or major version |
| Hardcoding credentials in workflow | Leaks secrets | Use GitHub Secrets |
| No concurrency control | Wasted runs on quick pushes | Use concurrency group |
| Running full test matrix always | Slow, expensive | Use path filters |
| Ignoring self-hosted runner security | Arbitrary code execution | Isolate runners, no public forks |
| Large artifact uploads | Slow pipelines, storage costs | Compress, limit retention |
| Missing error handling in shell scripts | Silent failures | Use `set -euo pipefail` |
| No conditional deployment | Accidental production deploys | Branch/event conditions |
| Every action on `push` instead of `pull_request` | Wasted CI runs | Use `pull_request` for PR checks |

## 9. Senior Engineer Perspective

### 9.1 Migration Strategy (Jenkins/CircleCI to GitHub Actions)

```text
Phase 1: Audit existing pipelines (3-5 days)
Phase 2: Port 1-2 simple workflows (1-2 days)
Phase 3: Port complex workflows with parallel testing (3-5 days)
Phase 4: Configure environments, secrets, and OIDC (1-2 days)
Phase 5: Train team, sunset old CI (1 week)
```

### 9.2 GitHub Actions Maturity Model

| Level | Capabilities |
|-------|-------------|
| 1 | Basic CI (build + test on push) |
| 2 | CD (deploy to staging, manual prod) |
| 3 | Multi-environment CD + environment approvals |
| 4 | Reusable workflows + composite actions + matrix builds |
| 5 | Full automation + OIDC + policy as code + auto-rollback |

### 9.3 Enterprise Governance

```yaml
# Organization-level workflow policies
# .github/workflows/required-checks.yml
name: Required Checks
on:
  pull_request_target:
    types: [opened, synchronize]

jobs:
  required-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v6
        with:
          script: |
            const { data: checks } = await github.rest.checks.listForRef({
              ...context.repo,
              ref: context.payload.pull_request.head.sha
            });
            // Enforce organizational policies
```

## 10. Interview Questions (Easy)

1. What is GitHub Actions and what is it used for?
2. What is the difference between a workflow, job, and step?
3. What is an action in GitHub Actions?
4. What file format is used to define workflows?
5. How do you trigger a workflow on a push event?
6. What is a runner in GitHub Actions?
7. How do you access secrets in a workflow?
8. What is the GitHub Actions marketplace?
9. What is the purpose of `actions/checkout`?
10. How do you run a workflow on a schedule?

## 10. Interview Questions (Medium)

11. How do you pass data between jobs in a workflow?
12. What is a matrix build and how do you configure it?
13. How does concurrency control work in GitHub Actions?
14. What is the difference between `pull_request` and `pull_request_target`?
15. How do you cache dependencies in GitHub Actions?
16. How do you create a reusable workflow?
17. What are environment protection rules?
18. How do you set up OIDC for cloud authentication?
19. What are composite actions and how are they different from workflows?
20. How do you implement deployment gates/approvals?

## 11. Advanced Interview Questions (Hard)

1. Design a secure CI/CD pipeline using GitHub Actions for a financial services company requiring audit trails and approval gates.

2. How would you implement a monorepo CI strategy with 100+ services using GitHub Actions that only builds changed services?

3. Design a multi-region deployment pipeline using GitHub Actions with canary analysis and automatic rollback.

4. How do you manage secrets across multiple environments and cloud providers in GitHub Actions?

5. Design a self-hosted runner auto-scaling solution for GitHub Actions.

6. How would you implement a blue-green deployment with traffic shifting using GitHub Actions?

7. Design a database migration pipeline that runs before application deployment with rollback capability.

8. How do you optimize GitHub Actions costs for a 200-developer organization?

9. Design a pipeline that enforces SLSA (Supply-chain Levels for Software Artifacts) compliance.

10. How would you implement feature flag deployments using GitHub Actions?

## 11. Advanced Interview Questions (System Design)

11. Design a GitHub Actions-based platform for a 1000-developer monorepo with 500+ microservices.

12. Design a multi-cloud deployment pipeline spanning AWS, Azure, and GCP using GitHub Actions.

13. Design a release management system using GitHub Actions with semantic versioning, changelog generation, and package publishing.

14. Design a governance system that enforces organizational policies across 100+ repositories' GitHub Actions workflows.

15. Design a disaster recovery automation workflow for a critical service.

## 12. Expert-Level Interview Questions (Architect)

1. Design an enterprise GitHub Actions architecture supporting 5,000+ developers across 500+ repositories with centralized governance, cost allocation, and security compliance.

2. Architect a platform migration from Jenkins to GitHub Actions for a large enterprise with 2,000+ pipelines, handling parallel operation during migration and zero downtime.

3. Design a GitHub Actions workflow composition system that allows teams to define custom pipelines while enforcing organizational security and compliance policies.

4. How would you design a build artifact management system integrated with GitHub Actions that handles 10TB+ of artifacts daily with geo-replication and retention policies?

5. Architect a GitHub Actions caching layer that spans across organizations, repositories, and runners to minimize build times for large enterprise monorepos.

6. Design a system that automatically generates and maintains GitHub Actions workflows from OpenAPI specifications, infrastructure definitions, and deployment topology.

7. How would you design a policy-as-code framework for GitHub Actions that enforces deployment policies, security gates, and compliance checks without blocking developer velocity?

8. Architect a multi-cloud, multi-region disaster recovery system using GitHub Actions that can failover critical services in under 5 minutes.

9. Design a CI/CD analytics platform using GitHub Actions webhooks and APIs that provides real-time visibility into deployment frequency, lead time, MTTR, and change failure rate across the organization.

10. How would you design a secure software supply chain using GitHub Actions that implements SLSA Level 4, including hermetic builds, provenance attestation, and verifiable artifacts?

## 13. Debugging & Troubleshooting

### 13.1 Common Issues and Solutions

```yaml
# Debugging: Enable debug logging
# Add secret: ACTIONS_STEP_DEBUG = true
# Add secret: ACTIONS_RUNNER_DEBUG = true

# Debug step outputs
- name: Debug
  run: |
    echo "GitHub context: ${{ toJSON(github) }}"
    echo "Runner context: ${{ toJSON(runner) }}"
    echo "Job context: ${{ toJSON(job) }}"
    echo "Steps context: ${{ toJSON(steps) }}"
```

### 13.2 Troubleshooting Workflow Failures

```bash
# Re-run failed jobs
gh run view <run-id>
gh run rerun <run-id> --failed

# Download and inspect logs
gh run view <run-id> --log > workflow.log

# List recent workflow runs
gh run list --limit 10 --workflow=ci.yml

# Check runner status
gh api repos/:owner/:repo/actions/runners

# Debug self-hosted runner
sudo journalctl -u actions.runner.* -f
cat /home/runner/_diag/*.log
```

### 13.3 Performance Debugging

```yaml
- name: Measure step timing
  run: |
    echo "Starting long running task at $(date)"
    SECONDS=0
    # ... long running command ...
    duration=$SECONDS
    echo "Task took $((duration / 60)) minutes and $((duration % 60)) seconds"
```

## 14. Comparison Section

### GitHub Actions vs Competitors

| Feature | GitHub Actions | Jenkins | GitLab CI | CircleCI | Azure DevOps |
|---------|---------------|---------|-----------|----------|--------------|
| Setup | Built-in | Self-hosted | Built-in | Cloud | Built-in/Azure |
| YAML Config | In-repo | Jenkinsfile/Groovy | .gitlab-ci.yml | .circleci/config.yml | azure-pipelines.yml |
| Marketplace | 10k+ actions | 1k+ plugins | Limited | Orbs | Extensions |
| Hosted Runners | Linux/Mac/Windows | N/A | Linux/Mac/Windows | Linux/Mac/Windows | Linux/Mac/Windows |
| Self-Hosted | Yes | Yes | Yes | Yes | Yes |
| Matrix Builds | Native | Plugin | Native | Native | Native |
| Reusable Config | Composite/Reusable | Shared Libraries | Includes | Orbs | Templates |
| Container Support | Native | Plugin | Native | Native | Native |
| Cost | Free tier | Free | Free tier | Free tier | Free tier |
| OIDC Support | Yes | Plugin | Yes | Yes | Yes |

### Actions vs Composite Actions vs Reusable Workflows

| Feature | Regular Action | Composite Action | Reusable Workflow |
|---------|---------------|------------------|-------------------|
| Language | JavaScript/Docker | YAML only | YAML only |
| Reusability | Across workflows | Across workflows | Across repos |
| Steps | Custom code | Shell steps | Full workflow |
| Secrets | Input parameters | Input parameters | Yes (secrets: inherit) |
| Use Case | Complex logic | Simplify multi-step | Standardize pipelines |

## 15. Revision Notes

### Key Concepts Quick Reference

```text
Workflow = One or more jobs triggered by events
Job = Series of steps on same runner
Step = Single command or action
Action = Reusable code unit
Runner = Execution environment
Event = Trigger condition

Workflow file location: .github/workflows/*.yml
Max workflow runtime: 6 hours (all jobs)
Max job runtime: 6 hours
Max artifact retention: 90 days
Max artifacts per run: 10 GB
Max workflow runs: 500 per hour per repo
```

### Commonly Used Actions

```yaml
actions/checkout@v4        # Checkout repository
actions/setup-node@v4      # Setup Node.js
actions/setup-python@v5    # Setup Python
actions/cache@v3           # Cache dependencies
actions/upload-artifact@v4 # Upload build artifacts
actions/download-artifact@v4 # Download build artifacts
actions/github-script@v6   # Run JavaScript with GitHub API
docker/login-action@v3     # Login to container registry
docker/build-push-action@v5 # Build and push Docker image
azure/k8s-set-context@v3   # Set Kubernetes context
```

### Environment Variables vs Secrets vs Variables

```yaml
env:           # Visible in workflow logs, available to all steps
  MY_VAR: value

secrets:       # Masked in logs, encrypted, set in repo settings
  API_KEY: ${{ secrets.API_KEY }}

vars:          # Plain text, not masked, set in repo/organization settings
  TEAM_NAME: ${{ vars.TEAM_NAME }}
```

## 16. Cheat Sheet

```text
+======================================================================+
|                    GITHUB ACTIONS CHEAT SHEET                         |
+======================================================================+

  WORKFLOW STRUCTURE
+----------------------------------------------------------------------+
| .github/workflows/*.yml                                              |
|                                                                      |
| name: My Workflow                                                    |
| on: [push, pull_request]                                             |
|                                                                      |
| jobs:                                                                |
|   build:                                                             |
|     runs-on: ubuntu-latest                                           |
|     steps:                                                           |
|       - uses: actions/checkout@v4                                    |
|       - run: npm ci && npm test                                      |
+----------------------------------------------------------------------+

  TRIGGER EVENTS
+----------------------------------------------------------------------+
| on: push                       | Any branch push                     |
| on: pull_request               | PR opened/synced                    |
| on: schedule:                  | Cron timer                          |
|   - cron: '0 2 * * *'         |                                     |
| on: workflow_dispatch          | Manual trigger                      |
| on: workflow_call              | Reusable workflow                   |
| on: release:                   | Release published                   |
|   types: [published]           |                                     |
+----------------------------------------------------------------------+

  EXPRESSIONS & CONTEXTS
+----------------------------------------------------------------------+
| ${{ github.ref }}             | Branch or tag ref                    |
| ${{ github.sha }}             | Commit SHA                           |
| ${{ github.event_name }}      | Trigger event type                   |
| ${{ github.actor }}           | User who triggered                   |
| ${{ secrets.MY_SECRET }}      | Access secrets                       |
| ${{ env.MY_VAR }}             | Access env variables                 |
| ${{ runner.os }}              | Runner OS (Linux/Windows/macOS)      |
| ${{ steps.meta.outputs.tags }}| Step outputs                         |
| ${{ hashFiles('**/lock') }}   | File hash for cache keys             |
| ${{ success() }}              | Previous steps succeeded             |
| ${{ always() }}               | Run regardless of previous           |
| ${{ failure() }}              | Previous step(s) failed              |
+----------------------------------------------------------------------+

  CONDITIONAL EXECUTION
+----------------------------------------------------------------------+
| if: github.ref == 'refs/heads/main'                                  |
| if: success()                                                        |
| if: always()                                                         |
| if: failure()                                                        |
| if: contains(github.event.pull_request.labels.*.name, 'safe')        |
| if: ${{ !cancelled() }}                                              |
+----------------------------------------------------------------------+

  MATRIX BUILDS
+----------------------------------------------------------------------+
| strategy:                                                            |
|   matrix:                                                            |
|     os: [ubuntu, windows]                                            |
|     node: [18, 20]                                                   |
|     exclude:                                                         |
|       - os: windows                                                  |
|         node: 18                                                     |
| runs-on: ${{ matrix.os }}-latest                                     |
+----------------------------------------------------------------------+

  CACHING
+----------------------------------------------------------------------+
| - uses: actions/cache@v3                                             |
|   with:                                                              |
|     path: ~/.npm                                                     |
|     key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}  |
|     restore-keys: ${{ runner.os }}-npm-                              |
+----------------------------------------------------------------------+

  REUSABLE WORKFLOWS
+----------------------------------------------------------------------+
| # Caller                                                            |
| jobs:                                                                |
|   call-workflow:                                                     |
|     uses: ./.github/workflows/deploy.yml                             |
|     with:                                                            |
|       env: production                                                |
|     secrets: inherit                                                 |
|                                                                      |
| # Callee (.github/workflows/deploy.yml)                              |
| on:                                                                  |
|   workflow_call:                                                     |
|     inputs:                                                          |
|       env:                                                           |
|         required: true                                               |
|         type: string                                                 |
+----------------------------------------------------------------------+

+======================================================================+
|  PRO TIPS:                                                           |
|  - Use actions/checkout@v4 as first step in every job                |
|  - Pin action versions for security (SHA256 or major version)        |
|  - Use concurrency groups to cancel redundant runs                   |
|  - Set environment protection rules for production deployments       |
|  - Use OIDC instead of static credentials for cloud auth             |
|  - Monitor Actions billing in organization settings                  |
|  - Test reusable workflows in isolation before sharing               |
+======================================================================+
```
