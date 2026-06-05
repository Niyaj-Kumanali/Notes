# CI/CD Study Guide

## 1. Executive Summary

CI/CD (Continuous Integration and Continuous Delivery/Deployment) is a set of practices that automate the software delivery process. Continuous Integration automatically builds and tests code changes on every commit, catching integration issues early. Continuous Delivery extends this by automatically deploying to staging environments, with manual approval for production. Continuous Deployment automates the entire pipeline to production. CI/CD is fundamental to modern software engineering, enabling faster releases, higher quality, and reduced risk.

## 2. Core Theory

### 2.1 The CI/CD Pipeline

```
Source -> Build -> Test -> Deploy to Staging -> Integration Tests -> Deploy to Production
  ^                                                                                  |
  |__________________________________________________________________________________|
```

### 2.2 Key Concepts

- **Continuous Integration**: Developers merge changes frequently (multiple times daily). Each merge triggers an automated build and test suite.
- **Continuous Delivery**: Code is always in a deployable state. Every successful build could be released to production.
- **Continuous Deployment**: Every change that passes all pipeline stages is automatically deployed to production.

### 2.3 The Feedback Loop

CI/CD shortens the feedback loop:

```
Developer commits code (minutes) -> Build (seconds-minutes) -> 
Unit Tests (seconds) -> Integration Tests (minutes) -> 
Staging Deploy (minutes) -> E2E Tests (minutes) -> 
Production Deploy (minutes)
```

Without CI/CD:

```
Developer works for weeks -> Manual merge hell (days) -> 
Manual deployment (hours-days) -> Production issues detected by users
```

### 2.4 Pipeline Stages in Detail

| Stage | Purpose | Duration | Frequency |
|-------|---------|----------|-----------|
| Code Commit | Trigger pipeline | Instant | Every push |
| Lint/Static Analysis | Code quality | < 1 min | Every push |
| Unit Tests | Isolated correctness | < 5 min | Every push |
| Build | Artifact creation | < 10 min | Every push |
| Integration Tests | Component interaction | < 15 min | Every push |
| Security Scan | Vulnerability check | < 10 min | Every push |
| Deploy Staging | Pre-prod validation | < 5 min | Every merge to main |
| E2E Tests | Full workflow | < 20 min | Every deploy to staging |
| Performance Test | Load/Stress | < 30 min | Scheduled |
| Deploy Production | Release | < 10 min | On demand / automatic |

## 3. Under-the-Hood Deep Dive

### 3.1 Build Artifacts and Versioning

```yaml
# Semantic versioning with CI metadata
version: MAJOR.MINOR.PATCH+BUILD_METADATA
# Example: 1.3.2+20240605.1430.ab12cd3
```

### 3.2 Artifact Repository Strategies

```
Docker Registry (ECR, Docker Hub, GCR)
  |-- team-a/
  |     |-- service-x:1.2.3   (tagged version)
  |     |-- service-x:latest  (latest stable)
  |     |-- service-x:sha-ab12cd3  (immutable reference)
  |
  |-- team-b/
        |-- service-y:2.1.0
```

### 3.3 Pipeline as Code

Pipelines defined as code (YAML, HCL, Groovy) provide version control, code review, and reproducibility:

```yaml
# Declarative pipeline (Azure DevOps)
trigger:
  branches:
    include:
      - main
      - release/*

variables:
  buildConfiguration: 'Release'
  majorVersion: 1

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - script: dotnet build --configuration $(buildConfiguration)
```

```groovy
// Scripted pipeline (Jenkins)
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Building...'
            }
        }
    }
}
```

### 3.4 Branch Strategies for CI/CD

```text
Trunk-Based Development
  feature/foo -> main (short-lived branches, < 1 day)

GitHub Flow
  feature/foo -> main -> deploy

GitFlow
  feature/foo -> develop -> release/x.x -> main -> tag

GitLab Flow
  feature/foo -> main -> production
  feature/foo -> main -> stable
```

### 3.5 Testing Pyramid in CI/CD

```
          /\
         /E2E\          Few tests, slow, high confidence
        /------\
       /Integration\    Some tests, medium speed
      /--------------\
     /   Unit Tests    \  Many tests, fast, low-level
    /--------------------\
```

## 4. Production Code Examples

### 4.1 Full .NET CI/CD Pipeline

```yaml
# azure-pipelines.yml
trigger:
  - main
  - develop

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: Production-Secrets
  - name: buildConfiguration
    value: 'Release'

stages:
  - stage: Build
    displayName: 'Build and Test'
    jobs:
      - job: Build
        steps:
          - task: DotNetCoreCLI@2
            displayName: 'Restore packages'
            inputs:
              command: 'restore'
              projects: '**/*.csproj'

          - task: DotNetCoreCLI@2
            displayName: 'Build solution'
            inputs:
              command: 'build'
              projects: '**/*.csproj'
              arguments: '--configuration $(buildConfiguration) --no-restore'

          - task: DotNetCoreCLI@2
            displayName: 'Run unit tests'
            inputs:
              command: 'test'
              projects: '**/*.Tests.csproj'
              arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage"'

          - task: PublishCodeCoverageResults@1
            displayName: 'Publish code coverage'
            inputs:
              codeCoverageTool: 'Cobertura'
              summaryFileLocation: '$(Agent.TempDirectory)/**/coverage.cobertura.xml'

          - task: DotNetCoreCLI@2
            displayName: 'Publish artifacts'
            inputs:
              command: 'publish'
              publishWebProjects: true
              arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'

          - task: PublishBuildArtifacts@1
            displayName: 'Upload artifacts'
            inputs:
              pathToPublish: '$(Build.ArtifactStagingDirectory)'
              artifactName: 'drop'

  - stage: SecurityScan
    displayName: 'Security Scanning'
    dependsOn: Build
    jobs:
      - job: OWASPScan
        steps:
          - script: |
              dotnet tool install --global dotnet-security-scan
              dotnet-security-scan --project **/*.csproj --output $(Build.ArtifactStagingDirectory)/security-report.json
            displayName: 'Run OWASP dependency check'

  - stage: DeployStaging
    displayName: 'Deploy to Staging'
    dependsOn: SecurityScan
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
    jobs:
      - deployment: Deploy
        environment: staging
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop

                - task: AzureWebApp@1
                  displayName: 'Deploy to Azure App Service'
                  inputs:
                    azureSubscription: '$(AZURE_SERVICE_CONNECTION)'
                    appName: 'app-staging'
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'
                    deploymentMethod: 'zipDeploy'

                - script: |
                    curl -X GET https://staging.myapp.com/api/health
                    echo "Smoke test passed"
                  displayName: 'Run smoke tests'

  - stage: DeployProduction
    displayName: 'Deploy to Production'
    dependsOn: DeployStaging
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: Deploy
        environment: production
        strategy:
          canary:
            increments: [10, 50, 100]
            deploy:
              steps:
                - download: current
                  artifact: drop

                - task: AzureWebApp@1
                  displayName: 'Deploy to Azure App Service'
                  inputs:
                    azureSubscription: '$(AZURE_SERVICE_CONNECTION)'
                    appName: 'app-production'
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'

            routeTraffic:
              steps:
                - script: |
                    echo "Routing 10% traffic, then 50%, then 100%"

            on:
              failure:
                steps:
                  - script: echo "Rolling back..."
              success:
                steps:
                  - script: echo "Deployment successful"
```

### 4.2 GitHub Actions CI/CD

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '20.x'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
      - run: npm ci
      - run: npm run lint
      - run: npm run format-check

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
      - run: npm ci
      - run: npm run test -- --coverage
      - uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v3
        with:
          name: build
          path: dist/

  docker-build:
    needs: test
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
            type=sha,prefix=,format=short
            type=ref,event=branch
            type=semver,pattern={{version}}
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: docker-build
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: superfly/flyctl-actions@1.3
        with:
          args: "deploy --image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}"

  deploy-production:
    needs: docker-build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - run: |
          flyctl deploy \
            --image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }} \
            --strategy canary \
            --wait-timeout 300
```

### 4.3 Jenkins Declarative Pipeline

```groovy
// Jenkinsfile
pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '--network host'
        }
    }

    parameters {
        string(name: 'VERSION', defaultValue: '', description: 'Release version')
        booleanParam(name: 'DEPLOY_TO_PROD', defaultValue: false, description: 'Deploy to production')
    }

    environment {
        REGISTRY = 'registry.mycompany.com'
        IMAGE_NAME = "${REGISTRY}/myapp"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            parallel {
                stage('NPM') {
                    steps {
                        sh 'npm ci'
                    }
                }
                stage('Go Modules') {
                    agent {
                        docker { image 'golang:1.21' }
                    }
                    steps {
                        sh 'go mod download'
                    }
                }
            }
        }

        stage('Quality Gate') {
            parallel {
                stage('Lint') {
                    steps {
                        sh 'npm run lint'
                    }
                }
                stage('Security Scan') {
                    steps {
                        sh 'npm audit --audit-level=high'
                    }
                }
            }
        }

        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm run test:unit -- --coverage'
                    }
                    post {
                        always {
                            junit 'reports/unit/*.xml'
                        }
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'npm run test:integration'
                    }
                    post {
                        always {
                            junit 'reports/integration/*.xml'
                        }
                    }
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    docker.build("${IMAGE_NAME}:${params.VERSION ?: env.BUILD_NUMBER}")
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY}", 'docker-credentials') {
                        docker.image("${IMAGE_NAME}:${params.VERSION ?: env.BUILD_NUMBER}").push()
                    }
                }
                sh 'kubectl set image deployment/myapp-staging myapp=${IMAGE_NAME}:${params.VERSION ?: env.BUILD_NUMBER}'
            }
        }

        stage('E2E Tests') {
            steps {
                sh 'npm run test:e2e -- --base-url=https://staging.myapp.com'
            }
        }

        stage('Deploy to Production') {
            when {
                expression { params.DEPLOY_TO_PROD }
            }
            input {
                message "Confirm deployment to production?"
                ok "Deploy"
                submitterParameter 'APPROVER'
            }
            steps {
                script {
                    sh """
                        kubectl set image deployment/myapp-prod myapp=${IMAGE_NAME}:${params.VERSION ?: env.BUILD_NUMBER}
                        kubectl rollout status deployment/myapp-prod
                    """
                }
            }
        }
    }

    post {
        success {
            emailext(
                subject: "Pipeline SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                to: 'team@mycompany.com',
                body: "The pipeline completed successfully."
            )
        }
        failure {
            emailext(
                subject: "Pipeline FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                to: 'team@mycompany.com',
                body: "The pipeline failed at stage ${env.STAGE_NAME}."
            )
        }
        cleanup {
            cleanWs()
        }
    }
}
```

### 4.4 Automated Rollback Script

```bash
#!/bin/bash
# rollback.sh

set -euo pipefail

NAMESPACE="${1:?Namespace required}"
DEPLOYMENT="${2:?Deployment name required}"
REVISION="${3:-}"

echo "=== Rolling back deployment $DEPLOYMENT in namespace $NAMESPACE ==="

if [ -n "$REVISION" ]; then
    echo "Rolling back to revision $REVISION"
    kubectl rollout undo deployment/"$DEPLOYMENT" \
        -n "$NAMESPACE" \
        --to-revision="$REVISION"
else
    echo "Rolling back to previous revision"
    kubectl rollout undo deployment/"$DEPLOYMENT" \
        -n "$NAMESPACE"
fi

echo "Waiting for rollout to complete..."
kubectl rollout status deployment/"$DEPLOYMENT" \
    -n "$NAMESPACE" \
    --timeout=300s

echo "=== Rollback completed successfully ==="
```

## 5. Real-World Scenarios

### 5.1 Monorepo with Selective CI/CD

```yaml
# Azure DevOps: path triggers for monorepo
trigger:
  paths:
    include:
      - src/service-a/*
      - src/shared/*
    exclude:
      - docs/*
      - README.md

# GitHub Actions: path filters
on:
  push:
    paths:
      - 'src/service-a/**'
      - 'src/shared/**'
      - '!docs/**'
```

### 5.2 Multi-Environment Promotion

```yaml
# Environment promotion gates
stages:
  - stage: Build
  - stage: Deploy_Dev
    dependsOn: Build
  - stage: Deploy_QA
    dependsOn: Deploy_Dev
    condition: succeeded()
  - stage: Deploy_Staging
    dependsOn: Deploy_QA
    condition: succeeded()
  - stage: Deploy_Prod
    dependsOn: Deploy_Staging
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    trigger: manual  # Requires approval
```

### 5.3 Database CI/CD

```yaml
# Database migration pipeline
stages:
  - stage: ValidateMigrations
    jobs:
      - job: Validate
        steps:
          - script: |
              # Validate SQL scripts
              sqlcmd -S localhost -U sa -P $SA_PASSWORD -d master -i migrations/validate.sql

  - stage: MigrateDev
    dependsOn: ValidateMigrations
    jobs:
      - job: RunMigrations
        steps:
          - script: |
              flyway -url=jdbc:sqlserver://dev-db -schemas=dbo migrate

  - stage: MigrateStaging
    dependsOn: MigrateDev
    jobs:
      - job: RunMigrations
        steps:
          - script: |
              flyway -url=jdbc:sqlserver://staging-db -schemas=dbo migrate

  - stage: MigrateProduction
    dependsOn: MigrateStaging
    jobs:
      - job: RunMigrations
        steps:
          - script: |
              flyway -url=jdbc:sqlserver://prod-db -schemas=dbo migrate
```

## 6. Performance

### 6.1 Pipeline Optimization

```text
Strategy                          Build Time    Resource Usage
Single stage                      15 min        100%
Parallel stages                   5 min         300%
Caching dependencies              8 min         100%
Incremental builds (NX/Turborepo) 3 min         150%
Distributed builds                2 min         400%
```

### 6.2 Caching Strategies

```yaml
# GitHub Actions dependency caching
- uses: actions/cache@v3
  with:
    path: |
      ~/.npm
      node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

# Azure DevOps caching
- task: Cache@2
  inputs:
    key: 'npm | "$(Agent.OS)" | package-lock.json'
    path: $(npm_config_cache)
```

### 6.3 Build Time Reduction Techniques

```text
1. Parallel execution of independent stages
2. Dependency caching (package managers, Docker layers)
3. Incremental builds (only build changed components)
4. Optimize test suite (prioritize failing tests, test impact analysis)
5. Use faster build agents (more CPU, SSD storage)
6. Pre-baked build environments (custom VM images)
7. Remote caching (shared build cache across developers)
8. Skip CI for trivial changes (docs, README)
```

## 7. Security

### 7.1 Pipeline Security Best Practices

```yaml
# Never store secrets in pipeline definitions
variables:
  - group: Production-Secrets  # Reference Azure DevOps variable group
  - name: DB_PASSWORD
    value: $(DB_PASSWORD_SECRET)  # Reference from Azure Key Vault

# GitHub Actions secrets
env:
  API_KEY: ${{ secrets.API_KEY }}
```

### 7.2 Supply Chain Security

```yaml
# Software Bill of Materials (SBOM) generation
- name: Generate SBOM
  run: |
    dotnet tool install --global Microsoft.Sbom.DotNetTool
    sbom-tool generate -b . -bc . -pn MyApp -pv 1.0.0

# Verify signed commits
- name: Verify commit signatures
  uses: 1password/check-signed-commits-action@v1

# Container image signing
- name: Sign container image
  uses: sigstore/cosign-installer@v3.1.1
- name: Sign image
  run: |
    cosign sign --key env://COSIGN_PRIVATE_KEY ghcr.io/myorg/myapp@sha256:${{ steps.docker.outputs.digest }}
```

### 7.3 Pipeline Hardening

```yaml
# Principle of least privilege for CI/CD
# GitHub Actions: minimum permissions
permissions:
  contents: read
  packages: write
  issues: none
  pull-requests: read

# Azure DevOps: restrict service connections
# Use managed identities instead of service principals where possible
```

## 8. Common Mistakes

| Mistake | Impact | Solution |
|---------|--------|----------|
| Building artifacts in multiple stages | Inconsistency, wasted time | Build once, promote same artifact |
| Hardcoded secrets in pipeline YAML | Security breach | Use secret variables/key vaults |
| No pipeline for database changes | Migration failures | Include DB migrations in pipeline |
| Building on every branch without filter | Wasted resources, slow CI | Use path/branch triggers |
| Ignoring flaky tests | Unreliable pipeline | Quarantine and fix flaky tests |
| Long-running pipelines | Slow feedback | Parallelize, optimize, incremental builds |
| No artifact versioning | Unable to trace releases | Semantic versioning with commit SHA |
| Manual deployment steps | Human error | Automate everything |
| No rollback strategy | Extended outages | Implement automatic rollback |
| Different environments behave differently | Surprises in production | Infrastructure as code, identical configs |

## 9. Senior Engineer Perspective

### 9.1 CI/CD Maturity Model

| Level | Name | Characteristics |
|-------|------|----------------|
| 0 | No CI/CD | Manual builds, deployments |
| 1 | Basic CI | Automated build + unit tests |
| 2 | Automated Testing | Integration, E2E tests automated |
| 3 | Continuous Delivery | Automated deploy to staging, manual production |
| 4 | Continuous Deployment | Fully automated to production |
| 5 | Optimized | Canary, feature flags, A/B testing, auto-rollback |

### 9.2 Metrics to Track

```yaml
# DORA metrics
deployment_frequency: "daily"  # How often you deploy
lead_time_for_changes: "hours" # Time from commit to production
mean_time_to_recover: "minutes" # Time to recover from failure
change_failure_rate: "5%"      # Percentage of changes causing failures
```

### 9.3 CI/CD Anti-Patterns

```text
1. "The pipeline is too slow, so we don't run it" - Increase parallelism, optimize
2. "We'll fix the tests after the release" - Tests must pass before release
3. "The build works on my machine" - Build in CI, not locally
4. "One pipeline to rule them all" - Monorepo needs selective execution
5. "We don't need rollbacks" - You always need rollbacks
6. "Tests are blocking our velocity" - Tests should be fast and reliable
7. "We deploy on Fridays" - Deploy early in the week, have on-call
```

## 10. Interview Questions (Easy)

1. What is CI/CD and what are its benefits?
2. What is the difference between Continuous Delivery and Continuous Deployment?
3. What stages are typically in a CI/CD pipeline?
4. What is a build artifact?
5. What is a trigger in CI/CD?
6. Why is testing important in a CI/CD pipeline?
7. What is a pull request and how does it relate to CI?
8. What is the purpose of a build server?
9. What is trunk-based development?
10. What are environment variables in CI/CD?

## 10. Interview Questions (Medium)

11. How do you handle secrets in a CI/CD pipeline?
12. What is the difference between a declarative and scripted pipeline?
13. Explain the concept of "build once, deploy many."
14. How do you implement rollbacks in CI/CD?
15. What is infrastructure as code and how does it relate to CI/CD?
16. How do you handle database migrations in CI/CD?
17. What is a canary deployment and how do you implement it?
18. How do you optimize a slow CI/CD pipeline?
19. What is the difference between blue-green and rolling deployments?
20. Explain the testing pyramid in the context of CI/CD.

## 11. Advanced Interview Questions (Hard)

1. Design a CI/CD system for a monorepo with 100+ microservices.
2. How would you implement progressive delivery (canary, feature flags) in CI/CD?
3. Design a pipeline that supports multi-cloud deployment (AWS, Azure, GCP).
4. How do you ensure compliance (SOC2, HIPAA) in a CI/CD pipeline?
5. Design a zero-downtime deployment strategy for a stateful application.
6. How do you handle schema changes in a distributed database during deployment?
7. Design a pipeline that can deploy to 1000+ Kubernetes clusters.
8. How do you implement artifact promotion with security gates?
9. Design a CI/CD system for ML models (MLOps pipeline).
10. How do you handle versioning and dependency management in a microservices CI/CD?

## 11. Advanced Interview Questions (System Design)

11. Design a multi-tenant CI/CD platform for 500+ development teams.
12. Design a global CI/CD system with build agents across 5 regions.
13. Design a CI/CD pipeline for a serverless architecture.
14. Design a pipeline that handles both mobile app releases and backend services.
15. Design a cost-optimized CI/CD infrastructure using spot instances.
16. Design a pipeline security system that prevents supply chain attacks.
17. Design a CI/CD observability platform.
18. Design a feature flag management system integrated with CI/CD.
19. Design a pipeline that auto-scales based on queue depth.
20. Design a disaster recovery strategy for the CI/CD system itself.

## 12. Expert-Level Interview Questions (Architect)

1. Design a CI/CD platform for a regulated financial institution that must enforce segregation of duties, audit trails, and immutable deployment records while maintaining deployment velocity.

2. Architect a build system that handles a monorepo with 5,000+ developers, 10,000+ daily commits, and 2,000+ microservices with sub-second dependency resolution.

3. Design a CI/CD pipeline that supports both traditional VM deployments and Kubernetes, with gradual migration capability and zero downtime.

4. How would you design a deployment verification system that automatically validates correctness using production traffic shadowing, canary analysis, and synthetic monitoring?

5. Architect a CI/CD platform that serves multiple business units with different compliance requirements (PCI, HIPAA, FedRAMP) on shared infrastructure while ensuring complete isolation.

6. Design a pipeline that automatically generates and manages Kubernetes manifests, Helm charts, Kustomize overlays, and Terraform configurations from a single source of truth.

7. How would you design a cross-organizational CI/CD platform that enables inner-source contributions with automatic dependency resolution, security scanning, and build caching?

8. Architect a chaos engineering pipeline that automatically injects failures into staging environments and validates system resilience as part of every deployment.

9. Design a CI/CD observability system that correlates deployments with application performance metrics, error budgets, and business KPIs to automatically halt bad rollouts.

10. How would you design a universal CI/CD abstraction layer that can run the same pipeline across GitHub Actions, Azure DevOps, Jenkins, GitLab CI, and AWS CodePipeline?

## 13. Debugging & Troubleshooting

### 13.1 Common Pipeline Failures

```bash
# Build failure
# Check build logs for compile errors
grep -i "error" build.log
grep -i "warning" build.log

# Test failures
# Run specific failing test
npm test -- --grep "failing test pattern"
# Check test reports
cat test-results.xml

# Deployment failure
# Check deployment logs
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous
kubectl rollout status deployment/myapp

# Artifact corruption
# Verify artifact checksum
sha256sum artifact.tar.gz
# Compare with expected hash
```

### 13.2 Pipeline Debugging Script

```bash
#!/bin/bash
# debug-pipeline.sh

STAGE=${1:-build}
BUILD_ID=${2:-latest}

echo "=== Pipeline Debug: Stage=$STAGE, Build=$BUILD_ID ==="

case $STAGE in
  build)
    echo "Checking build logs..."
    ls -la build/
    tail -100 build/build.log
    ;;
  test)
    echo "Checking test results..."
    ls -la test-results/
    cat test-results/failures.txt 2>/dev/null || echo "No failures"
    ;;
  deploy)
    echo "Checking deployment status..."
    kubectl get pods --all-namespaces | grep -E "(Error|CrashLoopBackOff|ImagePullBackOff)"
    ;;
  *)
    echo "Unknown stage: $STAGE"
    exit 1
    ;;
esac
```

## 14. Comparison Section

### CI/CD Tools Comparison

| Feature | GitHub Actions | Azure DevOps | Jenkins | GitLab CI | CircleCI |
|---------|---------------|--------------|---------|-----------|----------|
| Hosting | Cloud + self-hosted | Cloud + self-hosted | Self-hosted | Cloud + self-hosted | Cloud |
| Pipeline as Code | YAML | YAML | Groovy/Declarative | YAML | YAML |
| Container Support | Native | Native | Plugin | Native | Native |
| Marketplace | Extensive | Extensions | Extensive | Limited | Orbs |
| Pricing | Free for public | Free for 5 users | Free | Free tier | Free tier |
| Scaling | Automatic | Automatic | Manual | Automatic | Automatic |
| Matrix Builds | Native | Native | Plugin | Native | Native |
| Secrets | Encrypted | Variable groups | Credentials binding | Masked | Encrypted |
| Multi-Platform | Linux/Mac/Windows | Linux/Mac/Windows | Linux/Mac/Windows | Linux/Mac/Windows | Linux/Mac |

### GitOps vs CI/CD

| Aspect | Traditional CI/CD | GitOps |
|--------|------------------|--------|
| Deployment trigger | Pipeline event | Git commit |
| State management | Pipeline state | Git desired state |
| Drift detection | Manual | Automatic (reconcile) |
| Rollback | Pipeline re-run | git revert |
| Audit trail | Pipeline logs | Git history |
| Tools | Jenkins, GitHub Actions | ArgoCD, Flux |
| Complexity | Moderate | Higher initial setup |

## 15. Revision Notes

### Quick Recap: Pipeline Concepts

```text
CI/CD = Speed + Quality + Risk Reduction

Core Principles:
1. Automate everything
2. Build once, deploy many
3. Fail fast, fail loudly
4. Everything as code
5. Immutable artifacts
6. Environment parity
7. Security built-in

Pipeline Types:
- CI: lint, test, build
- CD: deploy, verify, smoke test
- CI/CD: full pipeline
- GitOps: Git as single source of truth

Key Metrics:
- Deployment Frequency: How often
- Lead Time: Commit to production
- MTTR: Time to recover
- Change Failure Rate: % of failures
```

### Common Pipeline Patterns

```text
1. CI only: PR checks, lint, test, build
2. CI + CD: Full pipeline to production
3. Multi-stage: Dev -> QA -> Staging -> Prod
4. Selective: Path-filtered for monorepos
5. Matrix: Multi-version testing
6. Fan-out/Fan-in: Parallel stages with aggregation
7. Canary: Gradual traffic shifting
8. Blue-Green: Instant switch between environments
```

## 16. Cheat Sheet

```text
+======================================================================+
|                       CI/CD CHEAT SHEET                              |
+======================================================================+

  CORE CONCEPTS
+----------------------------------------------------------------------+
| CI = Continuous Integration   (merge + build + test frequently)      |
| CD = Continuous Delivery      (always deployable, manual prod push)  |
| CD = Continuous Deployment    (every commit to prod automatically)   |
| Pipeline = automated workflow from commit to deployment              |
+----------------------------------------------------------------------+

  PIPELINE STAGES
+----------------------------------------------------------------------+
|  1. Source      | Checkout code from repository                      |
|  2. Lint/Format | Static analysis, code style                        |
|  3. Unit Test   | Fast, isolated tests                               |
|  4. Build       | Compile, package artifact                          |
|  5. Integration | Test component interactions                        |
|  6. Security    | SAST, dependency scan, container scan              |
|  7. Deploy      | Push artifact to environment                       |
|  8. Smoke Test  | Verify deployment                                 |
|  9. E2E Test    | Full workflow tests                               |
+----------------------------------------------------------------------+

  BRANCH STRATEGIES
+----------------------------------------------------------------------+
|  Trunk-Based  | feature -> main (short branches, < 1 day)            |
|  GitHub Flow  | feature -> main -> deploy                            |
|  GitFlow      | feature -> develop -> release -> main                |
|  GitLab Flow  | feature -> main -> production/staging                |
+----------------------------------------------------------------------+

  DEPLOYMENT STRATEGIES
+----------------------------------------------------------------------+
|  Rolling      | Gradual replacement of instances                     |
|  Blue-Green   | Instant switch between two identical environments    |
|  Canary       | Route small % of traffic to new version              |
|  A/B Testing  | Route traffic based on user segments                 |
|  Feature Flag | Deploy code hidden behind toggle                     |
+----------------------------------------------------------------------+

  SECURITY BEST PRACTICES
+----------------------------------------------------------------------+
|  - Never hardcode secrets in YAML files                              |
|  - Use secret stores (Azure KV, AWS Secrets Manager, HashiCorp Vault)|
|  - Sign commits and container images                                 |
|  - Run SAST/DAST scans in pipeline                                   |
|  - Least privilege for service connections                           |
|  - Generate SBOM for every build                                     |
|  - Scan dependencies for known vulnerabilities                       |
+----------------------------------------------------------------------+

  COMMANDS CHEAT SHEET
+----------------------------------------------------------------------+
|  Show pipeline status   | az pipelines run --id 123                  |
|  Trigger pipeline       | az pipelines build queue                   |
|  Download artifact      | az pipelines runs artifact download        |
|  View logs              | az pipelines build logs                    |
|  Jenkins CLI            | java -jar jenkins-cli.jar build job        |
|  GitHub CLI             | gh run list / gh run watch                 |
|  GitLab CLI             | glab ci view / glab ci status              |
+----------------------------------------------------------------------+

+======================================================================+
|  PRO TIPS:                                                           |
|  - Build once, promote the same artifact through all environments    |
|  - Make pipelines fast (under 10 minutes) for rapid feedback         |
|  - Use infrastructure as code for consistent environments            |
|  - Implement automatic rollback on health check failure              |
|  - Monitor DORA metrics and improve continuously                     |
|  - Test your pipeline as you would test your code                    |
+======================================================================+
```
