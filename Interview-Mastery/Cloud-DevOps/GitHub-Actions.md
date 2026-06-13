# GitHub Actions

---

## Overview

  - **Definition:** GitHub Actions is a CI/CD and automation platform built directly into GitHub repositories, enabling automated workflows triggered by any GitHub event. Workflows are defined as YAML files stored in `.github/workflows/` within the repository, making them co-located with source code and version-controlled. Actions supports building, testing, and deploying applications, but it also extends to any automation task: issue triage, release management, code quality checks, dependency updates, and security scanning. With over 10,000 pre-built actions in the GitHub Marketplace and support for custom JavaScript, Docker, or composite actions, almost any CI/CD pattern can be implemented without starting from scratch.

  - **Why It Exists:** Before GitHub Actions, teams needed separate CI/CD tools (Jenkins, CircleCI, Travis CI) that were disconnected from their code repository. This created friction — developers had to switch contexts between GitHub and the CI tool, pipeline configuration lived outside the repository, and pull request status checks required complex integrations. GitHub Actions solves this by embedding CI/CD directly into the GitHub workflow. When a developer opens a pull request, the CI pipeline runs automatically, and the results appear directly in the PR — green checkmarks or red X's on every commit. The deep GitHub integration extends to branch protection rules (required status checks block merges if CI fails), environment approval gates (manual approval required for production deployments), and deployment tracking (deployments visible in the GitHub UI). Actions also provides free minutes for public repositories and generous free tiers for private repositories, making it accessible to projects of all sizes. The platform supports Linux, macOS, and Windows runners, and scales automatically from zero to hundreds of concurrent jobs without any infrastructure management.

  - **Key Concepts:** A **Workflow** is the top-level automation unit — a YAML file in `.github/workflows/*.yml` that defines the automation process triggered by one or more events. Each workflow has a name, trigger events, environment variables, and a set of jobs. A **Job** is a collection of steps that all run on the same runner. Jobs within a workflow run in parallel by default, but dependencies can be declared with the `needs` keyword to create sequential execution. A **Step** is an individual task within a job — either a shell command (`run:`) or a reusable action (`uses:`). An **Action** is a reusable unit of automation, written as JavaScript (runs directly on the runner), Docker (runs in a container), or composite (combines multiple steps). Actions can be shared via the GitHub Marketplace or stored in private repositories. A **Runner** is the execution environment — either GitHub-hosted (Ubuntu, Windows, macOS, with all common tools pre-installed) or self-hosted (your own infrastructure, giving you full control over hardware, software, and network access). An **Event** is the trigger that starts the workflow — `push`, `pull_request`, `schedule` (cron), `workflow_dispatch` (manual trigger), `release`, `issues`, or any other GitHub webhook event.

---

## Core Concepts

  - **Workflow Structure:** A GitHub Actions workflow is a YAML file organized into a clear hierarchy. The `name` field identifies the workflow in the GitHub UI. The `on` field specifies the events that trigger the workflow — a single event like `push`, multiple events as an array, or event-specific configurations like branches and paths. The `env` field defines environment variables available to all jobs in the workflow. The `jobs` section is where the work happens: each job specifies `runs-on` (the runner type), optionally `needs` (dependencies on other jobs), and contains `steps` (the actual commands or actions to execute). Jobs run in parallel by default; sequential execution requires explicit `needs` declarations. This structure maps to a directed acyclic graph (DAG) where GitHub determines the optimal execution order based on declared dependencies.

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
env:
  NODE_ENV: test
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run lint
  test:
    needs: lint
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18.x, 20.x]
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
    environment: production
    steps:
      - run: echo "Deploying..."
```

  - **Expressions and Contexts:** GitHub Actions uses `${{ }}` syntax for expressions that are evaluated at runtime. Expressions can access contexts — objects that contain information about the workflow run, the repository, the event that triggered it, and more. The `github` context provides metadata about the trigger event: `github.sha` (commit SHA), `github.ref` (branch or tag reference), `github.event_name` (push, pull_request, etc.), and `github.actor` (the user who triggered the run). The `env` context accesses environment variables defined at any scope. The `secrets` context accesses encrypted secrets. The `vars` context accesses organization or repository variables (non-secret configuration values). The `runner` context provides information about the runner (OS, architecture, name). The `steps` context contains outputs from previous steps. The `job` context contains job status information (status, conclusion, services). Functions like `contains()`, `startsWith()`, `endsWith()`, `format()`, `join()`, `hashFiles()`, `fromJSON()`, and `toJSON()` enable complex conditional logic. Expressions are evaluated in a sandboxed environment and are not shell commands — they are functions that return values.

  - **Runner Types:** GitHub provides several runner options with different trade-offs. GitHub-hosted runners are the simplest: they run in ephemeral VMs that are automatically provisioned, configured with common tools (Node.js, Python, Java, Docker, .NET, Go, and many more), and terminated after each job. Linux runners (Ubuntu) start in approximately 20 seconds, Windows runners in about 60 seconds, and macOS runners can take several minutes. GitHub-hosted runners are free within usage limits (2000 minutes/month for free accounts, 3000 for pro, 50,000 for GitHub Enterprise) and charged per minute beyond that. They provide no access to your network — workflows cannot reach internal services or databases without exposing them through a VPN or self-hosted runner. Self-hosted runners are installed on your own infrastructure — physical servers, cloud VMs, or Kubernetes pods. They provide full network access (reach internal databases and services), unlimited execution time, custom hardware configurations (more CPU, memory, GPU), and pre-cached tools for faster job start times. The trade-offs are operational overhead (installation, updates, security patching) and security responsibility (self-hosted runners can execute arbitrary code, so they must be isolated from sensitive infrastructure). GitHub's recommended architecture uses ephemeral runners that are created per-job and destroyed after execution, preventing cross-job contamination.

  - **Matrix Builds:** Matrix strategies run a job across multiple combinations of variables simultaneously. The `strategy.matrix` keyword defines the dimensions — typically operating systems, language versions, or configuration parameters. GitHub creates a job instance for every combination in the matrix Cartesian product. For example, a matrix with `os: [ubuntu-latest, windows-latest]` and `node-version: [18.x, 20.x]` produces four job instances (2 OS x 2 versions). The `include` keyword adds additional combinations that are not part of the Cartesian product, while `exclude` removes specific combinations (useful for skipping known incompatibilities). The `fail-fast` option (default: true) cancels all in-progress matrix jobs when any job fails; setting it to false allows independent jobs to continue even if others fail, which is useful for collecting results from all test environments even if some fail. Matrices are controlled at the workflow level, not the repository level, so each workflow must define its own matrix. For large matrices with 20+ combinations, consider splitting into separate workflows or using dynamic matrix generation from a script.

  - **Artifacts vs Cache:** Artifacts and cache serve different purposes in GitHub Actions. **Artifacts** pass files between jobs in the same workflow run — for example, the build job produces a compiled binary and the deploy job needs that binary. Use `actions/upload-artifact` to upload and `actions/download-artifact` to download. Artifacts are retained for 90 days by default (configurable per repository), with a maximum of 10 GB total storage per repository. Artifacts are immutable once uploaded. **Cache** shares dependencies and build outputs across workflow runs on the same branch — for example, `node_modules` can be cached so that subsequent runs do not need to reinstall all dependencies. Use `actions/cache` with a key based on the lock file hash: a cache hit restores the cached files, and a cache miss triggers the full install and saves a new cache. Cache has a 7-day time-to-live, is key-based (not path-based), and is limited to 10 GB per repository. The key insight: artifacts transfer files forward (within a run), while cache transfers files backward (reusing results from previous runs). Use artifacts for build outputs (binaries, test reports), and cache for dependency folders (node_modules, .m2, ~/.cache).

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

- **Using `pull_request_target` without care** — The `pull_request_target` event trigger runs the workflow in the context of the base branch (not the merge commit), which means it has access to the base branch's secrets and environment variables. This is intentional — it allows workflows that need secrets (like commenting on a PR or labeling issues) to run safely. However, if an attacker opens a PR that modifies the workflow file itself, the modified workflow runs with access to the base branch's secrets. This is a common supply chain attack vector. The safe alternative is to use `pull_request` (which runs in the context of the merge commit with no secret access) for CI checks, and only use `pull_request_target` for actions that explicitly need base branch secrets (like posting a comment). When `pull_request_target` is necessary, never check out and execute code from the PR head branch — only run trusted actions from the base branch.
  - **Why it looks correct:** `pull_request_target` runs in the base branch context and has access to secrets, which is exactly what you need for PR workflows — the security implications of allowing PRs to modify the workflow that runs are non-obvious.

- **Not pinning action versions** — Using `@v4` or `@main` for action versions is convenient but risky. A tag like `v4` is a mutable reference — the action maintainer could push a new commit to the `v4` branch at any time, silently changing what your workflow executes. This is a supply chain security vulnerability: an action that was safe yesterday could be compromised today without any change in your workflow file. Always pin actions to the full commit SHA (`@e21f2714e9bf9e806e9f5f6f6f7f8f9f`). For official GitHub actions (actions/checkout, actions/setup-node), the risk is lower because GitHub controls them, but the same principle applies for third-party actions. Use Dependabot's "GitHub Actions" ecosystem to automatically create pull requests when pinned SHAs need updating — this gives you visibility into what changed before you update.
  - **Why it looks correct:** `@v4` looks like a well-defined version — in most package managers a major version is immutable, but Git tags are mutable by design.

- **Hardcoding credentials in workflow** — It is surprisingly common to find inline AWS access keys, database connection strings, or API tokens hardcoded in workflow YAML files. Because workflows are stored in version-controlled repositories, these credentials are exposed to everyone with repository access and remain in git history forever even after removal. All secrets must be stored in GitHub Secrets (Settings > Secrets and variables > Actions) and accessed via `${{ secrets.SECRET_NAME }}`. For cloud authentication, use OIDC instead of static credentials — this eliminates the need to manage and rotate secret keys altogether. For non-secret configuration values (like environment names or URLs), use GitHub Variables (`${{ vars.VAR_NAME }}`) instead of hardcoding.
  - **Why it looks correct:** putting configuration directly in the YAML file is the most natural, readable way to write a workflow, and there's no visible warning that the value will be stored in git history forever.

- **No concurrency control** — Without concurrency configuration, a new push to a branch starts a new workflow run even if the previous run for that branch is still in progress. For active branches with frequent pushes, this creates a backlog of redundant runs — each push invalidates the previous one because the code has changed. The `concurrency` keyword solves this by grouping runs and optionally canceling in-progress runs within the same group. The standard pattern is `group: ${{ github.workflow }}-${{ github.ref }}` (unique per branch) with `cancel-in-progress: true`, which cancels the running workflow when a new push arrives. For deployment workflows, you may want serial execution without cancellation: use `group: deploy-production` to queue deployment runs rather than canceling them.
  - **Why it looks correct:** each push should be tested, and parallel runs seem like they'd give faster feedback — the waste only becomes apparent when you see the queue of finished-but-stale runs.

- **Running full test matrix always** — Running the complete test matrix (every OS × every language version × every configuration) on every commit is expensive and slow. Most changes only need a subset of the matrix. Use path filters to limit matrix execution — for example, if the change is in a service that only needs Node.js 20 on Linux, do not also run it on Windows with Node.js 18. Use matrix `include` for targeted additional combinations only when relevant (e.g., only run Windows tests when Windows-specific code changes). For pull requests, consider running only a subset of the matrix (the latest LTS version on Linux) and running the full matrix only on merges to main.
  - **Why it looks correct:** thorough cross-platform testing feels like the responsible thing to do — the cost in time and compute isn't visible until the CI pipeline stretches past 30 minutes.

- **Ignoring self-hosted runner security** — Self-hosted runners can execute arbitrary code from any workflow they run. If a self-hosted runner is configured to accept workflows from the entire organization, a contributor from any repository could execute code on that runner. The most dangerous scenario is running untrusted pull request workflows on self-hosted runners — a malicious PR could compromise the runner, access the network, and pivot to internal systems. Mitigations: never run untrusted forks on self-hosted runners; use dedicated runners for each repository or team; use ephemeral runners (each job gets a fresh, disposable environment); isolate runners in a dedicated network segment with no access to production; restrict which workflows can execute on self-hosted runners using runner group permissions.
  - **Why it looks correct:** self-hosted runners just look like more cost-effective CI machines — the security implications of giving external contributors code execution on your network aren't obvious.

- **Large artifact uploads** — Uploading large files (hundreds of megabytes or gigabytes) as build artifacts slows the pipeline, consumes storage, and can exceed GitHub's limits (10 GB per repository for artifacts). Compress artifacts before uploading, especially for logs, test reports, and build outputs. Set artifact retention to the minimum practical duration — 7 days for most cases, 30 days for audit purposes. For very large files, consider uploading to cloud storage (S3, Azure Blob) instead of GitHub Actions artifacts. Exclude unnecessary files from artifact uploads (node_modules, .git, temporary build files).
  - **Why it looks correct:** artifacts are the obvious way to pass build outputs between jobs, and uploading everything "just in case" seems safer than selectively choosing what to keep.

- **Missing error handling in shell scripts** — Shell commands in `run:` steps fail silently by default. If a command fails (non-zero exit code), the shell continues executing the next command unless error handling is enabled. Always start shell scripts with `set -euo pipefail`: `-e` exits on any error, `-u` treats unset variables as errors, `-o pipefail` propagates failures through piped commands. Without these options, a failed npm install in the middle of a script would not stop the workflow, and subsequent commands would run against a broken environment, producing confusing failures.
  - **Why it looks correct:** shell scripts typically work fine without `set -e` during local testing — individual command failures are visible in the terminal, masking the fact that in CI the script keeps running after a failure.

- **No conditional deployment** — Deploying to production on every push is a disaster waiting to happen. Without conditional deployment, a push to a feature branch triggers the deployment workflow and deploys incomplete code to production. Always gate deployment jobs with explicit conditions: `if: github.ref == 'refs/heads/main'` for production, `if: startsWith(github.ref, 'refs/heads/release/')` for release branches. For stronger protection, use GitHub Environments with deployment branch restrictions — the environment rule enforces that only specified branches can deploy, regardless of what the `if` condition says.
  - **Why it looks correct:** the workflow is defined in the same repository as the code, so it feels natural that any branch should be deployable — the problem only surfaces when someone pushes a half-finished feature branch that triggers an unintended production deployment.

- **Every action on `push` instead of `pull_request`** — Running the full workflow on every push to every branch is wasteful for documentation-only changes, README updates, or work-in-progress pushes. Use `pull_request` as the primary trigger for CI checks — the workflow runs when a PR is opened or updated, not on every push to a branch with no PR. Use `push` only for the main branch (where merges happen) or for specific branches that need direct CI. The `pull_request` event also provides better integration with GitHub's branch protection rules — required status checks are evaluated against the merge commit, blocking merges if checks fail.
  - **Why it looks correct:** triggering CI on `push` means every commit gets tested immediately, which seems like the most thorough approach — the waste isn't obvious until you see the same branch tested 10 times for 10 commits that are all part of one PR.

---

## Key Design Considerations

- **Pipeline Optimization** — Reducing CI/CD pipeline time is a direct multiplier on developer productivity. The most impactful optimization is concurrency control: use `concurrency.group` with `cancel-in-progress: true` to automatically cancel redundant runs on the same branch, ensuring only the latest commit is tested. Path filters (`paths` and `paths-ignore`) skip unnecessary runs entirely — a change to `README.md` should not trigger a full CI pipeline. For monorepos, use path-based job filtering to run only the relevant service's tests. Dependency caching with `actions/cache` is the most impactful single optimization — caching `node_modules`, Maven `.m2`, Go module cache, or Python's `~/.cache/pip` can reduce install times from minutes to seconds. Use key strategies that combine the lock file hash with OS: `key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}`. For Docker-based builds, use Docker layer caching with the `type=gha` cache backend — `docker build --cache-from=type=gha --cache-to=type=gha`. Limit matrix builds to necessary combinations: test on the latest LTS and one previous LTS version, not every minor version published in the last five years. Use `if` conditions on expensive jobs to run them only when relevant: run integration tests only when backend code changes, run E2E tests only on merges to main.

- **Security** — GitHub Actions security requires attention at multiple levels. OIDC for cloud authentication is the most important security practice: it eliminates static cloud credentials entirely by using short-lived tokens that GitHub exchanges with the cloud provider. Each workflow job gets its own scoped token, and the cloud provider validates claims (repository, branch, environment) against the IAM trust policy. Permission scoping at the workflow level (`permissions:`) restricts what the `GITHUB_TOKEN` can do — by default, new workflows get read-only permissions for contents and packages, but existing workflows may have broad full-access `GITHUB_TOKEN` permissions. Pin every action to its commit SHA (not version tag) to prevent supply chain attacks. Prefer `pull_request` over `pull_request_target` for CI checks — `pull_request_target` runs in the base branch context with secret access and should only be used when the workflow explicitly needs that access. Add `actions/dependency-review-action` to pull requests to scan dependency changes for known vulnerabilities before they are merged. Use CodeQL or other SAST tools to scan the application code itself. For self-hosted runners, enforce runner group isolation and never run untrusted PR workflows on them.

```yaml
permissions:  # Least privilege for GITHUB_TOKEN
  contents: read
  pull-requests: read
  id-token: write  # Only needed for OIDC
```

- **Reusable Workflows** — As an organization grows, duplicating the same pipeline logic across dozens or hundreds of repositories becomes unmanageable. Reusable workflows solve this by defining a complete workflow in a central location that other workflows call. The reusable workflow is a standard `.github/workflows/*.yml` file with `on: workflow_call` — it accepts `inputs` (typed parameters) and `secrets` (sensitive values passed securely). Callers use the `uses:` syntax with a reference to the workflow file: `uses: org/.github/.github/workflows/ci.yml@v1`. Unlike composite actions, reusable workflows can define multiple jobs, use matrix strategies, and have their own triggers. They are the recommended pattern for standardizing CI/CD across an organization: define a "CI" reusable workflow with linting, testing, and building; a "CD" reusable workflow with deployment stages; and a "Security" reusable workflow with scanning and SBOM generation. Each project calls these workflows with project-specific inputs, maintaining consistency without duplication.

```yaml
# .github/workflows/ci.yml — reusable workflow
name: CI Pipeline
on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string
    secrets:
      sonar-token:
        required: false
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
      - run: npm ci && npm test
```

- **Cost Management** — GitHub Actions costs scale with usage, and uncontrolled costs can surprise organizations. GitHub-hosted runner costs vary by OS: Linux is cheapest ($0.008/min for 2-core), Windows is more expensive ($0.016/min), and macOS is the most expensive ($0.08/min). The primary cost reduction strategies are: path filters to skip unnecessary runs (a documentation-only change costs $0), concurrency control to cancel stale runs (preventing parallel runs that both complete), matrix size reduction (testing on two Linux variants instead of four), short artifact retention (7 days instead of 90), and self-hosted runners for heavy workloads (CI or build workloads that run 100+ minutes per day reach break-even with self-hosted runners within months). For organizations exceeding GitHub's included minutes, set budget alerts in billing settings and regularly audit workflow usage to identify the most expensive runs.

- **Enterprise Governance** — Large organizations need centralized control over Actions usage. GitHub provides several governance features: org-level workflow templates that appear in the "New workflow" dialog when creating a repository, enforcing standard patterns from the start. Required status checks in branch protection rules enforce that CI must pass before merges. OIDC trust policies at the organization level define which repositories can assume which cloud roles, preventing unauthorized deployments. Environment protection rules require approvers, wait timers, and branch restrictions for sensitive environments. Runner group permissions control which repositories can use which self-hosted runners. Workflow approval rules require that first-time contributors' workflows are approved by a maintainer before running (preventing malicious PRs from executing code on GitHub-hosted runners).

- **Maturity Model** — GitHub Actions adoption typically follows a progression of maturity levels. Level 1 (Basic CI) is the starting point: a single workflow that runs `build` and `test` on every push. Level 2 (CD) adds deployment to at least one environment, typically staging. Level 3 (Multi-Environment CD) adds staging and production with manual approval gates, environment-specific secrets, and deployment tracking. Level 4 (Reusable Patterns) extracts common pipeline logic into reusable workflows and composite actions, adds matrix builds for cross-platform testing, and implements dependency caching. Level 5 (Full Automation) achieves OIDC-based cloud authentication for all workflows, policy-as-code for deployment approvals, automated rollback on health check failure, SBOM generation for every build, and integration with security scanning and dependency review as mandatory gates.

---

## Real-World Scenarios

- **Scenario 1: Multi-Environment CD with OIDC and Environment Protection**

  - A FinTech company deploys microservices to AWS ECS across three environments: dev, staging, and production. The company has strict compliance requirements — all production deployments must be approved by at least two senior engineers, and error rates must be below 0.5% before traffic is fully switched. The pipeline uses OIDC for AWS authentication, eliminating the need to store or rotate any static AWS keys. Each environment has an AWS IAM role with a trust policy that validates specific claims: the dev role trusts any branch in the repository, the staging role trusts only the main branch, and the production role trusts only the main branch AND requires the deployment to come from the production GitHub Environment (which has its own protection rules).

  - The dev environment deploys automatically on every push to main — no approval needed, serving as a quick validation that the deployment process works. The staging environment runs automated smoke tests against the deployed service. The production environment requires approval from two designated approvers AND an automated metrics gate that checks Azure Monitor for error rates below 0.5% before allowing the deployment to proceed. If the metrics gate fails, the deployment is automatically blocked. If the deployment succeeds but error rates spike afterward, a separate monitoring workflow triggers an automatic rollback by redeploying the previous artifact version.

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
  deploy-staging:
    needs: deploy-dev
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-staging
      - run: aws ecs update-service --cluster myapp --service staging --force-new-deployment
  deploy-prod:
    needs: deploy-staging
    environment: prod
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-prod
          aws-region: us-east-1
      - run: aws ecs update-service --cluster myapp --service prod --force-new-deployment
```

- **Scenario 2: Monorepo CI with Dynamic Matrix for Changed Services**

  - A team manages 30 microservices in a single monorepo. Originally, they ran the full CI suite on every commit — linting, testing, and building all 30 services — which took 45 minutes per commit and wasted significant compute. The solution is dynamic matrix generation based on path filtering. A "changes" job runs first, using `dorny/paths-filter` to detect which service directories were modified in the commit. It outputs a JSON array of changed service names. A "build" job then uses a matrix strategy with `fromJSON(needs.changes.outputs.services)` to run CI only for the changed services. If a shared library is modified, all services that depend on it are detected through a dependency graph and added to the matrix.

  - This reduced average CI time from 45 minutes to under 5 minutes for typical changes. The full matrix still runs nightly as a scheduled workflow to catch cross-service integration issues. The key implementation detail is the dependency graph — a JSON file in the repository root maps each shared library to its dependent services, and the changes job consults this graph to expand the matrix beyond just the directly modified services.

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
            shared-lib: 'libs/shared/**'
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

- **Scenario 3: Self-Hosted Runner Auto-Scaling with ARC**

  - A 200-developer organization was spending $15,000/month on GitHub-hosted runners — the bill grew linearly with developer count because each PR triggered CI runs. They migrated to self-hosted runners using the Actions Runner Controller (ARC) on Kubernetes. ARC manages a pool of runner pods that scale automatically based on the GitHub Actions job queue depth. When a workflow is triggered, GitHub dispatches the job to an available runner pod. If no runners are available, the job remains queued, and ARC's metrics-based autoscaler detects the queue depth and creates additional runner pods.

  - Each runner pod is ephemeral — it runs exactly one job, then is destroyed. This prevents cross-job contamination (no stray files or credentials from previous runs) and ensures a clean environment every time. The runner pods use a custom Docker image with all tools pre-cached (Node.js versions, Python, Java, Docker, Go, .NET), reducing job setup time from 60 seconds (fresh GitHub-hosted runner) to under 5 seconds. The Kubernetes cluster uses a dedicated node pool for runners with spot instances for cost savings. Auto-scaling scales to zero when no jobs are queued, and the cluster autoscaler deallocates the underlying VMs. Total cost dropped to $2,000/month — the Kubernetes cluster with spot instances, with the same CI throughput. The security model isolates runners in a separate namespace with no access to production namespaces, and all outbound traffic goes through a NAT gateway with audit logging.

---

## Scenario-Based Questions

- 1. **Q: Design a secure CI/CD pipeline for a financial services company using GitHub Actions.**
   - **A:** A secure pipeline for financial services must address compliance requirements (SOC2, PCI-DSS) while maintaining developer velocity. Use OIDC for all cloud authentication — configure AWS IAM roles or Azure Managed Identities with trust policies that validate GitHub claims (repository, environment, branch) so credentials cannot be used outside the pipeline. Implement environment protection rules for production: require at least two designated approvers, a wait timer of at least 10 minutes (to allow for last-minute rollback decisions), and restrict deployments to the main branch only. Add signed commits verification as a mandatory status check — the pipeline verifies that every commit in the PR is signed with GPG or SSH. Integrate `actions/dependency-review-action` to scan pull requests for dependency changes containing known vulnerabilities. Run CodeQL analysis on every PR to detect security vulnerabilities in application code. Generate an SBOM (Software Bill of Materials) in CycloneDX format for every build and store it as a deploy artifact. Sign all Docker images with Cosign and enforce signature verification in the production container runtime. Enable GitHub's secret scanning on all repositories to detect accidentally committed credentials. Finally, configure immutable deployment records by requiring that every production deployment creates a GitHub deployment record with a link to the artifact version, commit SHA, and approval audit trail.

- 2. **Q: How would you implement monorepo CI with 100+ services that only builds changed services?**
   - **A:** A monorepo with 100+ services requires a multi-stage detection strategy to avoid building everything on every commit. The first job detects which services changed using path filters (`dorny/paths-filter` or a custom script). The filter configuration maps service directories to service names: `auth-service: 'services/auth/**'`. The detection job outputs a JSON array of changed service names. A build job uses a matrix strategy with `fromJSON(needs.changes.outputs.services)` to dynamically generate build matrix entries only for changed services. For shared library changes (e.g., `libs/shared-utils/**`), a dependency graph JSON file maps each shared library to all services that depend on it — when the shared library changes, all dependent services are added to the build matrix. The build job checks out only the relevant service directory (using sparse checkout) to reduce clone time. A caching strategy shared across services speeds up builds: lock file caches are keyed by service and OS, and shared library build outputs are cached for reuse by multiple services. A scheduled nightly workflow runs the full CI suite across all services to catch cross-service incompatibilities that path-based filtering might miss.
  - **Interview follow-up:** How do you detect transitive dependencies — if Service A depends on a shared library that depends on another shared library, how does the path filter discover the full chain?

- 3. **Q: Design a multi-region deployment pipeline with canary analysis and auto-rollback.**
   - **A:** A multi-region canary deployment pipeline minimizes blast radius while enabling rapid global rollouts. The pipeline deploys to a single region first as a canary — routing 10% of traffic to the new version. During the observation window (5-10 minutes), the pipeline monitors latency (p99), error rates, and business-specific metrics (conversion rate, signup completion). If metrics remain within thresholds, the canary is promoted to 100% in the first region, and the pipeline proceeds to the next region. Each region undergoes the same canary process — this sequential regional progression prevents a bad deployment from affecting all users simultaneously. If any metric exceeds its threshold at any point, the pipeline automatically triggers a rollback: traffic is shifted back to the previous version in that region, and subsequent regions are skipped. The rollback is executed by the same deployment pipeline — it redeploys the previous artifact version rather than requiring a manual revert. For implementation in GitHub Actions, each region's deployment is a separate job in the workflow (or a separate workflow called via `workflow_call`), with the canary and full-traffic stages as steps within each job. CloudWatch or Datadog metrics are queried from the pipeline using cloud-specific actions or curl commands against monitoring APIs.

- 4. **Q: How do you manage secrets across multiple environments and cloud providers?**
   - **A:** Secret management in GitHub Actions follows a tiered model. At the repository level, secrets are visible to all workflows in the repository — suitable for non-sensitive values shared across environments. At the environment level, each GitHub Environment (dev, staging, prod) has its own independent secret store — database passwords, API keys, and service connection strings are stored per-environment so that a staging compromise does not expose production credentials. At the organization level, organization secrets are shared across multiple repositories, useful for values like registry credentials or shared API tokens. For cloud authentication, OIDC eliminates static secrets entirely — instead of storing an AWS secret key or Azure service principal password, the pipeline authenticates using a short-lived token that GitHub exchanges with the cloud provider. The cloud provider's IAM trust policy validates that the token was issued for the correct repository, branch, and environment. The golden rule: secrets must never be reused across environments; a secret used in staging should be different from the same logical secret in production. Use GitHub's secret scanning to detect accidentally committed secrets, and rotate secrets regularly using automation (GitHub's REST API for secrets can be scripted).
  - **Interview follow-up:** If a developer forks your repository and opens a PR, their fork's workflow will fail because secrets aren't available to forks — how do you allow external contributors to get CI feedback without exposing secrets?

- 5. **Q: Design a self-hosted runner auto-scaling solution.**
   - **A:** A production-grade self-hosted runner solution uses the Actions Runner Controller (ARC) on Kubernetes. ARC manages runner pods that register with GitHub and process workflow jobs. The autoscaling is based on GitHub's job queue: when jobs are queued and no runners are available, ARC creates additional runner pods. Each runner is ephemeral — it processes exactly one job, then the pod is terminated. This ensures a clean environment for every job (no lingering files, no cross-job contamination, no stale credentials). The runner pods use a pre-cached Docker image with all common tools installed — Node.js, Python, Java, Docker, .NET, Go — reducing job setup time from 60 seconds on a fresh GitHub-hosted runner to under 5 seconds. The Kubernetes cluster uses a dedicated node pool for runners, with spot instances for cost savings and cluster autoscaler to scale the node pool. Network security isolates runner pods from production workloads: they run in a separate namespace with network policies that restrict egress to approved endpoints (registry, artifact storage, monitoring) and block ingress entirely. For organizations without Kubernetes, the alternative is VM-based runners using Terraform or AWS Auto Scaling Groups that register new runner instances when the GitHub API reports pending jobs.

- 6. **Q: How would you implement blue-green deployment with traffic shifting?**
   - **A:** Blue-green deployment in GitHub Actions deploys the new application version to a parallel "green" environment while the current "blue" environment continues serving production traffic. The pipeline begins by deploying the new version to the green environment — this is identical infrastructure (same compute, same configuration, same database) but with the new application code. Smoke tests run against the green environment to verify health endpoints respond, database connections work, and critical API paths return correct results. After smoke tests pass, the pipeline shifts traffic by updating the load balancer or service discovery to route traffic from blue to green. In AWS, this means updating an ECS service's task definition or an ALB's target group. In Kubernetes, this means updating the service selector or using a service mesh's traffic split configuration. The blue environment remains running to enable instant rollback — if issues are detected after the switch, the pipeline flips traffic back to blue. Implement this in GitHub Actions using deployment environments: the blue environment tracks the current production version, and the green environment tracks the new version. Use deployment status checks to monitor the health of both environments. After a stabilization period (30-60 minutes), the pipeline can optionally decommission the old blue environment and reprovision it as the next green target.
  - **Interview follow-up:** How do you handle database schema changes during a blue-green deployment when both environments must be able to read and write to the same database simultaneously?

- 7. **Q: Design a database migration pipeline with rollback capability.**
   - **A:** A database migration pipeline must ensure that schema changes can be safely applied and reverted without data loss or downtime. The pipeline is triggered by pushes to a `migrations/` directory in the repository — not by every application code change, because database migrations are deployed on a different cadence. The first stage runs migrations against a staging database that mirrors the production schema and data (anonymized). If the migration fails at this stage, the pipeline stops, and the developer is notified before any production impact occurs. The second stage validates schema integrity — checking indexes, constraints, and data types against expected definitions. The third stage deploys the application code that is compatible with both old and new schema (backward compatibility is a hard requirement). The fourth stage runs migrations against the production database, one transaction at a time, with monitoring for lock contention and query performance degradation. Each migration script has a corresponding rollback script, and the pipeline includes a `rollback` job that can be triggered manually or automatically. The expand-contract pattern is enforced: additive changes (new columns, new tables) run before the application deploy, and destructive changes (column removals, table drops) run in a separate pipeline execution hours or days later. Flyway or Liquibase manage migration versioning and track which migrations have been applied to each database.

- 8. **Q: How do you optimize GitHub Actions costs for a 200-developer organization?**
   - **A:** Cost optimization for a large GitHub Actions deployment requires multiple strategies working together. The highest-impact change is migrating from GitHub-hosted to self-hosted runners for the primary CI workload — at 200 developers generating thousands of CI runs per day, the cost savings are substantial (often 70-80% reduction). Use ARC on Kubernetes with spot instances to minimize infrastructure costs while maintaining autoscaling capability. Path filters are the second most impactful optimization — skip CI entirely for documentation, configuration, and non-code changes; use `paths-ignore: ['docs/**', '*.md', 'config/**']` on the main CI workflow. Concurrency control cancels stale runs on the same branch, eliminating wasted compute from rapid pushes. Limit matrix builds to the minimum necessary combinations — test only LTS versions of languages on Linux, not every version on every OS. Set artifact retention to 7 days (the minimum) for most artifacts, extending to 30 days only for audit-required artifacts. Cache dependencies aggressively using `actions/cache` with restore keys based on lock file hashes — this reduces the time (and cost) of dependency installation on every run. Monitor workflow usage monthly using GitHub's billing API or the Actions usage dashboard to identify the most expensive workflows and optimize them.

- 9. **Q: Design a pipeline that enforces SLSA Level 3 compliance.**
   - **A:** SLSA (Supply-chain Levels for Software Artifacts) Level 3 requires that builds are hermetic, provenance is attested, and the build platform is secured against tampering. A hermetic build means the build process has no network access — all dependencies must be pre-fetched and verified before the build starts, and the build must fail if it attempts any network connection. In GitHub Actions, this is enforced by specifying `container:` without network access and using `actions/cache` to pre-populate all dependencies. Provenance attestation means the build process generates a signed statement describing how the artifact was built, including the builder identity, build instructions, source repository, and commit hash. Use `slsa-framework/slsa-github-generator` to generate SLSA provenance — this action produces a signed attestation that is stored alongside the artifact in the registry. Verifiable build steps means the pipeline definition is in a version-controlled file, and the build process does not allow runtime code injection — no `pull_request_target` that executes untrusted code. Immutable build logs are captured and stored in an append-only store for audit purposes. Dependency review (`actions/dependency-review-action`) scans all dependencies before they are used in the build. Reproducible builds are the ideal: given the same source code and build instructions, the build produces byte-for-byte identical artifacts. While not all builds can be reproducible, the pipeline should strive for deterministic builds by pinning all tool versions and avoiding timestamp-based or random-output during compilation.

- 10. **Q: How would you implement feature flag deployments using GitHub Actions?**
    - **A:** Feature flag deployments separate the act of deploying code from the act of releasing features. The pipeline deploys the application with new code hidden behind flags that are initially off. After deployment, the pipeline interacts with the feature flag platform (LaunchDarkly, Flagsmith, Unleash) to progressively enable the feature. The steps are: deploy the application as usual (the new code is present but inactive), verify the deployment is healthy with standard health checks, enable the flag for internal testing (a specific user segment), monitor metrics for a configurable observation period, gradually ramp the flag to a percentage of users (1%, 10%, 25%, 50%, 100%), and finally — if metrics remain healthy — mark the flag as fully released. If any metric degrades during the ramp, the pipeline automatically toggles the flag off (instant rollback without redeployment). In GitHub Actions, this is implemented as a post-deployment workflow that calls the feature flag platform's REST API or uses a community action (like `launchdarkly-actions/flag-evaluate` or a custom curl command). The workflow includes wait steps (`github script` with a sleep or a dedicated wait action) for the observation periods. Each ramping step is a separate job with manual confirmation or automated metrics verification before proceeding to the next level.


---

## Interview Questions

- 1. **What is the difference between a workflow, job, and step in GitHub Actions?**
   - **A:** A **workflow** is the top-level automation unit — a YAML file in `.github/workflows/*.yml` that defines a complete automated process triggered by GitHub events. A workflow can contain multiple **jobs**, each running on a separate runner. Jobs within a workflow execute in parallel by default unless dependencies are declared with the `needs` keyword. Each job consists of **steps** — individual tasks that execute sequentially within that job's runner environment. Steps can be shell commands (`run:`) or reusable actions (`uses:`). An **action** is a reusable unit of automation that encapsulates one or more steps — it can be JavaScript (runs directly on the runner), Docker (runs in a container), or composite (combines multiple steps). The hierarchy is: Workflow includes Jobs (parallel or sequential), Jobs include Steps (always sequential), and Steps can be shell commands or Actions.

- 2. **What is the difference between `push`, `pull_request`, and `workflow_dispatch` triggers?**
   - **A:** The `push` trigger fires when commits are pushed to a branch — it runs the workflow against the pushed code. Use `push` for CI on the main branch and for workflows that need to run on every commit regardless of PR status. The `pull_request` trigger fires when PR events occur (opened, synchronized, reopened) — it runs the workflow against the merge commit (the result of merging the PR branch into the base branch). Use `pull_request` for PR validation checks: linting, testing, and build verification that must pass before merging. The `workflow_dispatch` trigger allows manual execution from the GitHub UI — it can accept input parameters defined in the workflow. Use `workflow_dispatch` for ad-hoc operations (manual deployments, database migrations, data exports) that should not run automatically. Best practice: use `pull_request` for CI checks (they run against the merge result and appear as PR status checks), use `push` to main for deployment workflows, and use `workflow_dispatch` for administrative operations.

- 3. **What is a matrix strategy and when would you use it?**
   - **A:** A matrix strategy (`strategy.matrix`) runs a job across multiple combinations of variables simultaneously, creating a separate job instance for each combination. For example, a matrix with `os: [ubuntu-latest, windows-latest]` and `node: [18, 20]` produces four job instances: ubuntu+18, ubuntu+20, windows+18, windows+20. Use matrices for: cross-platform testing (verify the application works on Linux, Windows, and macOS), multi-version compatibility (test against multiple language versions), and environment-specific deployments (deploy to dev, staging, and production with different configurations). The `include` keyword adds additional combinations outside the Cartesian product, and `exclude` removes specific combinations. The `fail-fast` option (default: true) cancels all matrix jobs when any job fails; set it to `false` when you want all combinations to complete even if some fail (useful for collecting results from all test environments). Limit matrix size to avoid excessive runs — a matrix with 5 OS choices and 10 version choices creates 50 job instances.

- 4. **What is the difference between GitHub-hosted and self-hosted runners?**
   - **A:** GitHub-hosted runners are managed by GitHub — they are ephemeral VMs with pre-installed tools (Node.js, Python, Java, Docker, .NET, Go, and many more) that are automatically provisioned, run one job, and are then destroyed. They require zero maintenance, support automatic scaling to hundreds of concurrent jobs, and are free within monthly limits. The trade-offs: no network access to your internal infrastructure, limited execution time (6 hours per job for Linux, 5 for Windows, 5 for macOS), and cost at scale ($0.008/min for Linux 2-core). Self-hosted runners are installed on your own infrastructure — physical servers, cloud VMs, or Kubernetes pods. They provide full network access (reach internal databases and services), unlimited execution time, custom hardware configurations, and pre-cached tools. The trade-offs: you are responsible for maintenance (OS updates, security patching, tool version updates), security isolation (self-hosted runners execute arbitrary code), and capacity planning. The standard recommendation: use GitHub-hosted runners for open-source projects, small teams, and workflows that do not need network access; use self-hosted runners for large organizations, workflows that need internal network access, or cost optimization at scale.

- 5. **How does OIDC work in GitHub Actions?**
   - **A:** OIDC (OpenID Connect) allows GitHub Actions workflows to authenticate directly with cloud providers (AWS, Azure, GCP) without storing any static credentials. The workflow requests an OIDC token from GitHub's OIDC provider (`token.actions.githubusercontent.com`). The token contains claims about the workflow run: the repository (`sub: repo:org/repo`), the branch (`ref: refs/heads/main`), the environment (if configured), and the job name. The cloud provider is configured to trust GitHub's OIDC provider through a trust policy or identity provider configuration. When the workflow runs a cloud authentication action (like `aws-actions/configure-aws-credentials` or `azure/login`), it presents the OIDC token to the cloud provider. The cloud provider validates the token's signature using GitHub's OIDC public keys, then verifies the claims against the trust policy (checking that the token was issued for the expected repository, branch, and environment). If the claims match, the cloud provider issues short-lived credentials (typically 1 hour) scoped to a specific IAM role or managed identity. No static secrets are stored in GitHub — the trust is established through the token's cryptographic signature and the cloud provider's trust policy.

- 6. **What is the `actions/cache` action and how does it improve build times?**
   - **A:** `actions/cache` is a first-party GitHub Action that caches dependencies and build outputs to speed up subsequent workflow runs. It works by key-based cache lookup: a `key` is provided (typically based on the hash of a lock file and the OS), and if the key matches an existing cache, the cached files are restored. If the key does not match (cache miss), the cache is saved at the end of the job for future runs. The key format `${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}` ensures the cache is specific to both the OS and the exact dependency versions. A `restore-keys` fallback allows partial cache hits — if no exact key match is found, the most recent cache matching the prefix is restored, which is still faster than a full install. Common caching targets: npm/node_modules (60-80% reduction in install time), Maven/Gradle (.m2, .gradle — 50-70%), Go module cache (~40%), Docker layers, and compiled build outputs. Cache is immutable once saved: if the key changes, a new cache is created; the old cache expires after 7 days of no access. Maximum cache size is 10 GB per repository. The trade-off: cache restoration takes a few seconds, so caching is beneficial only when the time saved exceeds the restoration overhead (typically for caches larger than a few megabytes).

- 7. **What is a reusable workflow vs a composite action?**
   - **A:** Reusable workflows and composite actions both enable code reuse in GitHub Actions, but at different levels. A **reusable workflow** is a complete `.github/workflows/*.yml` file with `on: workflow_call` that defines one or more jobs, each with multiple steps. It can accept `inputs` (typed parameters like string, number, boolean) and `secrets` (sensitive values passed securely). Reusable workflows are called from other workflows using the `uses:` syntax: `uses: org/.github/.github/workflows/ci.yml@v1`. They are ideal for sharing complete CI/CD patterns across an organization — a "CI" reusable workflow that runs linting, testing, and building for any project. A **composite action** is an `action.yml` file that defines a single job's worth of steps as a reusable unit. It can accept `inputs` (strings) but cannot define jobs, use matrix strategies, or access secrets. Composite actions are called as a step: `uses: org/repo/path/to/action@v1`. They are ideal for sharing step-level logic — a "setup-node" composite action that installs Node.js, configures npm, and caches dependencies. Use reusable workflows for complete pipeline patterns, composite actions for reusable step collections.

- 8. **How do environment protection rules work?**
   - **A:** GitHub Environments are logical deployment targets (dev, staging, prod) that provide security controls for deployments. Each environment has configurable protection rules. **Required reviewers** specify GitHub users or teams who must approve each deployment before it proceeds — approvals are visible in the deployment audit trail. This enforces separation of duties: the developer who created the deployment cannot be the sole approver. **Wait timer** adds a mandatory delay (in minutes) before the deployment is allowed to execute, giving the team time to cancel a mistaken deployment. **Deployment branches** restrict which branches can deploy to the environment — typically only the main branch can deploy to production. Environments also scope secrets: each environment has its own set of secrets that are only exposed to jobs targeting that environment. A job targets an environment with the `environment:` keyword: `environment: production`. The protection rules are enforced before the job starts: GitHub checks that the branch is allowed, waits for the timer, and waits for all required reviewers to approve. The deployment is recorded in GitHub's deployment API with a unique ID, linking the deployment to the environment, the commit, and the approval history.

- 9. **What is the difference between `needs` and `if` conditions in workflows?**
   - **A:** The `needs` keyword creates a structural dependency between jobs: `job_b` with `needs: job_a` will not start executing until `job_a` has completed. This is mandatory for any workflow where later jobs depend on outputs or artifacts from earlier jobs. `needs` supports arrays: `needs: [build, test, lint]` means all three must complete before the dependent job starts. The `if` keyword conditionally executes a job based on a runtime expression — it does not create dependencies. `if: github.ref == 'refs/heads/main'` means the job only runs when the trigger branch is main. `if: success()`, `if: failure()`, and `if: always()` provide status-based conditions that are evaluated after all dependencies complete. The two are commonly combined: `needs: [build, test]` ensures the deployment job waits for both build and test to complete, then `if: success()` ensures the deployment only runs if both dependencies succeeded. A job with `if: failure()` can be used to send notifications when any dependency fails.

- 10. **How do you handle concurrency in GitHub Actions?**
    - **A:** Concurrency control in GitHub Actions prevents multiple workflow runs from conflicting with each other. The `concurrency` keyword is defined at the workflow level and accepts a `group` name (string) and optional `cancel-in-progress` (boolean). The group name determines which runs are considered concurrent: runs with the same group are in the same concurrency queue. The standard pattern for CI workflows is `group: ${{ github.workflow }}-${{ github.ref }}` with `cancel-in-progress: true` — this groups runs by workflow name and branch reference, and cancels any in-progress run when a new push arrives on the same branch. For deployment workflows where you do not want to cancel in-progress deployments, use the same pattern without cancel-in-progress (runs queue up and execute sequentially) or use a fixed group name like `group: deploy-production` to serialize all production deployments regardless of branch. Concurrency is evaluated when the workflow is triggered — if a run with the same group is in progress, the new run is either queued (waiting for the previous run to complete) or the previous run is canceled, depending on the `cancel-in-progress` setting. This prevents wasted runs on rapid pushes and prevents deployment pipelines from stepping on each other.

---

## Developer Recommendations

- **Use OIDC over static secrets for all cloud authentication** — Static cloud credentials (AWS access keys, Azure service principal secrets, GCP service account keys) present a continuous security and operational burden. They must be stored securely, rotated regularly, and audited for use. If a static key is accidentally exposed (in a log, a debug output, or a leaked secrets file), an attacker can use it until it is rotated. OIDC eliminates all of these problems by replacing static credentials with short-lived, automatically scoped tokens. Configure each cloud provider to trust GitHub's OIDC provider once, then every workflow in the organization can authenticate without storing any cloud credentials. The IAM trust policy is the security boundary: it validates that the token was issued for the correct repository, environment, and branch. Lock down production roles to only accept tokens from the main branch and the production environment. The setup effort is one-time per cloud provider, and the security benefit is permanent.

- **Pin all actions to commit SHAs, not version tags** — A version tag like `actions/checkout@v4` is a mutable reference. If the `v4` tag is reassigned to a malicious version, your workflow silently executes the compromised code. Pinning to the full commit SHA (`actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11`) ensures the exact code is used regardless of what happens to the tag. This is especially critical for third-party actions from unfamiliar publishers, but it is a best practice even for official GitHub actions. Use Dependabot to automate SHA updates — Dependabot monitors the source repository for new releases, computes the new SHA, and opens a PR with the updated pin. Review the PR to see the release notes and verify the action's changes before merging. For actions you develop internally, apply the same practice: use SHA-pinned references even within your organization.

- **Use path filters to avoid unnecessary CI runs** — Running the full CI pipeline on every commit regardless of what changed is wasteful. A change to `README.md` should not trigger a 15-minute CI pipeline that deploys to staging. Use `paths-ignore` on the workflow trigger to exclude documentation, configuration, and non-code files: `paths-ignore: ['*.md', 'docs/**', 'config/**', '.github/**']`. For monorepos, use `paths` on individual jobs or use a detection job (like `dorny/paths-filter`) to dynamically determine which jobs need to run. The savings are substantial: a typical project can reduce CI runs by 30-50% with proper path filtering. Each skipped CI run saves not just compute time but also developer waiting time and CI/CD bill costs.

- **Cache everything, but cache wisely** — Dependency caching is the single most impactful CI optimization, reducing install times from minutes to seconds. However, incorrect caching can cause subtle bugs or waste storage. The key must accurately reflect the dependency state — always include the lock file hash and OS in the cache key: `key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}`. Provide a `restore-keys` fallback that is less specific to handle cache misses gracefully: `restore-keys: ${{ runner.os }}-npm-` (restores the most recent cache for that OS if no exact match is found). Be careful what you cache — caching the build output directory (`dist/`, `build/`, `out/`) can hide build errors if the cache is stale. Prefer caching only immutable dependency directories (node_modules, .m2, ~/.cache/pip) and let build outputs be regenerated each time. Set cache limits per key to avoid filling the 10 GB repository quota. Monitor cache hit rates and adjust key strategies if hit rates are low.

- **Use environments for production deployments, not manual conditions** — Relying solely on `if: github.ref == 'refs/heads/main'` to gate production deployments is weak — a modified workflow file could accidentally remove or change that condition. GitHub Environments provide a stronger security boundary: environment protection rules (required reviewers, deployment branch restrictions, wait timer) are enforced by GitHub's infrastructure, not by the workflow logic. Configure environments for every deployment target, even if protection rules are initially empty — this allows adding rules later without workflow changes. Each environment has its own secret store, ensuring production credentials are never exposed to non-production jobs. The deployment record created by environment-targeted jobs is visible in the GitHub UI and the API, providing an audit trail of who deployed what and when. The rule: every production deployment goes through an environment with at least one required approver and a branch restriction to main.

- **Prefer self-hosted runners at scale, but with caution** — At 50+ active developers, GitHub-hosted runner costs typically exceed the cost of self-hosted runners. The break-even point depends on usage patterns, but for organizations running 500+ CI hours per month, self-hosted runners are significantly cheaper. Use Actions Runner Controller (ARC) on Kubernetes for autoscaling ephemeral runners — each job gets a fresh runner pod that is destroyed after the job, preventing cross-job contamination. Pre-cache all common tools in the runner Docker image to eliminate tool installation time. However, self-hosted runners introduce security risks: any workflow that runs on the runner can execute arbitrary code. Never run untrusted PR workflows on self-hosted runners — use GitHub-hosted runners for PRs from forks. Isolate self-hosted runners in a dedicated network segment with no access to production infrastructure. Use runner group permissions to restrict which repositories and which workflows can use which runner groups. Implement regular security patching for runner host operating systems and tools.

---

## Summary

  - GitHub Actions provides a deeply integrated CI/CD platform that leverages GitHub's existing ecosystem of repositories, pull requests, and branch protection. Its key strengths are the tight integration with GitHub events, the extensive marketplace of pre-built actions, and the flexibility of supporting both GitHub-hosted and self-hosted runners. Effective use requires understanding the workflow DAG model, mastering expressions and contexts for conditional logic, implementing proper security through OIDC authentication and action pinning, and optimizing for cost and speed through caching, path filtering, and concurrency control. As organizations mature, reusable workflows enable consistent CI/CD patterns across hundreds of repositories, while environment protection rules and deployment tracking provide the governance needed for production deployments in regulated environments.