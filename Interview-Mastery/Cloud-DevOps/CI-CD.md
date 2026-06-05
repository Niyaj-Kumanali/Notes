# CI/CD

---

## Overview

- **Definition:** CI/CD (Continuous Integration and Continuous Delivery/Deployment) automates the software delivery process — building, testing, and deploying code changes frequently and reliably.
- **Why It Exists:** CI/CD shortens feedback loops from weeks to minutes, catches integration issues early, reduces manual errors, enables faster releases, and improves software quality by eliminating infrequent, risky manual deployments.
- **Key Concepts:** **Continuous Integration** (automatically build and test every commit), **Continuous Delivery** (always deployable state, manual production push), **Continuous Deployment** (fully automated to production), **Pipeline as Code** (YAML, Groovy, HCL), **Build Once Deploy Many** (promote immutable artifacts through environments)

---

## Core Pipeline Stages

- **Source / Commit** — Trigger pipeline on code push or PR. Use branch/path filters to run only for relevant changes. Use trunk-based or GitHub Flow for CI/CD.
- **Lint & Static Analysis** — Run linters, formatters, and SAST tools. Fail fast on code quality issues before slower tests run.
- **Unit Tests** — Fast, isolated tests validating individual components. Must pass on every commit. Aim for 70%+ coverage.
- **Build** — Compile code and produce immutable artifacts (Docker images, binaries, packages). Version using semantic versioning + commit SHA.
- **Integration Tests** — Test component interactions with real dependencies (databases, APIs). Run on every push to main.
- **Security Scan** — SAST, dependency scanning (npm audit, OWASP), container scanning (Trivy), SBOM generation for supply chain transparency.
- **Deploy to Staging** — Deploy to pre-production, run smoke tests to verify deployment succeeded, then run E2E tests.
- **Deploy to Production** — Manual approval for CD, automatic with gates for full CD. Use canary/blue-green/rolling strategies. Auto-rollback on health check failure.

```yaml
trigger:
  branches: { include: [main, develop] }
  paths: { exclude: [docs/*, README.md] }

stages:
  - stage: CI
    jobs:
      - job: Lint
      - job: Test
      - job: Build
  - stage: SecurityScan
    dependsOn: CI
  - stage: Deploy_Staging
    dependsOn: SecurityScan
  - stage: Deploy_Prod
    dependsOn: Deploy_Staging
    condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')
    trigger: manual
```

---

## Common Mistakes

- **Building artifacts in multiple stages** — Inconsistency; build once, promote same artifact through all environments
- **Hardcoded secrets in pipeline YAML** — Security breach; use secret variables or vault integration
- **No database changes in pipeline** — Migration failures; include DB migrations in pipeline
- **Building on every branch without filter** — Wasted resources; use path/branch triggers
- **Ignoring flaky tests** — Unreliable pipeline; quarantine and fix flaky tests
- **Long-running pipelines** — Slow feedback; parallelize, optimize, incremental builds
- **No artifact versioning** — Unable to trace releases; use semantic versioning with commit SHA
- **Manual deployment steps** — Human error; automate everything end-to-end
- **No rollback strategy** — Extended outages; implement automatic rollback on health failure
- **Environment drift** — Surprises in production; use IaC for identical environments

---

## Key Design Considerations

- **Build Once, Deploy Many** — Produce immutable, versioned artifacts in CI. Promote the exact same artifact through all environments. Never rebuild for deployment.
- **Pipeline as Code** — Define pipelines in version control (YAML, Groovy). Enables code review, audit trail, and reproducibility. Store in repo root.
- **Branch Strategy** — Trunk-Based Development (merge multiple times daily), GitHub Flow (feature to main to deploy), GitFlow (feature to develop to release to main). Choose based on release cadence and team size.
- **Deployment Strategies** — Rolling (gradual instance replacement), Blue-Green (instant environment switch), Canary (small traffic percentage), Feature Flags (deploy hidden behind toggle). Trade-offs between speed, risk, and complexity.
- **DORA Metrics** — Deployment Frequency (how often), Lead Time for Changes (commit to production), Mean Time to Recover (recovery speed), Change Failure Rate (failure percentage). Track to measure pipeline effectiveness.
- **Pipeline Security** — Least privilege for CI/CD credentials, never hardcode secrets, sign commits and images, generate SBOM, scan dependencies, restrict service connections per environment

---

## Real-World Scenarios

**Scenario 1: Enterprise CI/CD Migration from Jenkins to GitHub Actions**
A company with 200 Jenkins pipelines needs to migrate to GitHub Actions. Strategy: Audit all pipelines and categorize by complexity. Port simple build-only pipelines first. Create reusable composite actions for common patterns. Run both systems in parallel for 3 months. Use feature parity checklists. Migrate deployment pipelines last. Use GitHub Actions Importer tool for automated migration. Implement OIDC for AWS/Azure auth from day one.

```yaml
# GitHub Actions workflow replacing Jenkins pipeline
name: CI/CD Pipeline
on:
  push:
    branches: [main]
    paths-ignore: ['docs/**', 'README.md']
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
  security:
    needs: build
    steps:
      - uses: aquasecurity/trivy-action@master
        with:
          scan-type: fs
          severity: HIGH,CRITICAL
  deploy:
    needs: security
    if: success()
    steps:
      - run: echo "Deploying with OIDC"
```

**Scenario 2: Zero-Downtime Database Migration Pipeline**
A SaaS company needs to deploy database schema changes without downtime. Use Flyway with version-controlled migrations. Each migration is backward compatible. Pipeline stages: pre-deployment migration (add columns, new tables), deploy app (reads old + new schema), post-deployment migration (remove deprecated columns). Rollback: reverse migration script automatically applied if health checks fail.

```yaml
# expand-contract migration pattern
stages:
  - stage: Expand
    jobs:
      - job: AddNewColumns
        steps: - script: flyway migrate -target=expand_version
  - stage: Deploy
    jobs:
      - job: DeployApp
        steps: - script: helm upgrade app ./chart
  - stage: Contract
    dependsOn: Deploy
    jobs:
      - job: RemoveOldColumns
        condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
        steps: - script: flyway migrate -target=contract_version
```

**Scenario 3: Multi-Cloud CI/CD with Compliance Gates**
A regulated company deploys to both AWS and Azure with PCI-DSS compliance. Use Terraform for multi-cloud IaC. Single CI pipeline produces platform-agnostic artifacts. Two CD pipelines (AWS/Azure) consume the same artifact. Compliance gates: security scan (Trivy < CRITICAL), dependency check (no known CVEs), SBOM generation, and manual approval for production.

---

## Scenario-Based Questions

1. **Q: Design a CI/CD system for a monorepo with 100+ microservices.**
   A: Use path triggers to only build/test changed services. Cache dependencies. Matrix builds for multi-version testing. Fan-out/fan-in pattern with parallel execution. Artifact per service with semantic versioning.

2. **Q: How would you implement progressive delivery (canary, feature flags) in CI/CD?**
   A: Canary — deploy new version to small traffic percentage, monitor metrics, gradually increase. Feature flags — deploy code hidden behind toggle, enable for specific users, ramp up. Auto-rollback on metric degradation.

3. **Q: Design a zero-downtime deployment strategy for a stateful application.**
   A: Blue-green deployment with database migration in prepare phase. Run backward-compatible migrations first. Keep old version running until all connections drain. Use read-only mode during cutover. Health checks verify new version before traffic switch.

4. **Q: How do you ensure compliance (SOC2, HIPAA) in a CI/CD pipeline?**
   A: Enforce signed commits, pipeline must pass all security scans, deployment requires compliance approval, immutable audit trail of all deployments, artifact signing and verification, SBOM generation per build.

5. **Q: Design a pipeline that supports multi-cloud deployment (AWS, Azure, GCP).**
   A: Abstract cloud-specific steps into reusable pipeline templates. Use Terraform for IaC. Parameterize cloud provider credentials. Environment-specific variables per cloud. Multi-cloud testing matrix. Unified artifact registry.

6. **Q: How do you handle database schema changes during deployment?**
   A: Use Flyway/Liquibase for version-controlled migrations. Expand-contract pattern: add new schema alongside old, migrate data, then remove old. Always include rollback scripts. Run migrations before app deployment. Automate in pipeline.

7. **Q: Design a CI/CD pipeline for ML models (MLOps).**
   A: Stages: data validation, model training, model evaluation (compare metrics), model registry, model deployment to staging, A/B testing in production, monitoring for drift. Use DVC for data versioning, MLflow for experiment tracking.

8. **Q: How do you implement artifact promotion with security gates?**
   A: Each environment requires the artifact to pass that environment's gates: vulnerability scan < critical, unit tests > 80% coverage, integration tests pass, approval from security team. Artifacts are immutable and cryptographically signed.

9. **Q: Describe the testing pyramid in CI/CD.**
   A: Many unit tests (fast, isolated, 70%+), some integration tests (component interaction), few E2E tests (full workflow, slow but high confidence). CI runs unit + integration; E2E runs against staging after deploy.

10. **Q: How do you handle versioning in a microservices CI/CD pipeline?**
    A: Each service independently versioned using semantic versioning (MAJOR.MINOR.PATCH+BUILD). Git tag triggers release pipeline. Artifact stored with version tag in registry. Service dependency managed via API contracts and consumer-driven contract testing.


---

## Interview Questions

1. **What is the difference between Continuous Integration and Continuous Delivery?**
   A: CI automatically builds and tests every commit (catch integration issues early). CD automatically deploys every commit that passes CI to a staging environment, with manual approval for production. Continuous Deployment goes further — fully automated to production with no human intervention.

2. **What is the testing pyramid in CI/CD?**
   A: Base: many unit tests (fast, isolated, 70%+ coverage). Middle: integration tests (component interaction with real dependencies). Top: few E2E tests (full workflow, slow). CI runs unit + integration; E2E runs against staging post-deploy. Avoid the ice cream cone (too many E2E, few unit tests).

3. **What is Build Once Deploy Many?**
   A: The practice of producing a single immutable artifact in CI, then promoting that exact artifact through all environments. Never rebuild for deployment — the artifact tested in staging is the same one deployed to production. Eliminates environment-specific rebuild issues.

4. **What is artifact immutability and why does it matter?**
   A: An artifact, once created and versioned, is never modified or overwritten. Each version is uniquely identified (semantic version + commit SHA). Enables audit trail, reproducibility, and safe rollback (deploy previous version). Prevents the "same build" producing different results.

5. **What is the difference between a canary and blue-green deployment?**
   A: Blue-green: two full environments, switch traffic instantly (DNS/router change). Instant rollback (flip back). Resource intensive (2x environment). Canary: new version receives small traffic percentage, gradually increased. Slower, more granular, but resource efficient. Choose blue-green for critical apps needing instant rollback, canary for gradual risk reduction.

6. **What is CI/CD pipeline as code?**
   A: Defining the pipeline configuration in a version-controlled file (Jenkinsfile, .github/workflows/*.yml, .gitlab-ci.yml) alongside application code. Enables code review, versioning, branching, and audit trail for pipeline changes. Replaces manually configured CI jobs.

7. **What are DORA metrics?**
   A: Four key DevOps metrics: Deployment Frequency (how often), Lead Time for Changes (commit to production), Mean Time to Recover (recovery speed), Change Failure Rate (failure percentage). Elite performers deploy multiple times/day, lead time < 1 hour, MTTR < 1 hour, failure rate < 5%.

8. **How do you handle secrets in a CI/CD pipeline?**
   A: Never hardcode secrets. Use built-in secret management (GitHub Secrets, GitLab CI Variables, Azure DevOps Variable Groups). Use Vault integration (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault). Use OIDC for cloud provider auth (no static secrets). Rotate secrets automatically with tooling.

9. **What is trunk-based development and how does it relate to CI/CD?**
   A: Developers merge to main branch multiple times daily. Short-lived feature branches (hours, not days). No long-lived release branches. Enables continuous integration (every merge is built and tested). Reduces merge conflicts and integration hell. Essential for Continuous Deployment.

10. **What is the role of a package/artifact registry in CI/CD?**
   A: Central storage for immutable, versioned build artifacts (Docker images, npm packages, JAR files, binaries). Supports Build Once Deploy Many. Enables artifact promotion (promote between registries/environments). Integrates with security scanners. Example: Docker Hub, GitHub Packages, Artifactory, Azure Artifacts, ECR.

---

## Developer Recommendations

- **Build Once, Deploy Many — never rebuild for a different environment** — Rebuilding for each environment guarantees inconsistency. The artifact tested in staging must be byte-for-byte identical to production. Use versioned artifacts in a registry. Each environment promotes the same artifact. Environment config is injected at deploy time, not build time. Without this, you cannot trust your CI/CD.

- **Fail fast with a proper testing strategy** — Run fast unit tests first (seconds), then integration tests (minutes), then security scans, then deploy. Don't put expensive E2E tests in the CI path — run them post-deploy against staging. Use test impact analysis to run only tests relevant to changed code. Every minute of CI time that can be saved is developer throughput regained.

- **Use feature flags to decouple deployment from release** — Feature flags allow deploying code that isn't yet active. Deploy safely behind a toggle, enable for internal testing, gradually roll out to users, instantly disable if issues arise. No rollback needed — just toggle off. Tools: LaunchDarkly, Flagsmith, Unleash. Trade-off: flag management complexity and tech debt from untoggled flags.

- **Automate everything — including rollback** — Manual rollbacks are slow and error-prone (what if you forget a step?). Implement automatic rollback triggered by health check failures. Every deployment should have a tested rollback script. The rollback should also be a pipeline run (with audit trail). If you need to SSH into production to fix a deployment, your CI/CD is incomplete.

- **Monitor pipeline health with DORA metrics** — Track Deployment Frequency, Lead Time, MTTR, and Change Failure Rate. If lead time is increasing, your pipeline has bottlenecks. If failure rate is high, your testing is inadequate. If MTTR is high, your rollback automation is weak. Set improvement targets and review monthly. Elite DevOps teams ship 208x more frequently with 7x lower change failure rate.

- **Secure the pipeline itself — not just the app it deploys** — The CI/CD pipeline has access to production. If compromised, the attacker owns production. Use: signed commits, OIDC for cloud auth (no static secrets), minimal permissions per job, environment protection rules, dependency review, artifact signing, and hermetic builds. Treat the pipeline as the most critical security boundary.