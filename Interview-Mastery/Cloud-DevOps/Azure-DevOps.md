# Azure DevOps

---

## Overview

**Definition:** Azure DevOps is Microsoft's end-to-end DevOps platform providing integrated services for planning, developing, delivering, and maintaining applications. The platform unifies the entire software delivery lifecycle into a single toolchain, including Azure Boards for work tracking, Azure Repos for Git repositories, Azure Pipelines for CI/CD, Azure Test Plans for testing, and Azure Artifacts for package management. This integration eliminates the friction of stitching together multiple third-party tools and provides consistent authentication, permissions, and reporting across all DevOps activities. Azure DevOps is available as a cloud service (Azure DevOps Services) with Microsoft-managed infrastructure and as an on-premise server (Azure DevOps Server) for organizations with data residency or air-gapped requirements.

**Why It Exists:** Before Azure DevOps, Microsoft had separate tools (Team Foundation Server for version control and work tracking, Jenkins for CI/CD, NuGet for packages) that required complex integrations and custom scripting to work together. Azure DevOps unified these capabilities into a single platform with consistent authentication through Azure AD, unified permissions, and cross-service traceability — from a work item to the code commit that implements it to the build that compiled it to the release that deployed it. The platform's deep integration with Azure (deploying to App Service, AKS, Azure Functions, and VMs) and extensibility through the marketplace (1,000+ extensions for Slack, Jira, SonarQube, Docker, Kubernetes, and more) make it particularly attractive for Microsoft-centric organizations. Azure DevOps supports both cloud-hosted and self-hosted agent pools, enabling organizations to maintain control over their build infrastructure while benefiting from Microsoft-managed services for orchestration.

**Key Concepts:** The **Organization** is the top-level container that holds all projects, billing, and policies, typically representing the entire company. Each **Project** is a logical container for code repositories, work items, pipelines, and test plans, usually corresponding to a product or team. **Teams** are subsets of a project with their own area paths, backlogs, and dashboards, enabling teams within the same project to work independently with their own views and permissions. **Agent Pools** are collections of build and release agents that execute pipeline jobs, which can be Microsoft-hosted (zero maintenance, limited to 60 minutes per job) or self-hosted (custom hardware with network access to on-premise resources). **Variable Groups** store variables shared across multiple pipelines, with integration to Azure Key Vault for secure secret management, ensuring that connection strings, API keys, and passwords are never stored in pipeline YAML files. **Service Connections** provide secure, auditable access to external services like Azure subscriptions, GitHub repositories, Docker registries, and Jenkins servers, with permissions scoped to specific pipelines and environments. **Environments** represent deployment targets (development, staging, production) with approval gates, resource tracking, and deployment history, enabling governance workflows where each environment can require different approval levels before deployment proceeds.

---

## Core Services

**Azure Boards** provides work tracking with Kanban boards, backlog management, sprint planning, and customizable dashboards, supporting Scrum, Agile, and CMMI process templates. Work items can be hierarchically organized as Epics, Features, User Stories, and Tasks, with each type having customizable fields, states, and transitions that match the team's workflow. The boards integrate deeply with Azure Repos through automatic work item linking: when a developer includes "AB#1234" in a commit message, the commit is automatically linked to work item 1234, and the work item shows the associated code, build status, and deployment information. This traceability enables answering questions like "What work items are in this release?" and "Which commits fixed this bug?" without manual cross-referencing. Azure Boards also supports GitHub integration, allowing teams using GitHub for code to still use Azure Boards for work tracking with the same automatic linking capabilities through the GitHub + Azure Boards integration app.

**Azure Repos** provides Git repositories with branch policies, pull request workflows, and code reviews. Branch policies enforce quality gates before code can be merged: minimum number of reviewers, check for linked work items, require successful build validation, and require comment resolution. The policy system ensures that every merge to main has been reviewed, built successfully, and linked to an active work item, providing a complete audit trail for compliance requirements. Azure Repos supports Team Foundation Version Control (TFVC) for legacy systems but strongly recommends Git for all new projects. The repository integrates with Azure Pipelines through PR triggers, automatically building each pull request and reporting the build status directly in the PR view, preventing merges that would break the build.

**Azure Pipelines** is the CI/CD engine that supports multi-stage YAML pipelines with container support, matrix builds, and deployment strategies including canary, blue-green, and rolling deployments. The YAML pipeline definition lives in the repository alongside the code, enabling version control, code review, and branching of pipeline changes. Pipelines can build and deploy to any platform: Windows, Linux, and macOS agents support .NET, Java, Node.js, Python, Go, and C++ applications. Deployment jobs integrate with Environments, providing approval gates, resource-specific deployment history, and traceability from pipeline run to deployed resources. The pipeline caching feature significantly reduces build times by caching dependencies like npm packages, Maven artifacts, and NuGet packages across runs.

**Azure Test Plans** provides manual and exploratory testing capabilities with test case management and rich execution reporting. Test plans can be created from requirements and user stories, providing end-to-end traceability from customer requirements through test cases to test results. The exploratory testing extension allows testers to capture screenshots, screen recordings, and annotated notes during ad-hoc testing, automatically creating bug work items with reproduction steps. Test execution analytics provide insights into test pass rates, flaky tests, and test coverage gaps, helping teams identify quality trends over time.

**Azure Artifacts** provides package management for NuGet, npm, Maven, Python, and Universal Packages, with upstream source proxying that caches public packages while giving you control over which versions enter your environment. The service integrates with Azure Pipelines for automated package publishing and consumption, enabling the "Build Once, Deploy Many" pattern where each CI run produces versioned packages that are promoted through environments. Upstream sources enable teams to use a single feed for both internal and external packages, blocking vulnerable or deprecated versions before they reach developers. Retention policies automatically clean up old package versions, reducing storage costs while maintaining the latest versions needed for production traceability.

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

**Using classic build and release pipelines instead of YAML pipelines** is the most common mistake in Azure DevOps adoption. Classic pipelines are configured through the web UI with drag-and-drop editors, meaning the configuration is stored in Azure DevOps, not in the repository. This prevents version control, code review, and branching of pipeline changes. When a classic pipeline breaks, you cannot roll back to a known-good configuration from Git — you must manually recreate the configuration. Migrating to YAML pipelines requires upfront effort but pays dividends in traceability, auditability, and the ability to review pipeline changes in pull requests alongside code changes. Every new pipeline should be YAML from the start, and existing classic pipelines should have a migration plan with a sunset date. This *looks correct* because the visual editor is intuitive, requires no YAML knowledge, and gives immediate feedback through the drag-and-drop interface — making it feel more accessible and productive.

**Not using variable groups linked to Azure Key Vault** leads to secret sprawl where connection strings, API keys, and passwords are hardcoded in pipeline YAML files or stored as plain-text library variables. Variable groups linked to Key Vault fetch secrets at runtime, automatically masking them in pipeline logs. When a secret is rotated in Key Vault, all pipelines using that variable group immediately get the new value without any code changes. Without this integration, secret rotation becomes a manual, error-prone process that often results in outages when expired secrets are missed. The trade-off is a 1-2 second delay at pipeline startup while secrets are fetched from Key Vault, but this is negligible compared to the security benefits. This *looks correct* because storing secrets as plain-text library variables works and is simple — the security implications of having secrets visible in pipeline configuration aren't immediately obvious.

**Hardcoding subscription IDs in pipeline YAML** creates fragility because the YAML becomes environment-specific and cannot be reused across subscriptions. If a subscription ID changes (e.g., when migrating between development and production environments), every pipeline file referencing that ID must be updated. Service connections abstract this by storing the subscription reference securely, allowing pipelines to reference the connection by name and swap it per environment. Service connections also support managed identities and workload identity federation, eliminating the need for service principal secrets entirely. This *looks correct* because the subscription ID is a non-secret identifier — pasting it directly into the YAML seems harmless and avoids the overhead of setting up a service connection.

**Deploying without staging slots** causes downtime during application deployments because the app must be stopped, updated, and restarted. Azure App Service deployment slots enable zero-downtime deployments by deploying to a staging slot, running smoke tests to verify the deployment, and then swapping the staging and production slots with zero downtime. If the deployment has issues, swapping back restores the previous version instantly. Without deployment slots, even simple deployments require maintenance windows and cause user-facing downtime. This *looks correct* because deploying directly to production is the simplest, most straightforward approach — the 30 seconds of downtime during restart seems acceptable until you get paged for every deployment.

**Not using path filtering in pipeline triggers** causes unnecessary pipeline runs on documentation changes, README updates, and configuration changes that don't affect the application code. In a monorepo with multiple services, every change to any service triggers CI for all services, wasting compute resources and delaying feedback for the actual change. Path filters (`paths: { include: [src/app/**] }`) restrict pipeline execution to files that actually matter, reducing CI costs by 40-60% in monorepo setups. This *looks correct* because running the full pipeline on every push feels thorough and safe — filtering seems like an optimization you don't need until the CI queue grows to hours.

**Skipping environment approvals for production deployments** enables accidental deployments to production, which can cause outages, data loss, and compliance violations. Environment protection rules in Azure DevOps enforce mandatory approvers, wait timers, and deployment branch restrictions, preventing unauthorized or untested code from reaching production. The approval process creates an audit trail showing who approved each deployment and when, which is required for SOC2, HIPAA, and PCI compliance. This *looks correct* because approvals slow down the pipeline and seem like unnecessary bureaucracy when you trust your team and your automated tests.

**Not setting artifact retention policies** causes storage costs to grow unboundedly as every build artifact is retained forever. Old artifacts consume expensive storage space and make it harder to find the relevant artifacts for production traceability. Retention policies should be configured at the pipeline level to automatically delete old artifacts (e.g., keep only the last 30 days of development builds, but keep production builds for 1 year for audit purposes). This *looks correct* because artifacts feel valuable — deleting them seems risky, and storage is cheap, so keeping everything indefinitely seems like a safe default.

**Not using YAML templates** leads to duplicated pipeline code across teams and projects. Without templates, each team copies and pastes the same build, test, and deploy steps, creating maintenance nightmares when a shared step needs updating (e.g., a new security scanning tool). YAML templates define reusable stages, jobs, and steps in a central repository, and teams reference them with parameters for customization. The central DevOps team controls the template content, ensuring consistent quality and security across all pipelines. This *looks correct* because copying an existing pipeline YAML that works is faster than learning the template syntax — the duplication cost only becomes visible when you need to update 50 pipelines with a security fix.

**Using a single agent pool for all workloads** causes resource contention where long-running builds block quick CI feedback loops. A database migration pipeline that runs for 30 minutes occupies agent capacity that could be used by 30 one-minute unit test runs. Separate agent pools should be configured by workload type: a dedicated pool for quick CI builds, a separate pool for long-running integration tests, and another for deployment jobs that need access to production network resources. This *looks correct* because one pool is simpler to manage and all builds share the same capacity, which seems efficient — the contention only becomes visible as developers wait longer for CI feedback on simple commits.

**Not implementing pipeline caching** forces every build to re-download the same dependencies from the internet, wasting time and bandwidth. npm packages, Maven artifacts, NuGet packages, and Docker layers rarely change but are re-fetched on every pipeline run. Pipeline caching stores these dependencies between runs using a cache key based on the lock file hash. When the lock file hasn't changed, the cache is restored in seconds instead of minutes, reducing build times by 60-80%. This *looks correct* because downloading dependencies is just part of the build process, and it works — the wasted time is only apparent when you measure it against a cached run and see the difference.

---

## Key Design Considerations

**YAML over Classic pipelines** is the foundational decision for any Azure DevOps implementation. YAML pipelines live in the repository alongside code, benefiting from version control, code review through pull requests, and branching for pipeline changes. Classic pipelines are configured through the web UI and stored in Azure DevOps, making them unreviewable, unversioned, and prone to configuration drift. YAML also enables reusability through templates, where common build, test, and deploy patterns are defined once and referenced by all teams. Pipeline templates can be parameterized (passing environment names, service connections, and deployment strategies) and stored in a central Git repository that only the DevOps team can modify, enforcing governance without blocking developer velocity. The migration from classic to YAML should be treated as a technical debt reduction initiative, with a phased approach: convert the most complex pipelines first to validate the template patterns, then train teams and sunset classic pipelines.

**Agent strategy** should balance cost, control, and maintenance overhead. Microsoft-hosted agents are the simplest option — Microsoft manages the operating system, tools, and patches, and agents auto-scale based on demand. They include common tools pre-installed (Node.js, Python, .NET, Docker, Azure CLI) and provide clean environments for every job. The limitations include a 60-minute job timeout, 10 GB of storage, and no access to on-premise networks or self-hosted resources. Self-hosted agents provide full control: custom hardware specifications, pre-installed enterprise tools (Sybase, Oracle clients), network access to on-premise databases and servers, and no time limits on job execution. The trade-off is operational overhead — agents must be patched, monitored, and scaled. For organizations with more than 50 active developers, self-hosted agents using VM Scale Sets provide the best balance: auto-scaling based on pipeline queue depth, custom VM images with pre-cached tools, and scale-to-zero when idle to minimize costs.

**Secrets management** should follow the principle that no secret should ever appear in pipeline YAML, repository code, or build logs. Variable groups linked to Azure Key Vault provide the most secure approach: secrets are stored in Key Vault with access policies, audit logging, and automatic rotation, and pipelines fetch them at runtime through the Azure DevOps-agent Key Vault integration. Service connections should use Managed Identity or Workload Identity Federation where possible — managed identities eliminate the need for service principal secrets entirely because the identity is tied to the Azure resource and managed by Azure AD. Secure files (stored in Azure DevOps Library) handle certificates and SSH keys that aren't simple strings. The trade-off of centralized secrets management is a small startup delay for secret retrieval, but this is vastly outweighed by the elimination of secret sprawl, leaked credentials, and manual rotation processes.

**Deployment strategies** must be chosen based on application architecture, risk tolerance, and rollback speed requirements. `runOnce` is the simplest strategy — deploy to the target, verify health — suitable for development environments where speed matters more than risk mitigation. `canary` progressively shifts traffic: deploy to 10% of instances, monitor for errors, then ramp to 50%, then 100%. This catches issues with minimal blast radius and enables automatic rollback if health checks fail at any stage. `rolling` replaces instances in batches (e.g., 25% at a time), maintaining capacity throughout the deployment. `blueGreen` maintains two full environments: deploy to the inactive environment, validate, then swap traffic. Rollback is instant (swap back), but resource costs double during deployment. For Azure App Service, deployment slots provide built-in blue-green deployment with slot swap, warm-up, and auto-swap capabilities.

**Governance in Azure DevOps** should enforce quality and compliance without blocking developer velocity. Branch policies protect main and release branches: require pull request reviews (minimum 1-2 reviewers), require linked work items (traceability), require successful build (no broken builds), and require comment resolution. Environment approval gates ensure that deployments to sensitive environments require authorized approvals, with options for minimum number of approvers, team-based approvals, and timeouts. YAML templates enforced by the DevOps team ensure all pipelines include mandatory steps (security scanning, artifact publishing, SBOM generation) without individual teams having to remember them. Audit logging records all pipeline runs, approvals, and configuration changes for compliance reporting.

**Enterprise setup** for Azure DevOps should follow a single-organization model with multiple projects organized by business domain. The single organization enables cross-project visibility, shared agent pools, and centralized policies, while projects provide isolation for different products or teams. Managed identities for service connections eliminate the need for service principal secret management across hundreds of pipelines. Self-hosted agents with auto-scaling provide consistent performance and network access to on-premise resources. Standardized YAML templates and variable groups should be maintained by a central platform engineering team, with a clear contribution process for teams to request new template capabilities.

---

## Real-World Scenarios

**Scenario 1: Enterprise Migration from Classic to YAML Pipelines**

A large enterprise has 500+ classic build and release pipelines that were created over several years by different teams. These classic pipelines are unversioned, fragile, and difficult to maintain because any change requires navigating the web UI and remembering which settings were configured where. The classic releases cannot be code-reviewed, meaning configuration errors go undetected until they cause build or deployment failures.

The migration strategy begins with creating a YAML template library in a central Git repository with reusable job templates for build, test, security scan, and deploy. These templates are parameterized with inputs for environment name, service connection, application path, and build configuration. Each team converts their pipelines incrementally, starting with the simplest build-only pipelines to validate the template patterns, then moving to multi-stage deployment pipelines. Pipeline decorators enforce organization-wide governance by injecting required steps (license scanning, container vulnerability scanning, SBOM generation) into every pipeline without teams having to add them explicitly. Variable groups linked to Key Vault replace hardcoded secrets, and service connections replace hardcoded subscription IDs. A deprecation dashboard tracks migration progress, and classic pipelines are forcibly retired after 6 months by disabling the "Create classic pipeline" permission at the organization level.

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

**Scenario 2: Multi-Environment Release with Compliance Gates**

A FinTech company requires multi-person approval, security scanning, and compliance verification before any code reaches production. Their CI/CD pipeline must enforce that every release passes through increasing levels of validation, with clear approval gates and audit trails at each stage.

The pipeline is structured as a multi-stage YAML pipeline promoting a single build artifact through each environment. In Dev, every commit triggers automatic deployment and CI validation including unit tests, integration tests, and code coverage. In Staging, a comprehensive security scan runs including Trivy for container vulnerabilities, OWASP ZAP for dynamic application security testing, and dependency check for known CVEs in open-source libraries. Integration tests run against the staging database with realistic test data. Pre-Prod adds manual approval gates: the QA lead must approve after verifying test results, and the compliance officer must approve after reviewing the security scan report. An Azure Monitor query gate automatically validates that error rates are below 0.1% before allowing progression. Production requires change advisory board approval through Azure DevOps environment approvals, followed by a canary deployment starting at 10% traffic. If error rates or latency exceed thresholds during the canary, the pipeline automatically rolls back. The same build artifact (immutable container image with SHA256 digest) is promoted through every environment, ensuring that what was tested in Staging is exactly what runs in Production.

**Scenario 3: Monorepo CI/CD with 80+ Microservices**

A product team manages 80 microservices in a single Git repository, each in its own folder under `services/`. Without optimization, every commit triggers CI for all 80 services, taking hours and wasting compute on unchanged services.

The solution uses path triggers to only build services that have changed: each service's folder path is listed in the trigger's include pattern. A YAML template matrix dynamically generates jobs based on which paths changed, using a file-change detection script that outputs JSON to a pipeline variable. The fan-out pattern launches parallel build jobs for each changed service, and the pipeline caches npm packages and Docker layers using cache keys based on lock file hashes. After the build, a fan-in pattern consolidates deployment: only the changed services' artifacts are deployed, and only to environments relevant to the commit's branch (feature branches deploy to dev, main deploys to staging and production). This approach reduces average CI time from 4 hours to 5 minutes for a typical commit that changes 1-2 services.

---

## Scenario-Based Questions

**1. Q: Design a PCI-compliant CI/CD pipeline using Azure DevOps.**

A: A PCI-compliant pipeline requires multiple layers of security controls throughout the software delivery lifecycle. Self-hosted agents running in a restricted network segment provide isolation from the public internet and ensure build artifacts never traverse unsecured networks. All sensitive configuration values (database connection strings, encryption keys, API tokens) are stored in Azure Key Vault and referenced through variable groups, with Key Vault access policies restricted to only the specific secrets each pipeline needs. Environment approvals enforce a minimum of two approvers for production deployments, with approval history stored immutably for audit purposes. YAML templates from a central repository ensure every pipeline includes mandatory compliance steps: static code analysis, dependency vulnerability scanning, container image scanning with Trivy, and software bill of materials generation. Artifact retention policies keep production build artifacts for at least one year for audit traceability. Audit logging is enabled at the organization level, recording all pipeline runs, variable group modifications, and approval decisions. Signed commits enforced through branch policies ensure every code change is traceable to a specific developer with a verified identity. The pipeline runs as a single immutable build artifact that is promoted through environments — the artifact is built once in CI and promoted through Dev, QA, Staging, and Production without ever being rebuilt, ensuring what's tested is what's deployed.

**2. Q: How do you implement blue-green deployment with App Service slots?**

A: Azure App Service deployment slots provide built-in blue-green deployment capability without requiring infrastructure orchestration. The production slot runs the current application version; the staging slot is the "green" environment where the new version is deployed. The YAML pipeline deploys the new build to the staging slot, runs smoke tests against the staging slot's URL to verify the deployment succeeded, then swaps the staging and production slots. The swap operation is instant because it's just a routing rule change at the Azure Front Door level — the slots' content is not moved. After the swap, the old application version resides in the staging slot, enabling instant rollback by swapping again if issues are discovered. Slot-specific settings (connection strings, app configurations, environment variables) remain with their respective slots through the swap — staging continues using the staging database and production uses the production database even after the swap, because these settings are "sticky" to slots. Auto-swap can be configured to automatically perform the swap after a successful staging deployment and warm-up period, enabling fully automated blue-green deployments.

> **Interview follow-up:** What happens during a slot swap when the staging slot's application needs to connect to the production database for warm-up — how do you prevent it from writing to the production database during warm-up?

**3. Q: Design a multi-region deployment pipeline with release gates.**

A: The pipeline divides deployment into regional stages that execute sequentially with health validation gates between each stage. The first region (primary) receives the deployment first: deploy to Region 1, run integration tests against the region's endpoints, and verify health metrics through Azure Monitor query gates that check error rates, latency percentiles, and resource utilization. After Region 1 is validated, an approval gate may be required before proceeding (depending on compliance requirements). The pipeline then deploys to Region 2 using the same process, followed by Region 3, and so on. Each region has its own environment in Azure DevOps with scoped service connections and approval history. Azure Front Door's global load balancer routes traffic to only healthy regions, so a deployment failure in one region doesn't affect overall availability. The pipeline uses a ring-based deployment pattern: Ring 0 (canary, single region with 5% traffic), Ring 1 (3 additional regions, 25% traffic), Ring 2 (all regions, 100% traffic). Each ring has automated rollback if health checks fail at any point.

**4. Q: How would you manage Terraform infrastructure as code in Azure DevOps?**

A: Terraform IaC in Azure DevOps follows a validate-plan-apply workflow with environment isolation and state management. The pipeline has a validate stage that runs `terraform init`, `terraform validate`, and `terraform fmt` to catch syntax errors and formatting issues before any infrastructure changes. The plan stage runs `terraform plan` and publishes the plan output as a build artifact for review. For non-production environments, the apply stage runs automatically after a manual approval confirms the plan looks correct. For production environments, the approval requires multiple approvers based on environment protection rules. Terraform state is stored in Azure Storage with a separate container per environment and state locking enabled through Azure Storage's blob lease capabilities. Each environment uses a separate backend configuration, ensuring changes to dev infrastructure never affect production. The pipeline uses Terraform workspaces for additional isolation within environments where needed. A destroy stage can be optionally triggered for ephemeral environments (feature branch deployments) to clean up resources when the branch is deleted.

> **Interview follow-up:** Two developers run Terraform plan simultaneously, both see the same state, and both approve. What happens when the second apply runs?

**5. Q: Design a self-hosted agent auto-scaling solution.**

A: Azure VM Scale Set agents provide the most scalable self-hosted agent solution for Azure DevOps. The VM Scale Set is configured with a custom VM image that includes all required tools (SDKs, compilers, testing frameworks, deployment tools) pre-installed and cached. The scale set auto-scales based on the Azure DevOps pipeline queue depth: when jobs are queued, new VM instances are provisioned to handle them; when the queue is empty, instances scale down to zero. The scale-out policy adds instances aggressively (e.g., 2 instances per queued job up to a maximum of 20) to minimize wait times, while the scale-in policy waits 30 minutes after the queue empties before removing instances to handle burst traffic. Each agent is ephemeral — jobs run on a fresh VM with no state carried over between runs, eliminating the problem of agent drift where installed software accumulates over time. For even faster provisioning, pre-warmed instances can be maintained (e.g., always keep 2 instances running) to absorb sudden build demand. Kubernetes-based agents using the Azure Pipelines Agent provider offer an alternative: pods are created per job with container images containing the build tools, providing even faster start times (seconds vs minutes for VMs).

**6. Q: How do you handle database migrations in CI/CD pipelines?**

A: Database migrations in Azure DevOps pipelines should run as a pipeline stage before the application deployment, with the migration script included in the repository and version-controlled alongside code changes. Using Flyway or Entity Framework Core migrations, the pipeline stage connects to the target database using connection strings from a Key Vault-linked variable group, applies pending migrations, and validates the migration result. The migration must be backward-compatible: new columns should allow NULL values initially, renamed columns should follow the expand-contract pattern (add new column, migrate data, remove old column across multiple deployments), and dropped columns should be preceded by a deprecation phase. The pipeline should include rollback scripts for each migration, and the rollback should be automated: if the post-deployment health checks fail, the pipeline automatically applies the rollback migration and reverts the application deployment. For production databases, migrations should be validated against a staging copy of production data before being applied. The migration script is only run once per database — the migration tool tracks which migrations have been applied and only runs new ones.

**7. Q: Explain how Azure DevOps integrates with Azure Key Vault.**

A: The integration works through Variable Groups that are linked to Azure Key Vault. When a Variable Group is linked to a Key Vault, each secret in Key Vault is mapped to a pipeline variable with the same name. At pipeline runtime, the Azure DevOps agent authenticates to Key Vault using its managed identity (for self-hosted agents) or the pipeline's service connection (for Microsoft-hosted agents), fetches the secrets, and maps them to pipeline variables. The secrets are automatically masked in Azure DevOps logs — any output containing the secret value is replaced with `***`. When a secret is rotated in Key Vault (new version created), the next pipeline run automatically gets the new value because the integration fetches the latest version by default. This eliminates the need to update pipeline configurations when secrets change. The trade-off is a 1-2 second delay at pipeline startup as secrets are fetched, but this is negligible. For the most secure setup, use Managed Identity authentication for the service connection between Azure DevOps and Key Vault, eliminating the need for any service principal secrets in the configuration.

**8. Q: Design a container CI/CD with Azure DevOps and AKS.**

A: The pipeline builds a Docker image, pushes it to Azure Container Registry, and deploys it to Azure Kubernetes Service. The CI stage runs unit tests, builds the Docker image using a multi-stage Dockerfile for small image size, scans the image for vulnerabilities with Trivy or Microsoft Defender for Containers, and signs the image with Cosign for supply chain security. The image is tagged with the build ID and commit SHA and pushed to ACR. The CD stage uses the Kubernetes manifest task or Helm to deploy to AKS, referencing the new image tag. For blue-green deployment in AKS, the pipeline creates a new deployment in the green service, validates it with smoke tests, then updates the service selector to point to the green deployment. ACR geo-replication ensures fast image pulls in multiple regions. Container scanning in CI catches vulnerabilities before deployment — if a critical vulnerability is found, the pipeline fails and the image is not deployed. The AKS cluster uses Azure AD integration for RBAC, and the pipeline authenticates to AKS using a service connection with Kubernetes RBAC permissions scoped to the target namespace.

**9. Q: What are YAML templates and how do you use them?**

A: YAML templates are reusable YAML files that define common pipeline patterns and can be referenced from multiple pipelines. There are three types: job templates (reusable job definitions with steps), stage templates (reusable stage patterns for deployment), and variable templates (shared variable definitions). Templates support parameters with types, default values, and validation rules, enabling customization without duplication. For example, a deploy job template accepts parameters for environment name, service connection, and deployment strategy, and the template contains all the standard steps (deploy, health check, smoke test, rollback). Templates are stored in a central Git repository that teams reference using `template:` syntax. The DevOps team maintains the templates and ensures they include mandatory security and compliance steps. Templates can be extended with `extends` keyword, where the template defines the pipeline structure and teams provide specific values through parameters. This enforces consistent pipeline patterns across the organization while allowing teams to customize what they need.

> **Interview follow-up:** How do you version YAML templates so that a breaking change in a template doesn't break all pipelines simultaneously?

**10. Q: How do you implement artifact promotion with security gates?**

A: Artifact promotion ensures the same immutable build artifact is promoted through increasingly validated environments, never rebuilt between environments. Each environment runs validation specific to its purpose and only allows promotion if all gates pass. In Dev, the artifact is deployed and CI gates run: unit tests pass, code coverage meets threshold, static analysis finds no critical issues. In Staging, security gates run: vulnerability scan finds no critical or high CVEs, container image passes compliance scan, dynamic security scan finds no exploitable vulnerabilities. In Pre-Prod, manual gates apply: QA lead approves after verifying integration test results, security team approves after reviewing scan report. In Production, deployment requires change advisory board approval and passes automated health gates (error rate < 0.1%, p99 latency < 500ms, CPU utilization < 80%). If any gate fails, the artifact is rejected and cannot progress to the next environment. The artifact is stored in Azure Artifacts or ACR with immutable tags, ensuring the exact artifact that passed all gates in Dev is the same one deployed to Production.

---

## Interview Questions

**1. What is the difference between a Variable Group and a Variable in Azure DevOps?**

A: Variables are simple key-value pairs defined per pipeline in the YAML file or pipeline settings UI. They are scoped to a single pipeline and must be duplicated across pipelines if needed. Variable Groups are shared across multiple pipelines, managed centrally in the Library hub, and can be linked to Azure Key Vault for secure secret management. Variable Groups support role-based access control, allowing you to control which pipelines can access which groups. Use regular variables for pipeline-specific values; use variable groups for shared configuration and secrets.

**2. What is a YAML template and why use it?**

A: A YAML template is a reusable YAML file that defines jobs, stages, or steps for use in multiple pipelines. Templates enable DRY (Don't Repeat Yourself) pipeline definitions, enforce governance (security teams control deploy templates), and simplify multi-team adoption. Pass parameters to customize behavior for different applications and environments. Store templates in a central repository with semantic versioning tags, and teams reference specific versions for stability.

**3. What is the difference between a Microsoft-hosted and self-hosted agent?**

A: Microsoft-hosted agents are managed by Azure — zero maintenance, common tools pre-installed, automatic scaling, but limited to 60 minutes per job, no VNet access, and costs per minute. Self-hosted agents run on your infrastructure — full control over operating system, tools, and network access, unlimited execution time, and cost-effective at scale, but require maintenance (patching, monitoring, capacity planning). Choose Microsoft-hosted for simplicity and self-hosted for custom requirements or cost optimization at scale.

**4. What is a deployment group?**

A: A logical set of target machines for deployment used in classic release pipelines. Each machine has an agent that runs deployment tasks. Supports rolling deployments with health checks. Deployment groups are being replaced by YAML environments with VM resources, which provide better integration with multi-stage YAML pipelines and environment-level approvals.

**5. How do you implement canary deployments in Azure DevOps?**

A: In a YAML pipeline, set the deployment strategy to `canary` with traffic increments (e.g., 10%, 50%, 100%). The canary strategy deploys to a subset of instances, runs validation, then gradually increases the traffic percentage. If health checks fail at any increment, the pipeline automatically rolls back. Canary deployments require a load balancer or service mesh (like Azure App Service deployment slots or AKS traffic splitting) to control traffic percentages.

**6. What is a service connection and how do you secure it?**

A: A service connection stores credentials for external services (Azure subscriptions, GitHub, Docker Hub, Jenkins) in Azure DevOps. Secure by using Managed Identity where possible (eliminates credential management), limit the scope of permissions the connection grants (specific resource group or service), use Workload Identity Federation for automatic credential rotation, and restrict service connection permissions in Project Settings so only authorized pipelines can use sensitive connections.

**7. What is Azure Artifacts and how does it support CI/CD?**

A: A package management service for NuGet, npm, Maven, Python, and Universal Packages. Supports upstream sources that proxy public feeds (caching packages locally for reliability and speed). Integrates with pipelines for automated publish (CI pushes packages) and consume (CD pulls packages). Enables immutable, versioned artifacts for the "Build Once, Deploy Many" pattern, where each CI run produces versioned packages that are promoted through environments without rebuilding.

**8. What is the difference between Continuous Delivery and Continuous Deployment in Azure DevOps?**

A: Continuous Delivery means every commit that passes CI is automatically deployed to staging or pre-production environments, but deployment to production requires manual approval. Continuous Deployment is fully automated — every commit that passes all stages (including tests, scans, and approval gates) goes to production without human intervention. Choose Continuous Delivery for risk-sensitive applications (finance, healthcare) where human oversight is required. Choose Continuous Deployment when you have high confidence in automated testing and need the fastest possible time-to-production.

**9. What is a pipeline decorator?**

A: A pipeline decorator is a YAML template automatically injected into every pipeline in the organization by an admin. Defined in the `.azdevops` folder of a designated repository, decorators can add mandatory steps before or after every job (e.g., run a security scan at the end of every job) or validate pipeline configuration at runtime. Decorators ensure organization-wide compliance without individual teams having to add these steps themselves. Use cases: inject license compliance checks, enforce anti-tampering verification, add centralized logging.

**10. What is the difference between `trigger` and `pr` triggers in Azure Pipelines?**

A: `trigger` defines which branches trigger a CI build when code is pushed to that branch. `pr` defines which branches trigger a PR validation pipeline when a pull request targets that branch. Use CI triggers for main and develop branches to build and test every merge. Use PR triggers to validate pull requests before they are merged, giving reviewers confidence that the proposed changes don't break the build. Configure path filters in both to limit scope to relevant files only.

---

## Developer Recommendations

**Always use YAML pipelines over Classic pipelines because classic pipelines are not versionable, not reviewable, and cannot be stored in Git.** YAML pipelines live in the repository alongside code, enabling peer review through pull requests, branching for pipeline changes, and a complete audit trail of who changed what and when. The migration from classic to YAML is a one-time effort that pays dividends in traceability and automation. The process involves creating a template library, converting the simplest pipelines first to validate patterns, then progressively migrating complex multi-stage pipelines. Without this migration, every pipeline change requires navigating the web UI, and configuration errors go undetected because they cannot be code-reviewed. The trade-off is the upfront migration effort (weeks for large organizations) but the long-term benefit of having pipeline changes reviewed, tested, and deployed through the same process as code changes is immense. A team ran a classic release pipeline for two years with a missing "run DB migration" step — every deployment silently skipped schema changes, and production outages were blamed on "random" failures until someone traced the deployment logs back to the missing step that no one could see in the UI.

**Link variable groups to Key Vault for all secrets because hardcoded secrets in pipeline YAML or library variables are a security breach waiting to happen.** Key Vault-linked variable groups fetch secrets at runtime, log them as masked, and support automatic rotation. When a secret is rotated in Key Vault, all pipelines using that variable group immediately get the new value without any code or configuration changes. Authentication to Key Vault should use Managed Identity for self-hosted agents or Workload Identity Federation for Microsoft-hosted agents, eliminating the need for any static credentials in the pipeline configuration. The trade-off is that pipeline startup takes 1-2 seconds longer for the secret fetch operation, but this is negligible compared to the security benefit of eliminating secrets from pipeline configurations and build logs.

**Use environment-specific service connections because using a single service connection across environments is risky — a dev deployment mistake could affect production.** Create separate service connections per environment with permissions scoped to only the resources in that environment. The development service connection has permissions to deploy to the development resource group only; the production service connection is restricted to production resources. Use Azure AD Conditional Access policies to restrict each connection to its environment's IP ranges and require multi-factor authentication for production connections. Add manual approval gates on production service connections through Azure DevOps environment protection rules. This defense-in-depth approach ensures that even if a CI/CD pipeline is compromised, only the compromised environment is affected.

**Adopt templates early and enforce via Pipeline Decorators because without templates, every team writes their own pipeline logic, leading to inconsistent quality, missing security steps, and maintenance nightmares.** Create a central template repository with standardized build, test, and deploy stage templates that are parameterized for customization. Use Pipeline Decorators to inject mandatory steps (security scanning with Trivy, artifact publishing to Azure Artifacts, SBOM generation) into every pipeline without teams having to add them explicitly. The templates should be versioned with semantic tags, and teams reference specific versions for stability. The centralized team maintains the templates and rolls out updates through the standard software lifecycle. The trade-off is that template changes require coordination across teams, but this is far better than having 500 different pipeline configurations that all need individual updates.

**Cache dependencies aggressively because npm node_modules, Maven .m2 directories, Docker layers, and NuGet packages rebuild on every pipeline run without caching.** Use the Cache task (`CacheBeta@2`) with a restore key based on the lock file hash (e.g., `package-lock.json`, `yarn.lock`, `pom.xml`). When the lock file hasn't changed, the cache is restored in seconds instead of minutes. For Docker builds, use Docker layer caching with `--cache-from` pointing to the previously built image in ACR, which allows Docker to reuse cached layers for unchanged dependencies. Pipeline caching can reduce build times by 60-80%, providing faster feedback to developers and reducing CI infrastructure costs. A 30-person team was spending $4,000/month on Azure DevOps hosted agents because every pipeline re-downloaded node_modules until caching was implemented — the bill dropped to $1,200 after adding cache-as-key-based restore.

**Implement artifact promotion with immutability because rebuilding artifacts for each environment guarantees inconsistency.** Build once in CI, then promote the exact same artifact through Dev → Staging → Pre-Prod → Production. Each environment validates the artifact but never rebuilds it. The artifact is identified by its version (semantic version + commit SHA) and stored in Azure Artifacts or ACR with immutable tags. Environment-specific configuration is injected at deployment time through environment variables, not baked into the artifact during the build. This guarantees that the artifact tested in staging is byte-for-byte identical to the one deployed to production. Use retention policies to clean old artifacts but keep the production version indefinitely for audit traceability.