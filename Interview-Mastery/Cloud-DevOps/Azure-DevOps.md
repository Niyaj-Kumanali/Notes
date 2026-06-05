# Azure DevOps

---

## Overview

- **Definition:** Azure DevOps is Microsoft's end-to-end DevOps platform providing integrated services for planning, developing, delivering, and maintaining applications.
- **Why It Exists:** It unifies the entire software delivery lifecycle — work tracking, Git repos, CI/CD, testing, and package management — with deep Azure integration, broad extensibility, and both cloud and self-hosted options.
- **Key Concepts:** **Organization** (top-level container with all resources), **Project** (logical container for code, work items, pipelines), **Team** (subset with area paths), **Agent Pool** (collection of build/release agents), **Variable Group** (shared variables across pipelines), **Service Connection** (secure external service access), **Environment** (deployment target with approvals and gates)

---

## Core Services

- **Azure Boards:** Work tracking with Kanban boards, backlogs, sprints, dashboards. Supports Scrum, Agile, CMMI. Customizable work item types and workflows.
- **Azure Repos:** Git repositories with branch policies, PRs, code reviews. Supports TFVC. Integrates with Boards for automatic work item linking.
- **Azure Pipelines:** CI/CD with multi-stage YAML pipelines. Container support, matrix builds, deployment strategies (canary, blue-green, rolling). Microsoft-hosted or self-hosted agents.
- **Azure Test Plans:** Manual and exploratory testing. Test case management with requirement traceability. Rich execution reporting.
- **Azure Artifacts:** Package management for NuGet, npm, Maven, Python. Proxies upstream feeds. Integrates with pipelines for automated publishing.

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

- **Using classic pipelines** — Not code-as-config; migrate to YAML pipelines
- **No variable groups** — Secret sprawl; centralize secrets in Library linked to Key Vault
- **Hardcoding subscription IDs** — Fragile; use service connections
- **No deployment slots** — Downtime on deploy; use staging slots with swap
- **No path filtering** — Slow CI in monorepo; use path triggers
- **No environment approvals** — Accidental deploys; add approval gates
- **No artifact retention policies** — Storage costs; set retention policies
- **No YAML templates** — Duplicate pipeline code; use templates for reuse
- **Single agent pool for all** — Resource contention; separate pools by workload
- **No pipeline caching** — Slow builds; cache npm/Maven/Docker layers

---

## Key Design Considerations

- **YAML over Classic** — Define pipelines as code in repository for versioning, code review, and reusability. Use templates for shared patterns across projects and teams
- **Agent Strategy** — Microsoft-hosted for standard builds (zero maintenance). Self-hosted for custom tools, network access, or cost optimization. VM scale set agents for auto-scaling based on queue depth
- **Secrets Management** — Variable groups linked to Azure Key Vault. Service connections with managed identities. Secure files for certificates. Never hardcode secrets in YAML
- **Deployment Strategies** — runOnce for simple deploys, canary for gradual traffic shift, rolling for batch update, blueGreen for instant switch. Use deployment slots on App Service
- **Governance** — Branch policies with required reviewers, environment approvals (manual gates), YAML templates enforced by security team, artifact promotion with security gates, audit logging
- **Enterprise Setup** — Single org with multiple projects per domain. Managed identities for service connections. Self-hosted agents with auto-scaling. Standardized templates and variable groups

---

## Real-World Scenarios

**Scenario 1: Enterprise Migration from Classic to YAML Pipelines**
A large enterprise has 500+ classic build/release pipelines that are unversioned and fragile. Migration strategy: create a YAML template library in a central repository with shared job templates for build, test, scan, and deploy. Each team converts their pipelines incrementally, starting with the simplest. Use Pipeline Decorators for org-wide governance (enforce required steps). Use variable groups linked to Key Vault for secrets. Retire classic pipelines after 6 months with a deprecation dashboard.

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
A FinTech company requires multi-person approval, security scanning, and compliance verification before production releases. Each environment promotes the same build artifact. Dev: auto-deploy, CI validation. Staging: security scan (Trivy, OWASP ZAP), integration tests. Pre-Prod: manual approval from QA lead + compliance officer, Azure Monitor query gate verifying error rates < 0.1%. Prod: change advisory board approval, canary deployment with auto-rollback.

**Scenario 3: Monorepo CI/CD with 80+ Microservices**
A product team uses a single repository with 80 microservices. Each service in its own folder. Use path triggers so only changed services build. Use a YAML template matrix that generates jobs dynamically based on changed paths. Use pipeline caching for npm/Maven/Docker layers. Fan-out pattern: one trigger pipeline, multiple parallel build jobs, fan-in to deploy only what changed.

---

## Scenario-Based Questions

1. **Q: Design a PCI-compliant CI/CD pipeline using Azure DevOps.**
   A: Use self-hosted agents in restricted network, variable groups linked to Key Vault, environment approvals with 2-person minimum, YAML templates from central repo, artifact retention policies, audit logging, and signed commits.

2. **Q: How do you implement blue-green deployment with App Service slots?**
   A: Use two deployment slots (staging, production). Deploy to staging, run smoke tests, swap slots. Auto-swap option available. Rollback is a swap back. Slot-specific settings remain sticky.

3. **Q: Design a multi-region deployment pipeline with release gates.**
   A: Multi-stage pipeline with environment-specific jobs. Deploy to Region 1 first, run integration tests, then deploy to Region 2. Azure Monitor query gates verify health before proceeding. Manual approval for production.

4. **Q: How would you manage Terraform infrastructure as code in Azure DevOps?**
   A: Use Terraform task in pipeline. Backend state in Azure Storage with locking. Validate stage (init, validate, plan). Apply stage with deployment environment and approvals. Separate state files per environment.

5. **Q: Design a self-hosted agent auto-scaling solution.**
   A: Use Azure VM Scale Set agents — auto-scales based on pipeline queue depth. Or use Kubernetes-based agents with the Kubernetes agent provider. Scale-to-zero when idle. Custom VM image with pre-cached tools.

6. **Q: How do you handle database migrations in CI/CD pipelines?**
   A: Use Flyway or EF Core migrations as a pipeline step. Run migrations before app deployment. Include rollback scripts. Use environment-specific connection strings from Key Vault. Validate migration scripts in CI.

7. **Q: Explain how Azure DevOps integrates with Azure Key Vault.**
   A: Create a variable group linked to Key Vault. Secrets are fetched at runtime and mapped to pipeline variables. Managed identity authenticates the pipeline. Secrets are masked in logs automatically.

8. **Q: Design a container CI/CD with Azure DevOps and AKS.**
   A: Pipeline builds Docker image, pushes to ACR, then deploys to AKS. Use Kubernetes manifest task or Helm. Blue-green deployment via AKS. Container scanning in CI. ACR geo-replication for multi-region.

9. **Q: What are YAML templates and how do you use them?**
   A: Templates are reusable YAML snippets. Job templates for common steps (build, test). Stage templates for deployment patterns. Template parameters for customization. Stored in a central repository.

10. **Q: How do you implement artifact promotion with security gates?**
    A: Each environment promotes the same build artifact. Staging runs security scan (Trivy, OWASP). Only if scan passes and gates (approvals, monitor queries) succeed, artifact promotes to production. Immutable artifact stored in Azure Artifacts.


---

## Interview Questions

1. **What is the difference between a Variable Group and a Variable in Azure DevOps?**
   A: Variables are simple key-value pairs defined per pipeline. Variable Groups are shared across pipelines, can link to Azure Key Vault for secrets, and are managed in the Library hub. Use variables for pipeline-specific values, variable groups for shared/secret values.

2. **What is a YAML template and why use it?**
   A: A reusable YAML file that contains jobs, steps, or stages. Templates enable DRY pipelines, enforce governance (security teams control deploy templates), and simplify multi-team adoption. Pass parameters to customize behavior. Store in a central repository with version tags.

3. **What is the difference between a Microsoft-hosted and self-hosted agent?**
   A: Microsoft-hosted agents are managed by Azure (zero maintenance), have common tools pre-installed, but limited to 60 minutes/job and no VNet access. Self-hosted agents require maintenance but offer custom tooling, network access, unlimited execution time, and cost savings at scale.

4. **What is a deployment group?**
   A: A logical set of target machines for deployment. Used in classic release pipelines. Each machine has an agent that runs deployment tasks. Supports rolling deployments with health checks. Being replaced by YAML environments with VM resources.

5. **How do you implement canary deployments in Azure DevOps?**
   A: In a YAML pipeline, set the deployment strategy to `canary` with traffic increments (e.g., 10%, 50%, 100%). Deploy to canary instances, run validation, then gradually increase traffic. Auto-rollback on health check failure via Azure Monitor gates.

6. **What is a service connection and how do you secure it?**
   A: A service connection stores credentials for external services (Azure, GitHub, Docker Hub). Secure by using Managed Identity where possible, limit scope to specific resources, use Workload Identity Federation for automatic credential rotation, and restrict service connection permissions in Project Settings.

7. **What is Azure Artifacts and how does it support CI/CD?**
   A: A package management service for NuGet, npm, Maven, Python, and Universal Packages. Supports upstream sources (proxies public feeds). Integrates with pipelines for automated publish and consume. Enables immutable, versioned artifacts for Build Once Deploy Many.

8. **What is the difference between Continuous Delivery and Continuous Deployment in Azure DevOps?**
   A: Continuous Delivery means every commit is automatically built and tested, deployment to production requires manual approval. Continuous Deployment is fully automated — every commit that passes all stages goes to production without human intervention. Choose CDelivery for risk-sensitive apps, CDeployment for mature DevOps practices.

9. **What is a pipeline decorator?**
   A: A pipeline decorator is a YAML template automatically injected into every pipeline in the organization by an admin. Used to enforce compliance: add a mandatory security scan step, require approval gates, inject anti-tampering checks. Defined in the `.azdevops` folder of a repository.

10. **What is the difference between `trigger` and `pr` triggers in Azure Pipelines?**
   A: `trigger` defines which branches trigger CI on push. `pr` defines which branches trigger a PR validation pipeline. Use CI triggers for main/develop branches, PR triggers for validating pull requests before merge. Configure path filters in both to limit scope.

---

## Developer Recommendations

- **Always use YAML pipelines over Classic** — Classic pipelines are not versionable, not reviewable, and cannot be stored in Git. YAML pipelines live in the repository alongside code, enabling peer review, branching, and audit trail. Migration is one-time effort; the long-term gain in traceability and automation is immense.

- **Link variable groups to Key Vault for all secrets** — Hardcoded secrets in pipeline YAML or library variables are a security breach waiting to happen. Key Vault-linked variable groups fetch secrets at runtime, log them as masked, and support automatic rotation. Use Managed Identity for authentication. Trade-off: pipeline startup takes 1-2 seconds longer for secret fetch.

- **Use environment-specific service connections** — Using a single service connection across environments is risky — a dev deployment mistake could affect production. Create separate service connections per environment with scoped permissions. Use Azure AD conditions to restrict each connection to its environment. Add manual approval gates on production connections.

- **Adopt templates early, enforce via decorators** — Without templates, every team writes their own pipeline logic — leading to inconsistent quality, missing security steps, and maintenance nightmares. Create a central template repository with standardized build/test/deploy stages. Use Pipeline Decorators to inject mandatory steps (security scan, artifact publish) into every pipeline.

- **Cache dependencies aggressively** — npm/node_modules, Maven/.m2, Docker layers, and NuGet packages rebuild on every pipeline run without caching. Use the Cache task (`CacheBeta@2`) with a restore key based on lock file hash. For Docker, use Docker layer caching with `--cache-from`. This can reduce build times by 60-80%.

- **Implement artifact promotion with immutability** — Rebuilding artifacts for each environment guarantees inconsistency. Build once, promote the exact same artifact through Dev -> Staging -> Pre-Prod -> Prod. Each environment validates the artifact but never rebuilds it. Use retention policies to clean old artifacts but keep the production version indefinitely for traceability.