# Azure DevOps

---

## Overview

- **Definition:** Azure DevOps is Microsoft's end-to-end DevOps platform providing integrated services for planning, developing, delivering, and maintaining applications.
  - Unifies the entire software delivery lifecycle into a single toolchain: Azure Boards, Azure Repos, Azure Pipelines, Azure Test Plans, and Azure Artifacts
  - Eliminates friction of stitching together multiple third-party tools with consistent authentication, permissions, and reporting
  - Available as cloud service (Azure DevOps Services) with Microsoft-managed infrastructure and as on-premise server (Azure DevOps Server)

- **Why It Exists:** Before Azure DevOps, Microsoft had separate tools (Team Foundation Server, Jenkins, NuGet) requiring complex integrations and custom scripting.
  - Azure DevOps unified these capabilities with consistent authentication through Azure AD, unified permissions, and cross-service traceability — from work item to code commit to build to release
  - Deep integration with Azure (App Service, AKS, Azure Functions, VMs) and extensibility through 1,000+ marketplace extensions
  - Supports both cloud-hosted and self-hosted agent pools for control over build infrastructure

- **Key Concepts:**
  - **Organization** — The top-level container holding all projects, billing, and policies, typically representing the entire company
  - **Project** — A logical container for code repositories, work items, pipelines, and test plans, usually corresponding to a product or team
  - **Teams** — Subsets of a project with their own area paths, backlogs, and dashboards for independent work
  - **Agent Pools** — Collections of build and release agents executing pipeline jobs; Microsoft-hosted (zero maintenance, 60-minute limit) or self-hosted (custom hardware, on-premise access)
  - **Variable Groups** — Store variables shared across multiple pipelines with integration to Azure Key Vault for secure secret management
  - **Service Connections** — Provide secure, auditable access to external services like Azure subscriptions, GitHub, Docker registries, and Jenkins servers
  - **Environments** — Deployment targets with approval gates, resource tracking, and deployment history

---

## Core Services

- **Azure Boards** — Work tracking with Kanban boards, backlog management, sprint planning, and customizable dashboards supporting Scrum, Agile, and CMMI process templates.
  - Work items hierarchically organized as Epics, Features, User Stories, and Tasks with customizable fields and states
  - Deep integration with Azure Repos through automatic work item linking — including "AB#1234" in a commit message links the commit to work item 1234
  - Provides traceability: what work items are in this release, which commits fixed this bug
  - Supports GitHub integration for teams using GitHub for code with Azure Boards for work tracking

- **Azure Repos** — Git repositories with branch policies, pull request workflows, and code reviews.
  - Branch policies enforce quality gates: minimum reviewers, linked work items, successful build validation, comment resolution
  - Ensures every merge to main is reviewed, built successfully, and linked to an active work item
  - Supports TFVC for legacy systems but strongly recommends Git for all new projects
  - Integrates with Azure Pipelines through PR triggers, automatically building each pull request

- **Azure Pipelines** — CI/CD engine supporting multi-stage YAML pipelines with container support, matrix builds, and deployment strategies (canary, blue-green, rolling).
  - YAML pipeline definition lives in the repository alongside code for version control and code review
  - Builds and deploys to any platform: Windows, Linux, macOS; supports .NET, Java, Node.js, Python, Go, C++
  - Deployment jobs integrate with Environments for approval gates and resource-specific deployment history
  - Pipeline caching reduces build times by caching dependencies across runs

- **Azure Test Plans** — Manual and exploratory testing with test case management and rich execution reporting.
  - Test plans created from requirements and user stories for end-to-end traceability
  - Exploratory testing extension captures screenshots, screen recordings, and annotated notes
  - Test execution analytics provide pass rates, flaky tests, and coverage gaps

- **Azure Artifacts** — Package management for NuGet, npm, Maven, Python, and Universal Packages with upstream source proxying.
  - Integrates with Azure Pipelines for automated package publishing and consumption
  - Enables "Build Once, Deploy Many" pattern where each CI run produces versioned packages promoted through environments
  - Upstream sources enable a single feed for both internal and external packages
  - Retention policies automatically clean up old package versions

```yaml
trigger:
  branches: { include: [main, develop] }
  paths: { exclude: [docs/*, README.md] }

variables:
  - group: 'Global-Variables'
  - name: buildConfiguration
    value: 'Release'

stages:
  - stage: CI
    jobs:
      - job: Build
        steps:
          - script: npm ci && npm run build
          - task: PublishBuildArtifacts@1

  - stage: Deploy_Dev
    dependsOn: CI
    condition: succeeded()
    jobs:
      - deployment: Deploy
        environment: dev
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1

  - stage: Deploy_Prod
    dependsOn: Deploy_Dev
    condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')
    jobs:
      - deployment: Deploy
        environment: prod
        strategy:
          canary:
            increments: [10, 50, 100]
```

---

## Common Mistakes

- **Using classic build and release pipelines instead of YAML pipelines** — Classic pipelines are configured through the web UI and stored in Azure DevOps, not in the repository, preventing version control, code review, and branching of pipeline changes.
  - **Why it looks correct:** The visual editor is intuitive, requires no YAML knowledge, and gives immediate feedback through drag-and-drop.

- **Not using variable groups linked to Azure Key Vault** — Leads to secret sprawl where connection strings, API keys, and passwords are hardcoded in pipeline YAML or stored as plain-text library variables.
  - **Why it looks correct:** Storing secrets as plain-text library variables works and is simple — the security implications aren't immediately obvious.

- **Hardcoding subscription IDs in pipeline YAML** — Makes YAML environment-specific and unreusable across subscriptions; if a subscription ID changes, every pipeline referencing it must be updated.
  - **Why it looks correct:** The subscription ID is a non-secret identifier — pasting it directly in YAML seems harmless and avoids service connection setup.

- **Deploying without staging slots** — Causes downtime during deployments because the app must be stopped, updated, and restarted rather than using zero-downtime slot swaps.
  - **Why it looks correct:** Deploying directly to production is the simplest approach — 30 seconds of downtime during restart seems acceptable until you get paged for every deployment.

- **Not using path filtering in pipeline triggers** — Causes unnecessary pipeline runs on documentation changes, README updates, and config changes that don't affect application code.
  - **Why it looks correct:** Running the full pipeline on every push feels thorough and safe — filtering seems like an optimization you don't need until the CI queue grows to hours.

- **Skipping environment approvals for production deployments** — Enables accidental deployments to production, causing outages, data loss, and compliance violations.
  - **Why it looks correct:** Approvals slow down the pipeline and seem like unnecessary bureaucracy when you trust your team and your automated tests.

- **Not setting artifact retention policies** — Causes storage costs to grow unboundedly as every build artifact is retained forever, consuming space and making traceability harder.
  - **Why it looks correct:** Artifacts feel valuable — deleting them seems risky, and storage is cheap, so keeping everything seems like a safe default.

- **Not using YAML templates** — Leads to duplicated pipeline code across teams, creating maintenance nightmares when shared steps need updating.
  - **Why it looks correct:** Copying an existing pipeline YAML that works is faster than learning template syntax — the duplication cost only becomes visible when updating 50 pipelines.

- **Using a single agent pool for all workloads** — Causes resource contention where long-running builds block quick CI feedback loops.
  - **Why it looks correct:** One pool is simpler to manage — contention only becomes visible as developers wait longer for CI feedback.

- **Not implementing pipeline caching** — Forces every build to re-download the same dependencies from the internet, wasting time and bandwidth.
  - **Why it looks correct:** Downloading dependencies is just part of the build process — wasted time is only apparent when measured against a cached run.

---

## Key Design Considerations

- **YAML over Classic pipelines** is the foundational decision for any Azure DevOps implementation.
  - YAML pipelines live in the repository for version control, code review, and branching
  - Classic pipelines are stored in Azure DevOps — unreviewable, unversioned, prone to drift
  - YAML enables reusability through parameterized templates stored in a central Git repository
  - Treat migration from classic to YAML as technical debt reduction with phased approach

- **Agent strategy** should balance cost, control, and maintenance overhead.
  - Microsoft-hosted agents: zero maintenance, common tools pre-installed, auto-scaling, 60-minute limit, no VNet access
  - Self-hosted agents: full control, custom hardware, on-premise network access, no time limits, operational overhead
  - For 50+ active developers, self-hosted agents using VM Scale Sets provide the best balance with auto-scaling and scale-to-zero

- **Secrets management** should follow the principle that no secret should ever appear in pipeline YAML, repository code, or build logs.
  - Variable groups linked to Azure Key Vault provide the most secure approach with automatic rotation and masked logs
  - Service connections should use Managed Identity or Workload Identity Federation where possible
  - Secure files handle certificates and SSH keys that aren't simple strings
  - The small startup delay for secret retrieval is vastly outweighed by elimination of secret sprawl

- **Deployment strategies** must be chosen based on application architecture, risk tolerance, and rollback speed.
  - `runOnce`: simplest, deploy and verify health — suitable for dev environments
  - `canary`: progressively shift traffic (10% → 50% → 100%) with automatic rollback on health check failure
  - `rolling`: replace instances in batches maintaining capacity throughout deployment
  - `blueGreen`: two full environments, deploy to inactive then swap — instant rollback but double resource cost
  - For App Service, deployment slots provide built-in blue-green with slot swap and warm-up

- **Governance in Azure DevOps** should enforce quality and compliance without blocking developer velocity.
  - Branch policies protect main and release branches: require PR reviews, linked work items, successful build, comment resolution
  - Environment approval gates ensure sensitive deployments require authorized approvals
  - YAML templates enforced by DevOps team ensure all pipelines include mandatory steps
  - Audit logging records all pipeline runs, approvals, and configuration changes

- **Enterprise setup** should follow a single-organization model with multiple projects organized by business domain.
  - Single organization enables cross-project visibility, shared agent pools, and centralized policies
  - Managed identities for service connections eliminate service principal secret management
  - Self-hosted agents with auto-scaling provide consistent performance
  - Standardized YAML templates and variable groups maintained by central platform engineering team

---

## Real-World Scenarios

- **Scenario 1: Enterprise Migration from Classic to YAML Pipelines**
  - **Context:** A large enterprise has 500+ classic build and release pipelines that are unversioned, fragile, and difficult to maintain because changes require navigating the web UI.
  - **Resolution:** Create a YAML template library in a central Git repository with reusable job templates. Each team converts pipelines incrementally, starting with simple build-only pipelines. Pipeline decorators enforce organization-wide governance (license scanning, container vulnerability scanning, SBOM generation). Variable groups linked to Key Vault replace hardcoded secrets. A deprecation dashboard tracks progress, and classic pipelines are forcibly retired after 6 months.

```yaml
# Central template: deploy-template.yml
parameters:
- name: environment
  type: string
  values: [dev, staging, prod]
- name: approvalRequired
  type: boolean
  default: true

jobs:
- deployment: Deploy
  environment: ${{ parameters.environment }}
  strategy:
    runOnce:
      deploy:
        steps:
        - task: AzureWebApp@1
          inputs:
            azureSubscription: 'service-connection-${{ parameters.environment }}'
            appName: 'myapp-${{ parameters.environment }}'
        - script: smoke-test.sh ${{ parameters.environment }}
```

- **Scenario 2: Multi-Environment Release with Compliance Gates**
  - **Context:** A FinTech company requires multi-person approval, security scanning, and compliance verification before any code reaches production.
  - **Resolution:** Multi-stage YAML pipeline promoting a single build artifact through each environment. Dev: automatic CI validation (unit tests, integration tests, code coverage). Staging: comprehensive security scan (Trivy, OWASP ZAP, dependency check). Pre-Prod: manual approval gates (QA lead, compliance officer) plus Azure Monitor query gate. Production: change advisory board approval, canary deployment starting at 10% traffic, automatic rollback if thresholds exceeded. The same immutable build artifact (SHA256 digest) is promoted through every environment.

- **Scenario 3: Monorepo CI/CD with 80+ Microservices**
  - **Context:** A product team manages 80 microservices in a single Git repository, each in its own folder under `services/`. Without optimization, every commit triggers CI for all 80 services.
  - **Resolution:** Use path triggers to only build changed services. A YAML template matrix dynamically generates jobs based on changed paths using a file-change detection script. Fan-out pattern launches parallel builds for each changed service with cached npm packages and Docker layers. Fan-in pattern consolidates deployment — only changed services' artifacts are deployed to environments relevant to the commit's branch. Average CI time reduced from 4 hours to 5 minutes.

---

## Scenario-Based Questions

- **Q: Design a PCI-compliant CI/CD pipeline using Azure DevOps.**
  - **A:** Use self-hosted agents in a restricted network segment for isolation from the public internet. All sensitive values stored in Azure Key Vault referenced through variable groups. Environment approvals enforce minimum two approvers for production deployments. Central YAML templates ensure mandatory compliance steps: static code analysis, dependency vulnerability scanning, container image scanning with Trivy, and SBOM generation. Artifact retention policies keep production artifacts for at least one year. Audit logging enabled at organization level. Signed commits enforced through branch policies. A single immutable build artifact is promoted through Dev, QA, Staging, and Production without ever being rebuilt.

- **Q: How do you implement blue-green deployment with App Service slots?**
  - **A:** The production slot runs the current version; the staging slot is the "green" environment. The YAML pipeline deploys to staging, runs smoke tests, then swaps staging and production. The swap is instant (routing rule change at Azure Front Door level). After swap, the old version resides in staging for instant rollback by swapping again. Slot-sticky settings (connection strings, app configurations) remain with their slots. Auto-swap enables fully automated blue-green deployments after successful warm-up.
  - **Interview follow-up:** What happens during a slot swap when the staging slot's application needs to connect to the production database for warm-up — how do you prevent it from writing to the production database during warm-up?

- **Q: Design a multi-region deployment pipeline with release gates.**
  - **A:** Divide deployment into regional stages executing sequentially with health validation gates between each. First region receives deployment, runs integration tests, verifies health metrics through Azure Monitor query gates. Approval gate may be required before proceeding. Subsequent regions deploy using the same process. Each region has its own environment with scoped service connections and approval history. Azure Front Door routes traffic to only healthy regions. Use ring-based deployment: Ring 0 (canary, single region, 5% traffic), Ring 1 (3 regions, 25% traffic), Ring 2 (all regions, 100% traffic). Automated rollback at each ring.

- **Q: How would you manage Terraform infrastructure as code in Azure DevOps?**
  - **A:** Follow a validate-plan-apply workflow with environment isolation and state management. Validate stage: `terraform init`, `terraform validate`, `terraform fmt`. Plan stage: `terraform plan` published as artifact for review. Apply runs automatically for non-production after manual approval; production requires multiple approvers. Terraform state stored in Azure Storage with separate container per environment and state locking. Each environment uses separate backend configuration. Terraform workspaces for additional isolation. Optional destroy stage for ephemeral environments.
  - **Interview follow-up:** Two developers run Terraform plan simultaneously, both see the same state, and both approve. What happens when the second apply runs?

- **Q: Design a self-hosted agent auto-scaling solution.**
  - **A:** Azure VM Scale Set agents with a custom VM image containing all required tools pre-installed and cached. Scale set auto-scales based on Azure DevOps pipeline queue depth. Scale-out: add instances aggressively (2 per queued job, max 20). Scale-in: wait 30 minutes after queue empties. Each agent is ephemeral — fresh VM with no state carried over. Pre-warmed instances (always keep 2 running) for sudden demand. Kubernetes-based agents using Azure Pipelines Agent provider offer even faster start times (seconds vs minutes).

- **Q: How do you handle database migrations in CI/CD pipelines?**
  - **A:** Database migrations run as a pipeline stage before application deployment. Migration script included in repository and version-controlled alongside code changes. Using Flyway or EF Core migrations, the pipeline connects to the target database using connection strings from Key Vault-linked variable groups. Migrations must be backward-compatible: new columns allow NULL initially, renamed columns follow expand-contract pattern. Pipeline includes rollback scripts with automated rollback if post-deployment health checks fail. Migrations validated against a staging copy of production data before being applied.

- **Q: Explain how Azure DevOps integrates with Azure Key Vault.**
  - **A:** Variable Groups linked to Azure Key Vault map each secret to a pipeline variable. At runtime, the Azure DevOps agent authenticates to Key Vault using managed identity (self-hosted) or service connection (Microsoft-hosted), fetches secrets, and maps them to pipeline variables. Secrets are automatically masked in logs (`***`). When a secret is rotated in Key Vault, the next pipeline run automatically gets the new value. The 1-2 second startup delay is negligible. Use Managed Identity authentication for the service connection for the most secure setup.

- **Q: Design a container CI/CD with Azure DevOps and AKS.**
  - **A:** CI stage: run unit tests, build Docker image using multi-stage Dockerfile, scan for vulnerabilities with Trivy, sign image with Cosign. Image tagged with build ID and commit SHA and pushed to ACR. CD stage: use Kubernetes manifest task or Helm to deploy to AKS referencing the new image tag. For blue-green in AKS, create new deployment in green service, validate, then update service selector. ACR geo-replication for fast image pulls. Container scanning catches vulnerabilities before deployment. AKS uses Azure AD integration for RBAC.

- **Q: What are YAML templates and how do you use them?**
  - **A:** YAML templates are reusable YAML files defining common pipeline patterns referenced from multiple pipelines. Three types: job templates (reusable job definitions), stage templates (reusable deployment patterns), and variable templates (shared variable definitions). Templates support parameters with types, default values, and validation. Stored in a central Git repository referenced using `template:` syntax. Can be extended with `extends` keyword for enforced pipeline structure. The DevOps team maintains templates ensuring mandatory security and compliance steps.
  - **Interview follow-up:** How do you version YAML templates so that a breaking change in a template doesn't break all pipelines simultaneously?

- **Q: How do you implement artifact promotion with security gates?**
  - **A:** The same immutable build artifact is promoted through increasingly validated environments, never rebuilt. Dev: CI gates (unit tests pass, code coverage meets threshold, static analysis passes). Staging: security gates (vulnerability scan finds no critical/high CVEs, compliance scan passes, dynamic scan finds no exploitable vulnerabilities). Pre-Prod: manual gates (QA lead approves, security team reviews). Production: deployment requires change advisory board approval and automated health gates (error rate < 0.1%, p99 latency < 500ms, CPU < 80%). Artifact stored in Azure Artifacts or ACR with immutable tags.

---

## Interview Questions

- **What is the difference between a Variable Group and a Variable in Azure DevOps?**
  - **A:** Variables are simple key-value pairs defined per pipeline, scoped to a single pipeline. Variable Groups are shared across multiple pipelines, managed centrally in the Library hub, and can be linked to Azure Key Vault for secure secret management with RBAC support. Use regular variables for pipeline-specific values; use variable groups for shared configuration and secrets.

- **What is a YAML template and why use it?**
  - **A:** A YAML template is a reusable YAML file defining jobs, stages, or steps for use in multiple pipelines. Templates enable DRY pipeline definitions, enforce governance, and simplify multi-team adoption. Pass parameters to customize behavior. Store templates in a central repository with semantic versioning tags.

- **What is the difference between a Microsoft-hosted and self-hosted agent?**
  - **A:** Microsoft-hosted agents are managed by Azure — zero maintenance, common tools pre-installed, automatic scaling, limited to 60 minutes per job, no VNet access, costs per minute. Self-hosted agents run on your infrastructure — full control, unlimited execution time, cost-effective at scale, require maintenance. Choose Microsoft-hosted for simplicity; self-hosted for custom requirements or cost optimization at scale.

- **What is a deployment group?**
  - **A:** A logical set of target machines for deployment used in classic release pipelines. Each machine has an agent that runs deployment tasks. Supports rolling deployments with health checks. Being replaced by YAML environments with VM resources providing better integration with multi-stage YAML pipelines.

- **How do you implement canary deployments in Azure DevOps?**
  - **A:** In a YAML pipeline, set the deployment strategy to `canary` with traffic increments (e.g., 10%, 50%, 100%). Deploy to a subset of instances, run validation, then gradually increase traffic. Automatic rollback if health checks fail at any increment. Requires a load balancer or service mesh (App Service slots or AKS traffic splitting).

- **What is a service connection and how do you secure it?**
  - **A:** A service connection stores credentials for external services (Azure subscriptions, GitHub, Docker Hub) in Azure DevOps. Secure by using Managed Identity where possible, limit permission scope, use Workload Identity Federation for automatic credential rotation, and restrict service connection permissions so only authorized pipelines can use sensitive connections.

- **What is Azure Artifacts and how does it support CI/CD?**
  - **A:** A package management service for NuGet, npm, Maven, Python, and Universal Packages. Supports upstream sources proxying public feeds. Integrates with pipelines for automated publish and consume. Enables immutable, versioned artifacts for the "Build Once, Deploy Many" pattern.

- **What is the difference between Continuous Delivery and Continuous Deployment in Azure DevOps?**
  - **A:** Continuous Delivery: every commit passing CI is deployed to staging/pre-production, but production deployment requires manual approval. Continuous Deployment: fully automated — every commit passing all stages goes to production without human intervention. Choose Continuous Delivery for risk-sensitive applications; Continuous Deployment for high test confidence and fastest time-to-production.

- **What is a pipeline decorator?**
  - **A:** A YAML template automatically injected into every pipeline in the organization by an admin. Defined in the `.azdevops` folder of a designated repository. Can add mandatory steps before/after every job or validate pipeline configuration at runtime. Ensures organization-wide compliance without individual teams adding steps themselves. Use cases: inject license compliance checks, enforce anti-tampering verification, add centralized logging.

- **What is the difference between `trigger` and `pr` triggers in Azure Pipelines?**
  - **A:** `trigger` defines which branches trigger a CI build when code is pushed. `pr` defines which branches trigger a PR validation pipeline when a pull request targets that branch. Use CI triggers for main/develop branches. Use PR triggers to validate pull requests before merge. Configure path filters in both to limit scope to relevant files.

---

## Developer Recommendations

- **Always use YAML pipelines over Classic pipelines** — Classic pipelines are not versionable, not reviewable, and cannot be stored in Git. YAML pipelines live in the repository alongside code, enabling peer review through pull requests and a complete audit trail. The migration is a one-time effort that pays dividends in traceability. Create a template library, convert simplest pipelines first, then progressively migrate complex multi-stage pipelines. Pipeline changes are reviewed, tested, and deployed through the same process as code changes.
  - **Production story:** A team ran a classic release pipeline for two years with a missing "run DB migration" step — every deployment silently skipped schema changes, and production outages were blamed on "random" failures until someone traced the deployment logs back to the missing step that no one could see in the UI.

- **Link variable groups to Key Vault for all secrets** — Hardcoded secrets in pipeline YAML or library variables are a security breach waiting to happen. Key Vault-linked variable groups fetch secrets at runtime, mask them in logs, and support automatic rotation. Authentication should use Managed Identity or Workload Identity Federation to eliminate static credentials. The 1-2 second startup delay is negligible compared to the security benefit.

- **Use environment-specific service connections** — A single service connection across environments is risky — a dev deployment mistake could affect production. Create separate service connections per environment with permissions scoped to only that environment's resources. Use Azure AD Conditional Access policies to restrict connections to environment-specific IP ranges and require MFA for production. Add manual approval gates on production service connections.

- **Adopt templates early and enforce via Pipeline Decorators** — Without templates, every team writes their own pipeline logic leading to inconsistent quality and missing security steps. Create a central template repository with standardized build, test, and deploy templates parameterized for customization. Use Pipeline Decorators to inject mandatory steps (Trivy scanning, artifact publishing, SBOM generation) without teams adding them explicitly. Version templates with semantic tags.

- **Cache dependencies aggressively** — npm node_modules, Maven .m2 directories, Docker layers, and NuGet packages rebuild on every pipeline run without caching. Use the Cache task with a restore key based on the lock file hash. For Docker builds, use `--cache-from` pointing to the previously built image in ACR. Pipeline caching can reduce build times by 60-80%, providing faster feedback and reducing CI infrastructure costs.
  - **Production story:** A 30-person team was spending $4,000/month on Azure DevOps hosted agents because every pipeline re-downloaded node_modules until caching was implemented — the bill dropped to $1,200 after adding cache-as-key-based restore.

- **Implement artifact promotion with immutability** — Rebuilding artifacts for each environment guarantees inconsistency. Build once in CI, then promote the exact same artifact through Dev → Staging → Pre-Prod → Production. Each environment validates but never rebuilds. The artifact is identified by its version (semantic version + commit SHA) and stored in Azure Artifacts or ACR with immutable tags. Environment-specific configuration is injected at deployment time through environment variables. Use retention policies to clean old artifacts but keep production versions indefinitely for audit traceability.
