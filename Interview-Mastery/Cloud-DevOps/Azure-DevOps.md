# Azure DevOps Study Guide

## 1. Executive Summary

Azure DevOps is Microsoft's end-to-end DevOps platform providing developer services for planning, developing, delivering, and maintaining applications. It includes Azure Boards (work tracking), Azure Repos (Git repos), Azure Pipelines (CI/CD), Azure Test Plans (testing), and Azure Artifacts (package management). With deep integration into Azure cloud services, Microsoft tooling, and broad third-party ecosystem support, Azure DevOps is a comprehensive platform for teams of any size.

## 2. Core Theory

### 2.1 Azure DevOps Services

```
+=============================================================+
|                    AZURE DEVOPS                             |
|  +--------+ +--------+ +--------+ +--------+ +----------+  |
|  | Boards | | Repos  | |Pipelines| |Test Plans| Artifacts| |
|  +--------+ +--------+ +--------+ +--------+ +----------+  |
+=============================================================+
```

### 2.2 Key Concepts

- **Organization**: Top-level container for all Azure DevOps resources
- **Project**: A logical container for code, work items, pipelines
- **Team**: A subset of project members with specific area paths
- **Agent Pool**: Collection of build/release agents
- **Variable Group**: Shared variables across pipelines
- **Service Connection**: Securely connect to external services
- **Environment**: Target deployment location with approvals

### 2.3 Organization Structure

```
Organization (mycompany)
  +-- Project A (Shopping Cart)
  |     +-- Repos (Git)
  |     +-- Pipelines (Builds + Releases)
  |     +-- Boards (Work Items)
  |     +-- Test Plans
  |     +-- Artifacts
  +-- Project B (User Mgmt)
        +-- Repos + Pipelines + Boards
```

## 3. Under-the-Hood Deep Dive

### 3.1 Agent Architecture

Pipeline -> Agent Job -> Agent Pool -> Agent VM -> Job Runner -> Steps

| Agent Type | Maintenance | Scaling | Capabilities |
|------------|-------------|---------|--------------|
| Microsoft-hosted | None | Automatic | Ubuntu, Windows, macOS |
| Self-hosted (VM) | Manual | Manual | Custom tools, network access |
| Self-hosted (K8s) | Automated | Auto-scaling | Ephemeral, containerized |
| VM Scale Set | Automated | Auto-scaling | Windows/Linux VMs |

### 3.2 Pipeline Structure (YAML)

```yaml
trigger:                    # CI trigger
pr:                         # PR trigger
schedules:                  # Scheduled triggers
variables:                  # Pipeline variables
stages:
  - stage: Build
    displayName: Build
    dependsOn: []
    condition: succeeded()
    variables:              # Stage-scoped variables
    jobs:
      - job: BuildJob
        pool:
          vmImage: ubuntu-latest
        strategy:
          matrix:
            Release:
              config: Release
            Debug:
              config: Debug
        steps:
          - script: echo Hello
          - task: DotNetCoreCLI@2
```

### 3.3 Classic vs YAML Pipelines

| Aspect | Classic Editor | YAML |
|--------|---------------|------|
| Definition | UI-based | Code in repository |
| Versioning | Manual | Git-tracked |
| Review | N/A | Pull requests |
| Reusability | Task groups | Templates, extends |
| Portability | Azure DevOps only | Multi-platform |

## 4. Production Code Examples

### 4.1 Multi-Stage Build and Deploy Pipeline

```yaml
trigger:
  branches:
    include: [main, develop, release/*]
  paths:
    exclude: [docs/*, README.md]

variables:
  - group: 'Global-Variables'
  - name: buildConfiguration
    value: 'Release'
  - name: version
    value: '2.$(Build.BuildId)'

stages:
  - stage: CI
    displayName: 'Continuous Integration'
    jobs:
      - job: Lint
        steps:
          - script: |
              npm ci
              npm run lint
              npm run format-check
            displayName: 'Run linters'

      - job: Build
        dependsOn: Lint
        strategy:
          matrix:
            Node18: { nodeVersion: '18.x' }
            Node20: { nodeVersion: '20.x' }
        steps:
          - task: NodeTool@0
            inputs: { versionSpec: '$(nodeVersion)' }
          - script: npm ci && npm run build
          - script: npm run test:ci -- --coverage
          - task: PublishTestResults@2
            inputs:
              testResultsFormat: 'JUnit'
              testResultsFiles: '**/junit.xml'
          - task: PublishCodeCoverageResults@2
            inputs:
              summaryFileLocation: '$(System.DefaultWorkingDirectory)/coverage/cobertura-coverage.xml'
          - task: PublishBuildArtifacts@1
            inputs:
              pathToPublish: 'dist'
              artifactName: 'drop'

      - job: Security
        dependsOn: Lint
        steps:
          - task: DependencyCheck@0
            inputs:
              projectName: 'MyApp'
              scanPath: '$(System.DefaultWorkingDirectory)'
              format: 'SARIF'
          - task: PublishSecurityAnalysisLogs@1
            condition: always()

  - stage: Deploy_Dev
    dependsOn: CI
    condition: succeeded()
    jobs:
      - deployment: Deploy
        environment: 'dev'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: '$(AZURE_SERVICE_CONNECTION)'
                    appName: 'app-dev'
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'
                - script: |
                    curl -f --retry 5 --retry-delay 10 https://dev.myapp.com/api/health

  - stage: Deploy_Staging
    dependsOn: Deploy_Dev
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
    jobs:
      - deployment: Deploy
        environment: 'staging'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: '$(AZURE_SERVICE_CONNECTION)'
                    appName: 'app-staging'
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'
                - task: AzurePowerShell@5
                  inputs:
                    azureSubscription: '$(AZURE_SERVICE_CONNECTION)'
                    Inline: |
                      Swap-AzWebAppSlot -ResourceGroupName 'rg-staging' `
                        -Name 'app-staging' `
                        -SourceSlot 'staging' -DestinationSlot 'production'

  - stage: Deploy_Prod
    dependsOn: Deploy_Staging
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: Deploy
        environment: 'prod'
        strategy:
          canary:
            increments: [10, 50, 100]
            deploy:
              steps:
                - download: current
                  artifact: drop
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: '$(AZURE_SERVICE_CONNECTION)'
                    appName: 'app-prod'
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'
            on:
              failure:
                steps:
                  - script: |
                      echo "Rolling back..."
                      az webapp deployment slot swap -g rg-prod -n app-prod --slot staging --action swap
```

### 4.2 Terraform Pipeline

```yaml
trigger:
  branches: { include: [main] }
  paths: { include: [infrastructure/*] }

stages:
  - stage: Validate
    jobs:
      - job: Validate
        steps:
          - task: TerraformInstaller@1
            inputs: { terraformVersion: '1.7.0' }
          - task: TerraformTaskV4@4
            inputs:
              provider: 'azurerm'
              command: 'init'
              workingDirectory: 'infrastructure'
              backendServiceArm: '$(AZURE_SERVICE_CONNECTION)'
              backendAzureRmResourceGroupName: 'rg-terraform-state'
              backendAzureRmStorageAccountName: 'stterraformstate'
              backendAzureRmContainerName: 'tfstate'
              backendAzureRmKey: 'terraform.tfstate'
          - task: TerraformTaskV4@4
            inputs:
              provider: 'azurerm'
              command: 'validate'
              workingDirectory: 'infrastructure'
          - task: TerraformTaskV4@4
            inputs:
              provider: 'azurerm'
              command: 'plan'
              workingDirectory: 'infrastructure'
              commandOptions: '-out=tfplan'

  - stage: Deploy
    dependsOn: Validate
    condition: succeeded()
    jobs:
      - deployment: Apply
        environment: 'terraform'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: TerraformTaskV4@4
                  inputs:
                    provider: 'azurerm'
                    command: 'apply'
                    workingDirectory: 'infrastructure'
                    commandOptions: 'tfplan'
```

### 4.3 AKS Deployment Pipeline

```yaml
trigger:
  branches: { include: [main] }

variables:
  - group: 'Kubernetes-Variables'
  - name: imageRepository
    value: 'myacr.azurecr.io/myapp'
  - name: tag
    value: '$(Build.BuildId)'

stages:
  - stage: Build
    jobs:
      - job: Build
        steps:
          - task: Docker@2
            inputs:
              containerRegistry: '$(DOCKER_REGISTRY_SERVICE_CONNECTION)'
              repository: 'myapp'
              command: 'buildAndPush'
              tags: '$(tag)'

  - stage: Deploy
    dependsOn: Build
    jobs:
      - deployment: Deploy
        environment: 'aks-production'
        strategy:
          blueGreen:
            deploy:
              steps:
                - task: KubernetesManifest@1
                  inputs:
                    action: 'deploy'
                    kubernetesServiceConnection: '$(AKS_SERVICE_CONNECTION)'
                    namespace: 'production'
                    manifests: 'k8s/deployment.yaml'
                    containers: '$(imageRepository):$(tag)'
```

### 4.4 Variable Groups and Key Vault

```yaml
variables:
  - group: 'Production-Secrets'        # Linked to Azure Key Vault
  - group: 'Alert-Webhooks'
  - name: connectionString
    value: $(SQL-CONNECTION-STRING)     # Fetched from Key Vault automatically
```

## 5. Real-World Scenarios

### 5.1 Self-Hosted Agent Setup

```bash
#!/bin/bash
AGENT_VERSION="3.234.0"
AGENT_URL="https://vstsagentpackage.azureedge.net/agent/${AGENT_VERSION}/vsts-agent-linux-x64-${AGENT_VERSION}.tar.gz"
ORG_URL="https://dev.azure.com/myorg"
AGENT_POOL="Production-Agents"

curl -O $AGENT_URL
tar xzf vsts-agent-linux-x64-${AGENT_VERSION}.tar.gz -C $HOME/agent
cd $HOME/agent

./config.sh --unattended \
    --url $ORG_URL \
    --auth PAT --token "<PAT>" \
    --pool "$AGENT_POOL" \
    --agent "prod-agent-$(hostname)" \
    --replace --work "_work" --runAsService --once

sudo ./svc.sh install && sudo ./svc.sh start
```

### 5.2 Approval Gates

```yaml
environment: 'production'
approvals:
  - group: 'Release-Managers'
    minimumApprovers: 2
    timeout: 43200

gates:
  - name: 'Security Gate'
    conditions:
      - type: 'AzureMonitorQuery'
        metricNamespace: 'azuremonitor'
        metricName: 'vulnerability-severity'
        operator: 'lessThan'
        threshold: 3
```

## 6. Performance

### 6.1 Pipeline Optimization

```yaml
# Path filtering for monorepo
trigger:
  paths:
    include: [src/api/*]
    exclude: [tests/*, docs/*]

# Parallel execution
jobs:
  - job: Lint
  - job: UnitTests
    dependsOn: []
  - job: IntegrationTests
    dependsOn: []

# Self-hosted agents for faster builds
pool:
  name: 'Production-Agents'
  demands:
    - Agent.OS -equals Linux
    - CPU -equals 8
```

### 6.2 Caching

```yaml
- task: Cache@2
  inputs:
    key: 'npm | "$(Agent.OS)" | package-lock.json'
    path: '$(npm_config_cache)'
    cacheHitVar: 'CACHE_RESTORED'
    restoreKeys: |
      npm | "$(Agent.OS)"
      npm
```

## 7. Security

### 7.1 Secrets Management

```yaml
# Variable group linked to Azure Key Vault
variables:
  - group: 'App-Secrets'
  - name: DbPassword
    value: $(DB-PASSWORD)

# Secure files
- task: DownloadSecureFile@1
  name: certificate
  inputs:
    secureFile: 'wildcard-myapp-com.pfx'
```

### 7.2 Pipeline Permissions

```yaml
# Managed identity for Azure resources
variables:
  - name: ARM_USE_MSI
    value: true
```

## 8. Common Mistakes

| Mistake | Impact | Solution |
|---------|--------|----------|
| Classic pipelines | No code-as-config | Migrate to YAML |
| No variable groups | Secret sprawl | Centralize in Library |
| Hardcoding subscription IDs | Fragile | Use service connections |
| No deployment slots | Downtime | Use staging slots |
| No path filtering | Slow CI | Use path triggers |
| No environment approvals | Accidental deploys | Add approval gates |
| No artifact retention | Storage costs | Set retention policies |

## 9. Senior Engineer Perspective

### 9.1 Migration Classic to YAML

1. Export classic pipeline to JSON
2. Use YAML assistant to translate
3. Create template for shared steps
4. Test alongside classic pipeline
5. Remove classic after migration

### 9.2 Enterprise Setup

- Structure: One org, multiple projects per domain
- Security: Managed identities for service connections
- Governance: Branch policies, required reviewers
- Standardization: YAML templates, variable groups
- Scalability: Self-hosted agents with auto-scaling

## 10. Interview Questions (Easy)

1. What is Azure DevOps and its services?
2. What is the difference between build and release pipeline?
3. What is an agent pool?
4. How do you trigger a pipeline on push?
5. What is a variable group?
6. What is a service connection?
7. How do you publish build artifacts?
8. What is Azure Boards?
9. What is the difference between project and organization?
10. How do you view pipeline logs?

## 10. Interview Questions (Medium)

11. What is the difference between YAML and classic pipelines?
12. How do you implement multi-stage pipelines?
13. What are deployment groups?
14. How do you set up environment approvals?
15. How does Azure DevOps integrate with Key Vault?
16. What is a YAML template?
17. How do you implement canary deployments?
18. What are pipeline caching strategies?
19. How do you handle database migrations?
20. Microsoft-hosted vs self-hosted agents?

## 11. Advanced Interview Questions (Hard)

1. Design a PCI-compliant CI/CD pipeline using Azure DevOps.
2. Implement blue-green deployment with App Service slots.
3. Design multi-region deployment with release gates.
4. Manage infrastructure as code with Terraform.
5. Self-hosted agent auto-scaling solution.
6. Feature flag deployments across environments.
7. Container CI/CD with Azure DevOps and AKS.
8. Artifact promotion with security gates.
9. Database-first deployment with backward compatibility.
10. Secrets rotation in Azure DevOps pipelines.

## 11. Advanced Interview Questions (System Design)

11. Multi-tenant Azure DevOps for 500+ teams.
12. Global deployment pipeline across 10 regions.
13. Compliance and audit system for regulated deployments.
14. Cost-optimized pipeline infrastructure.
15. Disaster recovery for Azure DevOps itself.
16. GitOps with Azure DevOps and ArgoCD.
17. Pipeline managing app + DB changes.
18. Release management with auto-rollback.
19. Cross-project dependency management.
20. Mobile app CI/CD with Azure DevOps.

## 12. Expert-Level Interview Questions (Architect)

1. Design an enterprise Azure DevOps platform for 10,000+ developers with federated identity, compliance controls, and global agent pools.

2. Architect migration from Jenkins to Azure DevOps YAML for 2,000+ pipelines, 500+ repos, 100+ teams.

3. Design multi-cloud deployment orchestration (AWS, GCP, Azure) with unified approvals.

4. Design an Azure DevOps extension for policy enforcement, cost tracking, and governance.

5. Pipeline composition framework from approved building blocks with security enforcement.

6. Real-time pipeline analytics identifying slow stages, flaky tests, cost optimization.

7. Zero-trust security model for Azure DevOps pipelines with attestation.

8. Cross-organization DevOps topology supporting M&A scenarios.

9. AI-powered pipeline failure prediction and auto-remediation.

10. Universal deployment abstraction layer for on-premise, cloud, and edge.

## 13. Debugging & Troubleshooting

### 13.1 Pipeline Issues

```yaml
variables:
  - name: System.Debug
    value: true

- script: |
    echo "Agent OS: $(Agent.OS)"
    echo "Build ID: $(Build.BuildId)"
    echo "Source Version: $(Build.SourceVersion)"
```

### 13.2 Agent Issues

```bash
cd $HOME/agent
sudo ./svc.sh status
cat $HOME/agent/_diag/Agent_*.log | grep -i error
sudo ./svc.sh restart
curl -I https://dev.azure.com/myorg
```

### 13.3 API Queries

```powershell
$url = "https://dev.azure.com/myorg/myproject/_apis/pipelines/123/runs?api-version=7.0"
$runs = Invoke-RestMethod -Uri $url -Headers @{Authorization="Basic $base64AuthInfo"}
$runs.value | Select id, result, createdDate
```

## 14. Comparison Section

### Azure DevOps vs GitHub Actions

| Feature | Azure DevOps | GitHub Actions |
|---------|-------------|---------------|
| Work Tracking | Azure Boards (rich) | Issues (basic) |
| CI/CD | Azure Pipelines | GitHub Actions |
| Package Mgmt | Azure Artifacts | GitHub Packages |
| Test Plans | Yes | Basic |
| Marketplace | Extensions | Actions |
| Hosted Agents | Linux/Windows/macOS | Linux/Windows/macOS |
| Secrets | Variable groups + Key Vault | Encrypted secrets |
| Environments | Approvals + Gates | Protection rules |

### Azure DevOps vs Jenkins

| Feature | Azure DevOps | Jenkins |
|---------|-------------|---------|
| Hosting | Cloud + self-hosted | Self-hosted |
| Setup | 5 minutes | 30+ min |
| Maintenance | Zero (cloud) | Significant |
| Scalability | Built-in | Manual |
| Pipeline as Code | YAML | Groovy |

## 15. Revision Notes

```
Azure DevOps = Boards + Repos + Pipelines + Test Plans + Artifacts

Organization -> Project -> Repo/Pipeline/Board
Agent Pool -> Agent -> Job -> Step
Service Connection -> External Service
Variable Group -> Shared Variables
Environment -> Deployment Target + Approvals

Common YAML Keywords:
  trigger, pr, schedules, variables, stages, stage,
  dependsOn, condition, jobs, job, pool, steps, task, script
```

## 16. Cheat Sheet

```text
+======================================================================+
|                    AZURE DEVOPS CHEAT SHEET                           |
+======================================================================+

  PIPELINE TRIGGERS
+----------------------------------------------------------------------+
| trigger: branches: { include: [main] }    | CI on branch push        |
| pr: branches: { include: [main] }         | PR trigger               |
| trigger: none                              | Manual only              |
| schedules: - cron: '0 2 * * *'            | Scheduled                |
+----------------------------------------------------------------------+

  COMMON TASKS
+----------------------------------------------------------------------+
| DotNetCoreCLI@2      | .NET build, test, publish                     |
| Npm@1                | npm install, custom                          |
| Docker@2             | Docker build, push                           |
| AzureWebApp@1        | Deploy to App Service                        |
| KubernetesManifest@1 | Deploy to AKS                               |
| Cache@2              | Cache dependencies                           |
+----------------------------------------------------------------------+

  VARIABLE SYNTAX
+----------------------------------------------------------------------+
| $(variableName)              | Macro syntax                          |
| ${{ variables.var }}         | Template expression (compile time)    |
| variables:                   | Define inline                         |
|   - name: VAR value: val     |                                      |
| variables:                   | Reference group                      |
|   - group: 'MyGroup'         |                                      |
+----------------------------------------------------------------------+

  CONDITIONS
+----------------------------------------------------------------------+
| succeeded(), failed(), always()                                      |
| eq(variables['var'], 'val'), in(variables['var'], 'a','b')           |
| startsWith(variables['var'], 'refs/heads/')                          |
+----------------------------------------------------------------------+

  DEPLOYMENT STRATEGIES
+----------------------------------------------------------------------+
| runOnce:                     | Deploy once                           |
| canary: increments: [10,50,100] | Gradual traffic shift            |
| rolling: maxParallel: 2     | Rolling update                        |
| blueGreen:                   | Blue-green deployment                 |
+----------------------------------------------------------------------+

  AGENT POOLS
+----------------------------------------------------------------------+
| pool: vmImage: ubuntu-latest  | Microsoft-hosted                     |
| pool: name: 'MyPool'          | Self-hosted                         |
| demands: - Agent.OS -equals Linux | Agent capabilities              |
+----------------------------------------------------------------------+

+======================================================================+
|  PRO TIPS: Use YAML, not classic. Store secrets in Key Vault.        |
|  Use managed identities. Branch policies for production.             |
|  Deployment slots for zero-downtime. Set artifact retention.         |
|  Use YAML templates for reusable patterns.                          |
+======================================================================+
```
