# Jenkins

---

## Overview

- **Definition:** Jenkins is the leading open-source automation server for CI/CD, supporting build, test, and deploy pipelines through an extensive plugin ecosystem (1,800+ plugins) and Groovy-based pipeline definitions.
- **Why It Exists:** Jenkins provides maximum customization and control as a self-hosted solution. Pipelines defined as code (Jenkinsfile) enable version-controlled, reproducible CI/CD workflows across any technology stack.
- **Key Concepts:** **Master** (central server managing jobs, plugins, build queue), **Agent** (remote worker executing builds), **Executor** (build slot on agent), **Pipeline** (code-defined CI/CD in Groovy), **Plugin** (extension module), **Jenkinsfile** (pipeline definition stored in SCM)

---

## Core Concepts

- **Pipeline Types:** **Declarative** — structured syntax (`pipeline { agent any; stages { ... } }`), easier, enforced structure. **Scripted** — full Groovy DSL (`node { stage('Build') { ... } }`), maximum flexibility. **Multibranch** — auto-creates pipelines per branch. **Organization Folder** — auto-discovers repos from GitHub/Bitbucket.
- **Master/Agent Architecture:** Master handles scheduling, UI, and API. Agents execute jobs. Use agents for builds, master for scheduling only. Labels select agents by capability (`linux && docker`). Agents can be permanent VMs, containers, or ephemeral cloud instances.
- **Jenkinsfile in SCM:** Pipeline definition stored in repository root. Enables version control, code review, and branch-specific behavior. Use `checkout scm` to pull the same commit being built.
- **Shared Libraries:** Groovy code in separate repository (`vars/` for global functions, `src/` for classes). Imported via `@Library('my-lib@version') _`. Enables reusable pipeline components across teams.

```groovy
// Declarative pipeline with Docker agent
pipeline {
    agent {
        docker { image 'node:20-alpine' }
    }
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 1, unit: 'HOURS')
        timestamps()
    }
    stages {
        stage('Build') {
            steps { sh 'npm ci && npm run build' }
        }
        stage('Test') {
            steps {
                sh 'npm test'
            }
            post {
                always { junit 'reports/*.xml' }
            }
        }
        stage('Deploy') {
            when { branch 'main' }
            steps { sh 'deploy.sh' }
        }
    }
    post {
        success { emailext subject: "SUCCESS", to: 'team@co.com' }
        failure { emailext subject: "FAILED", to: 'team@co.com' }
        cleanup { cleanWs() }
    }
}
```

---

## Common Mistakes

- **Using freestyle jobs** — Not repeatable or versionable; use Pipeline jobs with Jenkinsfile in SCM
- **No build discard policy** — Disk eventually full; set `buildDiscarder(logRotator(...))`
- **Hardcoding credentials** — Security breach; use Credentials Binding plugin
- **Master doing builds** — Performance bottleneck; offload to agents
- **Too many plugins** — Upgrade hell and conflicts; only install needed plugins
- **No backup of JENKINS_HOME** — Data loss; automate regular backups
- **Scripts not in SCM** — No traceability; always use "Pipeline from SCM"
- **Single point of failure** — Downtime; implement HA with active/passive
- **No pipeline testing** — Broken pipelines; test shared libraries and pipeline logic
- **Using `any` agent for everything** — Resource contention; use labels to match capabilities

---

## Key Design Considerations

- **Pipeline as Code** — Always use Pipeline jobs with Jenkinsfile stored in SCM. Declarative preferred for simplicity. Scripted for complex scenarios. Multibranch for branch-based deployments.
- **Agent Strategy** — Use labels to match agent capabilities (`linux && docker`). Cloud agents (Kubernetes, EC2, Azure VMs) for elastic scaling. Permanent agents for latency-sensitive tasks. Master should only schedule.
- **Shared Libraries** — Centralize deployment logic, security scanning, notifications. Version libraries with Git tags/branches. Test libraries in isolation. Document API for pipeline authors.
- **Security** — Credentials Binding plugin (never plain text), run Jenkins as non-root, RBAC for access control, CSRF protection, Agent-to-Master security restrictions, Pipeline Groovy sandbox for untrusted SCMs, regular plugin updates
- **Plugin Management** — Maintain plugin manifest with versions. Use Configuration as Code plugin for declarative setup. Test plugin upgrades in staging. Consider compatibility before core upgrades. Backup JENKINS_HOME before changes.
- **High Availability** — Active/passive with shared NFS for JENKINS_HOME. Active/active with externalized build metadata. Cloud-native HA with Kubernetes operators. Regular DR drills.

---

## Real-World Scenarios

**Scenario 1: Jenkins HA with Active-Passive and Shared NFS**
A team's Jenkins master goes down every 3 months causing 6+ hours of downtime. Implement active-passive HA: primary Jenkins master with shared JENKINS_HOME on NFS. Secondary master in standby. Keepalived provides virtual IP for automatic failover. PostgreSQL as external database for build records. Blue-green plugin upgrade process: upgrade standby, test, promote to active.

```groovy
// Groovy script for HA health check
pipeline {
    agent any
    stages {
        stage('Check HA Status') {
            steps {
                script {
                    def masterStatus = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:8080/login", returnStdout: true).trim()
                    if (masterStatus != '200') {
                        error 'Primary master unhealthy, triggering failover'
                    }
                }
            }
        }
        stage('Validate NFS Mount') {
            steps {
                sh 'ls -la /var/jenkins_home/jobs/ | head -5'
            }
        }
    }
}
```

**Scenario 2: Shared Library for 50+ Microservice Teams**
A platform team supports 50+ microservice teams with CI/CD. Create a shared library repository with `vars/` for reusable pipeline steps (`buildGoApp.groovy`, `deployToK8s.groovy`, `securityScan.groovy`) and `src/` for utility classes. Each team imports with `@Library('platform-pipelines@v2')_`. Teams only define service-specific config.

```groovy
// vars/buildGoApp.groovy
def call(String serviceName, GoVersion = '1.21') {
    podTemplate(containers: [containerTemplate(name: 'go', image: "golang:${GoVersion}")]) {
        node(POD_LABEL) {
            stage("Build ${serviceName}") {
                checkout scm
                container('go') {
                    sh "cd services/${serviceName} && go build -o bin/app ."
                }
            }
        }
    }
}
```

**Scenario 3: Migrating 500 Freestyle Jobs to Declarative Pipelines**
An enterprise has 500+ freestyle jobs with 4 years of technical debt. Create a phased migration plan. Phase 1: Convert build logic to shared library. Phase 2: Create template pipelines. Phase 3: Migrate team by team. Phase 4: Enforce Pipeline from SCM for all new jobs. Use Job DSL plugin to auto-generate pipeline jobs from YAML config.

---

## Scenario-Based Questions

1. **Q: Design a shared library for 50+ microservice teams.**
   A: Repository with `vars/` for reusable pipeline steps (build, deploy, scan), `src/` for utility classes. Versioned with Git tags. Imported via `@Library('my-lib@v2.1') _`. Comprehensive documentation and unit tests. Change log and deprecation policy.

2. **Q: How would you implement rolling deployment with health checks in Jenkins?**
   A: Pipeline deploys instances one by one: remove from load balancer, deploy new version, run health check, add back to LB. Use `kubectl rollout` or custom script. Auto-rollback if health check fails. Configurable parallelism.

3. **Q: Design Jenkins HA with zero downtime maintenance.**
   A: Active/passive pair with shared JENKINS_HOME on NFS. Keepalived for IP failover. Blue-green Jenkins (two masters, only one active). Load balancer health checks. Stagger plugin upgrades. Test failover regularly.

4. **Q: How would you secure Jenkins in a multi-tenant environment?**
   A: RBAC with folder-based authorization (team-specific folders). Pipeline Groovy sandbox for untrusted projects. Agent-to-Master security (disable executors on master). Credential scoping per folder. Audit trails. Network isolation.

5. **Q: Design a canary release pipeline with auto-rollback in Jenkins.**
   A: Deploy canary (10% instances). Run smoke tests. Monitor metrics via API (Datadog, Prometheus) for 5 min. If error rate increases, rollback. If healthy, ramp to 50%, then 100%. All automated with conditional stages and timeout.

6. **Q: How would you migrate from Jenkins to GitHub Actions without downtime?**
   A: Phase 1: Audit all pipelines. Phase 2: Port simple workflows to Actions. Phase 3: Run both systems in parallel. Phase 4: Redirect all new builds to Actions. Phase 5: Sunset Jenkins. Use webhook interceptors during transition.

7. **Q: How do you implement pipeline testing in Jenkins?**
   A: Use Jenkins Pipeline Unit Testing Framework. Test shared library code with JUnit. Mock `sh`, `readFile`, `withCredentials` steps. Run pipeline tests in CI. Use Supporting APIs plugin for local validation.

8. **Q: Design a backup and DR strategy for Jenkins.**
   A: Back up JENKINS_HOME daily (jobs, config, plugins, secrets). Use thinBackup plugin for granular backup. Store backups in S3/Blob Storage. DR: restore JENKINS_HOME to new instance, install same plugins, validate jobs. Test DR monthly.

9. **Q: How do you handle secrets rotation across 500+ pipelines?**
   A: Use Jenkins Credentials Provider API. Central credential store. Create new credential version, update pipelines to use label. Deprecate old credential, remove after all pipelines updated. Automate with scripted pipeline and REST API.

10. **Q: How would you design a multi-region Jenkins for global teams?**
    A: Master in each region, connected via shared database or event bus. Agents in same region for reduced latency. Regional build caches. S3/Blob backed artifact storage. Global load balancer for web UI. Git-based pipeline definitions synced across regions.


---

## Interview Questions

1. **What is the difference between Declarative and Scripted Pipeline?**
   A: Declarative has a structured syntax (`pipeline { agent any; stages { stage('Build') { steps { sh '...' } } } }`), easier, enforced structure, and is the recommended approach. Scripted uses full Groovy DSL (`node { stage('Build') { sh '...' } }`), maximum flexibility, but harder to maintain. Prefer Declarative for 90% of use cases.

2. **What is a Jenkins shared library and when should you use one?**
   A: A shared library is Groovy code stored in a separate Git repository with `vars/` (global functions) and `src/` (utility classes). Use when: (1) You have multiple teams with similar pipelines. (2) You need to enforce standard deployment patterns. (3) Complex pipeline logic that shouldn't be duplicated. Imported via `@Library('my-lib@v2') _`.

3. **What is the difference between a freestyle job and a pipeline job?**
   A: Freestyle jobs are configured through the Jenkins UI (point and click), not versionable, not repeatable, no pipeline-as-code. Pipeline jobs use a Jenkinsfile stored in SCM (Groovy), version-controlled, code-reviewed, and testable. Always use Pipeline jobs.

4. **What is the master/agent architecture in Jenkins?**
   A: Master (controller) handles scheduling, UI, API, job configuration. Agents (workers) execute builds. Master should not run builds (set executors to 0). Agents can be permanent VMs, Docker containers, or ephemeral cloud instances. Labels select agents by capability. Benefits: security isolation, horizontal scaling.

5. **What is a Multibranch Pipeline?**
   A: A Pipeline job that automatically discovers branches from Git, creates a pipeline per branch, and builds them. Uses Jenkinsfile from each branch for branch-specific behavior. Auto-cleans up pipeline for deleted branches. Supports branch filters to limit scanning.

6. **How does Jenkins integrate with Kubernetes?**
   A: Kubernetes Plugin allows Jenkins agents to run as Kubernetes Pods. Each build gets a Pod with one or more containers (tools). PodTemplate defines the container images, volumes, and resource limits. Auto-scales: idle Pods are destroyed. Benefits: ephemeral agents, resource efficient, no permanent agent management.

7. **What is the Configuration as Code plugin (JCasC)?**
   A: Defines Jenkins configuration in a YAML file (`jenkins.yaml`). Configures plugins, security realms, authorization strategies, credentials, agent configurations, job definitions. Changes are applied at startup or via reload. Enables reproducible Jenkins setups, version-controlled config, and quick disaster recovery.

8. **What is the Pipeline Unit Testing Framework?**
   A: A testing framework for testing Pipeline Shared Libraries. Mocks `sh`, `readFile`, `withCredentials`, and other pipeline DSL methods. Uses JUnit for assertions. Run tests as part of CI/CD for the shared library. Ensures pipeline logic changes don't break consumers.

9. **How do you secure Jenkins credentials?**
   A: Use the Credentials Binding plugin — credentials stored encrypted in Jenkins, bound to environment variables in the build. Use scoped credentials per folder. Never use `withCredentials` with plain text. Use Managed Files for config files with credentials. Integrate with HashiCorp Vault for external credential management.

10. **What is Blue Ocean in Jenkins?**
   A: A modern Jenkins UI plugin that provides a visual pipeline editor, real-time pipeline visualization, and improved user experience. Shows pipeline stages, logs, and test results in a clean interface. Available since Jenkins 2.x. Deprecated but still widely used. Modern alternative: the standard Jenkins UI with Pipeline Stage View.

---

## Developer Recommendations

- **Always use Pipeline from SCM, never freestyle jobs** — Freestyle jobs are unrepeatable, unversioned, and manually configured. A Jenkinsfile in Git is peer-reviewed, versioned, branch-aware, and auditable. Migration from freestyle to pipeline is the single highest-impact improvement you can make. Start with a shared library, then convert jobs incrementally.

- **Use shared libraries, but keep them small and tested** — Shared libraries enable standardization but grow complex quickly. Each `vars/*.groovy` should do one thing. Version libraries with Git tags. Unit test library code with the Pipeline Unit Testing Framework. Document every function with usage examples. Deprecation policy: tag old versions, communicate breaking changes, provide migration scripts.

- **Never run builds on the Jenkins master** — Setting master executors to 0 is non-negotiable. Builds on master consume JVM heap, impact API responsiveness, and if a build goes rogue (fork bomb, disk full) it takes down the entire Jenkins instance. Use agents for everything. For very small setups (1-2 users), use a separate VM as an agent.

- **Maintain a plugin manifest and test upgrades** — Jenkins plugins are both the platform's strength and its biggest risk. Pin plugin versions in Configuration as Code. Test plugin upgrades in a staging Jenkins before production. Remove unused plugins (each plugin consumes memory and adds startup time). Run `jenkins-plugin-manager` to check for compatibility before upgrades.

- **Backup JENKINS_HOME daily, test DR monthly** — JENKINS_HOME contains all job configs, plugin configs, build records, and credentials. Back it up daily to S3/Blob Storage (at least 30-day retention). Use the thinBackup plugin for granular backups. Test DR: restore backup to a fresh Jenkins instance, verify all jobs load, credentials are available, and pipelines run. Simulating a full Jenkins loss is the only way to know your DR works.

- **Use ephemeral agents for elasticity and consistency** — Permanent agents accumulate state (disk usage, leftover processes, package drift). Kubernetes Pod templates or EC2/VMSS agents give you a fresh environment per build. The PodTemplate YAML (image, tools, env) defines the exact build environment — no drift, no conflicts, no cleanup needed. Scale to zero when idle saves costs.