# Jenkins Study Guide

## 1. Executive Summary

Jenkins is the leading open-source automation server for CI/CD. It supports building, testing, and deploying software through an extensive plugin ecosystem (1,800+ plugins) and flexible pipeline definitions using Groovy DSL. As a self-hosted solution, Jenkins offers maximum customization and control but requires significant maintenance. Pipelines can be defined as code (Jenkinsfile) using declarative or scripted syntax, enabling version-controlled, reproducible CI/CD workflows.

## 2. Core Theory

### 2.1 Jenkins Architecture

```
+-------------------+       +-------------------+
|   Jenkins Master  |       |   Build Agent     |
|  (Web UI + API)   | <---> |   (Node)          |
|  Scheduler + UI   |       |  Executors        |
+-------------------+       +-------------------+
         |
   Plugin System (1800+)
         |
   Build Queue / Jobs
```

### 2.2 Master/Agent Architecture

- **Master**: Central server managing jobs, plugins, and build queue
- **Agent**: Remote worker that executes build jobs
- **Executor**: A slot for running build jobs on an agent
- **Node**: Any machine that can run Jenkins jobs (master or agent)

### 2.3 Pipeline Types

| Type | Definition | Pros | Cons |
|------|-----------|------|------|
| Declarative | Structured YAML-like | Simple, enforced structure | Less flexible |
| Scripted | Full Groovy DSL | Maximum flexibility | Complex, steep learning |
| Multibranch | Auto-creates pipelines per branch | Branch management | Complex config |
| Organization Folder | Auto-discovers repos | Organizational scale | Complex setup |

## 3. Under-the-Hood Deep Dive

### 3.1 Pipeline Structure

```groovy
// Declarative Pipeline
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Building...'
            }
        }
    }
    post {
        always { cleanWs() }
    }
}

// Scripted Pipeline
node {
    stage('Build') {
        echo 'Building...'
    }
}
```

### 3.2 Job Types

- **Freestyle job**: Simple, UI-configured (legacy, avoid)
- **Pipeline**: Code-defined (recommended)
- **Multibranch Pipeline**: Auto-creates per branch
- **Folder**: Organize jobs
- **Organization Folder**: Auto-discovers repositories from GitHub/Bitbucket

### 3.3 Jenkins Home Directory Structure

```
JENKINS_HOME/
  +-- jobs/          # Job configurations and build records
  +-- plugins/       # Installed plugins (.jpi files)
  +-- nodes/         # Agent configurations
  +-- secrets/       # Credentials and encryption keys
  +-- users/         # User configurations
  +-- workspace/     # Build workspaces
  +-- config.xml     # Main Jenkins configuration
  +-- hudson.model.*.xml  # Plugin configurations
```

## 4. Production Code Examples

### 4.1 Complete Declarative Pipeline

```groovy
pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '--network host'
        }
    }

    triggers {
        cron('H */4 * * *')
        pollSCM('H/5 * * * *')
    }

    parameters {
        string(name: 'VERSION', defaultValue: '', description: 'Release version')
        booleanParam(name: 'DEPLOY_TO_PROD', defaultValue: false)
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'])
    }

    environment {
        REGISTRY = 'registry.mycompany.com'
        IMAGE_NAME = "${REGISTRY}/myapp:${BUILD_NUMBER}"
        DOCKER_CREDS = credentials('docker-hub-credentials')
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 1, unit: 'HOURS')
        retry(2)
        timestamps()
        ansiColor('xterm')
        disableConcurrentBuilds()
        skipDefaultCheckout()
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Install Dependencies') {
            parallel {
                stage('npm') {
                    steps { sh 'npm ci' }
                }
                stage('Go') {
                    agent { docker { image 'golang:1.21' } }
                    steps { sh 'go mod download' }
                }
            }
        }

        stage('Quality Gate') {
            parallel {
                stage('Lint') {
                    steps { sh 'npm run lint' }
                }
                stage('Security Scan') {
                    steps {
                        sh 'npm audit --audit-level=high'
                        dependencyCheck additionalArguments: '--format SARIF'
                    }
                    post {
                        always { dependencyCheckPublisher() }
                    }
                }
            }
        }

        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps { sh 'npm run test:unit -- --coverage' }
                    post {
                        always {
                            junit 'reports/unit/*.xml'
                            jacoco()
                        }
                    }
                }
                stage('Integration Tests') {
                    steps { sh 'npm run test:integration' }
                    post { always { junit 'reports/integration/*.xml' } }
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    docker.build("${IMAGE_NAME}")
                }
            }
        }

        stage('Deploy to Staging') {
            when { branch 'main' }
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY}", 'docker-credentials') {
                        docker.image("${IMAGE_NAME}").push()
                    }
                }
                sh "kubectl --kubeconfig=${KUBE_CONFIG_STAGING} set image deployment/myapp myapp=${IMAGE_NAME}"
                sh "kubectl --kubeconfig=${KUBE_CONFIG_STAGING} rollout status deployment/myapp --timeout=300s"
            }
        }

        stage('E2E Tests') {
            when { branch 'main' }
            steps {
                sh "npm run test:e2e -- --base-url=https://staging.myapp.com"
            }
        }

        stage('Deploy to Production') {
            when { expression { params.DEPLOY_TO_PROD } }
            input {
                message "Deploy ${IMAGE_NAME} to production?"
                ok "Deploy"
                submitterParameter 'APPROVER'
            }
            steps {
                script {
                    sh """
                        kubectl --kubeconfig=${KUBE_CONFIG_PROD} set image deployment/myapp myapp=${IMAGE_NAME}
                        kubectl --kubeconfig=${KUBE_CONFIG_PROD} rollout status deployment/myapp --timeout=300s
                    """
                }
            }
        }
    }

    post {
        success {
            emailext(
                subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                to: 'team@mycompany.com',
                body: "Pipeline succeeded. ${env.BUILD_URL}"
            )
        }
        failure {
            emailext(
                subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                to: 'team@mycompany.com',
                body: "Pipeline failed at ${env.STAGE_NAME}. ${env.BUILD_URL}"
            )
        }
        cleanup { cleanWs() }
    }
}
```

### 4.2 Shared Library

```groovy
// vars/deploy.groovy
def call(String environment, String imageName) {
    def config = readYaml file: "deploy-config/${environment}.yaml"
    echo "Deploying ${imageName} to ${environment}"
    withKubeConfig(caCertificate: config.caCert, serverUrl: config.serverUrl) {
        sh "kubectl set image deployment/${config.appName} ${config.containerName}=${imageName}"
        sh "kubectl rollout status deployment/${config.appName} --timeout=${config.timeout}"
    }
}

// src/com/mycompany/PipelineUtils.groovy
package com.mycompany
class PipelineUtils implements Serializable {
    static String getVersion() {
        return env.BRANCH_NAME == 'main' ? '1.0.' + env.BUILD_NUMBER : env.BRANCH_NAME + '-' + env.BUILD_NUMBER
    }
}
```

### 4.3 Shared Library Usage

```groovy
@Library('my-shared-library@main') _

pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps {
                deploy('staging', 'myapp:latest')
                echo "Version: ${new com.mycompany.PipelineUtils().getVersion()}"
            }
        }
    }
}
```

### 4.4 Multibranch Pipeline

```groovy
pipeline {
    agent any
    triggers { pollSCM('H/5 * * * *') }

    stages {
        stage('Build') {
            steps { sh 'npm ci && npm run build' }
        }
        stage('Deploy') {
            when { anyOf { branch 'main'; branch 'release/*' } }
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        sh 'deploy production'
                    } else {
                        sh 'deploy staging'
                    }
                }
            }
        }
    }
}
```

### 4.5 Docker Agent Pipeline

```groovy
pipeline {
    agent {
        dockerfile {
            filename 'Dockerfile.build'
            dir 'build'
            args '-v /tmp:/tmp --network host'
            reuseNode true
            label 'docker-node'
        }
    }
    stages {
        stage('Build') {
            steps { sh 'make build' }
        }
    }
}
```

## 5. Real-World Scenarios

### 5.1 Jenkins HA Setup

```yaml
# docker-compose for HA Jenkins
version: '3.8'
services:
  jenkins-master:
    image: jenkins/jenkins:lts
    volumes:
      - jenkins-home:/var/jenkins_home
    environment:
      - JENKINS_OPTS=--httpPort=8080
    deploy:
      replicas: 2
    networks:
      - jenkins-net

  jenkins-agent:
    image: jenkins/inbound-agent:latest
    environment:
      - JENKINS_URL=http://jenkins-master:8080
      - JENKINS_SECRET=<secret>
      - JENKINS_AGENT_NAME=docker-agent
    depends_on: [jenkins-master]
    deploy:
      replicas: 5
    networks:
      - jenkins-net

  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
    ports: ["8080:8080"]
    depends_on: [jenkins-master]
    networks:
      - jenkins-net

volumes:
  jenkins-home:
    driver: nfs
```

### 5.2 Rolling Deployment Pipeline

```groovy
pipeline {
    agent any
    parameters {
        choice(name: 'STRATEGY', choices: ['rolling', 'blue-green', 'canary'])
    }
    stages {
        stage('Rolling') {
            when { expression { params.STRATEGY == 'rolling' } }
            steps {
                script {
                    def replicas = sh(
                        script: 'kubectl get deployment myapp -o=jsonpath="{.spec.replicas}"',
                        returnStdout: true
                    ).trim().toInteger()
                    for (int i = 1; i <= replicas; i++) {
                        sh "kubectl set image deployment/myapp myapp=${IMAGE_NAME}"
                        sh "kubectl rollout status deployment/myapp --timeout=120s"
                    }
                }
            }
        }
        stage('Blue-Green') {
            when { expression { params.STRATEGY == 'blue-green' } }
            steps {
                script {
                    sh 'kubectl apply -f k8s/deployment-green.yaml'
                    sh 'sleep 30'
                    sh 'kubectl apply -f k8s/service-green.yaml'
                    sh 'kubectl delete -f k8s/service-blue.yaml'
                    sh 'kubectl delete -f k8s/deployment-blue.yaml'
                }
            }
        }
    }
}
```

## 6. Performance

### 6.1 Optimization

```groovy
// Discard old builds
options {
    buildDiscarder(logRotator(
        numToKeepStr: '20',
        daysToKeepStr: '30',
        artifactNumToKeepStr: '5'
    ))
    timestamps()
}

// Fast fail pipeline
pipeline {
    agent any
    stages {
        stage('Fast Fail') {
            steps {
                sh '''
                    set -e
                    npm run lint
                    npm run test:quick
                '''
            }
        }
        stage('Slow Tests') {
            steps { sh 'npm run test:slow' }
        }
    }
}

// Per-stage agents
pipeline {
    agent none
    stages {
        stage('Linux Build') {
            agent { label 'linux && docker' }
            steps { sh 'make build' }
        }
        stage('Windows Build') {
            agent { label 'windows' }
            steps { bat 'msbuild.exe' }
        }
    }
}
```

## 7. Security

### 7.1 Credential Management

```groovy
withCredentials([
    string(credentialsId: 'api-key', variable: 'API_KEY'),
    usernamePassword(credentialsId: 'db-creds', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS'),
    file(credentialsId: 'kube-config', variable: 'KUBECONFIG'),
    sshUserPrivateKey(credentialsId: 'ssh-key', keyFileVariable: 'SSH_KEY')
]) {
    sh 'echo "Credentials are masked in logs"'
}
```

### 7.2 Security Best Practices

- Use credentials binding (never plain text in Jenkinsfile)
- Run Jenkins as non-root user
- Use Role-Based Access Control
- Enable CSRF protection
- Restrict Agent-to-Master security
- Keep Jenkins and plugins updated
- Use Pipeline Groovy sandbox for untrusted SCMs
- Back up JENKINS_HOME regularly

## 8. Common Mistakes

| Mistake | Impact | Solution |
|---------|--------|----------|
| Using freestyle jobs | Not repeatable | Use Pipeline jobs |
| No build discard policy | Disk full | Set logRotator |
| Hardcoding credentials | Security breach | Use credentials plugin |
| Master doing builds | Performance issues | Use agents |
| Too many plugins | Upgrade hell | Only needed plugins |
| No backup | Data loss | Back up JENKINS_HOME |
| Scripts not in SCM | No traceability | Pipeline from SCM |
| Single point of failure | Downtime | HA with active/passive |

## 9. Senior Engineer Perspective

### 9.1 Plugin Management

- Keep a plugin manifest with versions
- Use configuration-as-code plugin
- Test plugin upgrades in staging
- Consider compatibility before core upgrades
- Use thinBackup for plugin configuration

### 9.2 Migration Strategy (Freestyle to Pipeline)

1. Export freestyle job XML config
2. Use Job DSL plugin to generate seed jobs
3. Create Pipeline from SCM
4. Test alongside freestyle
5. Delete old freestyle jobs

## 10. Interview Questions (Easy)

1. What is Jenkins and what is it used for?
2. What is a Jenkins Pipeline?
3. What is the difference between freestyle and pipeline jobs?
4. What is a Jenkins agent/node?
5. What is a Jenkinsfile?
6. What is the difference between declarative and scripted pipelines?
7. What is a plugin in Jenkins?
8. How do you trigger a Jenkins job automatically?
9. What is JENKINS_HOME?
10. How do you view build logs?

## 10. Interview Questions (Medium)

11. How do you implement parallel stages in Jenkins?
12. What is the Jenkins shared library?
13. How do you handle credentials in Jenkins?
14. Agent any vs agent none?
15. How do you implement post-build actions?
16. What is the Jenkins DSL plugin?
17. How do you configure multibranch pipelines?
18. Jenkins security best practices?
19. How do you manage plugins?
20. What is Blue Ocean?

## 11. Advanced Interview Questions (Hard)

1. Design a shared library for 50+ microservice teams.
2. Implement rolling deployment with health checks.
3. Design Jenkins HA with zero downtime maintenance.
4. Secure Jenkins in multi-tenant environment.
5. Design canary release pipeline with auto-rollback.
6. Migrate Jenkins to GitHub Actions without downtime.
7. Design cross-team pipeline orchestration.
8. Implement pipeline testing (testing the pipeline).
9. Design backup and DR strategy for Jenkins.
10. Secrets rotation across 500+ pipelines.

## 11. Advanced Interview Questions (System Design)

11. Multi-region Jenkins for global teams.
12. Auto-scaling build farm with ECS/Kubernetes agents.
13. Cost-optimized Jenkins on spot instances.
14. Jenkins analytics platform for pipeline performance.
15. Plugin compatibility testing system.
16. Multi-tenant Jenkins for 100+ teams.
17. GitOps pipeline with Jenkins, ArgoCD, Helm.
18. Mobile CI/CD for iOS and Android.
19. Edge/IoT deployment pipeline.
20. SBOM and compliance report generation pipeline.

## 12. Expert-Level Interview Questions (Architect)

1. Design a global Jenkins platform for 10,000+ developers with zero downtime, multi-region failover, and self-service pipeline creation.

2. Architect a Jenkins-to-GitHub Actions migration for 5,000 developers running 20,000+ daily pipeline executions.

3. Design a pipeline intelligence system that predicts build failures using historical data, code changes, and test patterns.

4. Design a Jenkins plugin for supply chain security (SBOM, dependency attestation, signature verification).

5. Architect multi-cloud build infrastructure with agents auto-provisioned across AWS, Azure, GCP based on cost optimization.

6. Design system that auto-generates pipelines from architectural diagrams and API specs.

7. Zero-trust architecture where every build is cryptographically verified and immutably logged.

8. AIOps platform correlating pipeline failures with infrastructure changes, commits, and deployments.

9. Pipeline DSL that compiles to multiple CI/CD platforms from a single definition.

10. Chaos engineering pipeline that injects failures into Jenkins to validate HA/DR.

## 13. Debugging & Troubleshooting

### 13.1 Debug Pipeline

```groovy
pipeline {
    agent any
    options { skipDefaultCheckout() }
    stages {
        stage('Debug') {
            steps {
                script {
                    sh 'env | sort'
                    sh 'ls -la'
                    echo "Branch: ${env.BRANCH_NAME}"
                    echo "Build: ${env.BUILD_NUMBER}"
                    echo "Workspace: ${env.WORKSPACE}"
                    echo "Node: ${env.NODE_NAME}"
                }
            }
        }
    }
}
```

### 13.2 Agent Issues

```bash
cat /var/log/jenkins/agent.log
curl http://jenkins-master:8080/tcpSlaveAgentListener/
java -jar jenkins-cli.jar -s http://jenkins:8080 list-nodes
java -jar agent.jar -jnlpUrl http://jenkins:8080/computer/agent/slave-agent.jnlp -secret <secret>
```

## 14. Comparison Section

### Jenkins vs GitHub Actions

| Feature | Jenkins | GitHub Actions |
|---------|---------|---------------|
| Hosting | Self-hosted | Cloud + self-hosted |
| Setup | Complex | Built-in |
| Pipeline DSL | Groovy | YAML |
| Plugin Ecosystem | 1,800+ | 10,000+ actions |
| Scalability | Manual | Automatic |
| Maintenance | High | Low |
| Cost | Free (infrastructure) | Free tier + usage |

### Jenkins vs Azure DevOps

| Feature | Jenkins | Azure DevOps |
|---------|---------|-------------|
| Integration | Plugin-based | Deep Azure integration |
| Pipelines as Code | Groovy | YAML |
| UI | Classic + Blue Ocean | Modern |
| Hosting | Self-hosted | Cloud + self-hosted |
| Learning Curve | Steep | Moderate |

## 15. Revision Notes

```
JENKINS COMPONENTS
- Master: Scheduler, UI, API
- Agent: Worker node
- Executor: Process slot on agent
- Job: Configurable automation
- Pipeline: Code-defined CI/CD
- Plugin: Extension module

PIPELINE KEYWORDS
pipeline, agent, stages, stage, steps, post, when,
parallel, environment, options, parameters, triggers, tools, input

ESSENTIAL PLUGINS
Pipeline, Git, Blue Ocean, Credentials Binding,
Docker Pipeline, Kubernetes, Configuration as Code,
Job DSL, Slack/Email, SonarQube
```

## 16. Cheat Sheet

```text
+======================================================================+
|                     JENKINS CHEAT SHEET                               |
+======================================================================+

  PIPELINE STRUCTURE
+----------------------------------------------------------------------+
| pipeline {                                                           |
|     agent any                                                        |
|     stages {                                                         |
|         stage('Build') {                                             |
|             steps { echo 'Building...' }                             |
|         }                                                            |
|     }                                                                |
|     post { success { echo 'OK' } failure { echo 'FAIL' } }           |
| }                                                                    |
+----------------------------------------------------------------------+

  AGENT DIRECTIVES
+----------------------------------------------------------------------+
| agent any               | Any available agent                        |
| agent none              | No global agent (per-stage)                |
| agent { label '...' }   | Agent with specific label                 |
| agent { docker '...' }  | Run in Docker container                   |
+----------------------------------------------------------------------+

  STAGE CONDITIONS
+----------------------------------------------------------------------+
| when { branch 'main' }            | Branch condition                 |
| when { expression { ... } }       | Custom condition                |
| when { anyOf { branch 'main'; branch 'release/*' } } | Multiple      |
| when { buildingTag() }            | Tag build                       |
+----------------------------------------------------------------------+

  OPTIONS
+----------------------------------------------------------------------+
| buildDiscarder(logRotator(daysToKeepStr: '30'))                      |
| timeout(time: 1, unit: 'HOURS')                                      |
| retry(3), timestamps(), disableConcurrentBuilds()                    |
| skipDefaultCheckout(), ansiColor('xterm')                            |
+----------------------------------------------------------------------+

  POST CONDITIONS
+----------------------------------------------------------------------+
| always, success, failure, unstable, changed, aborted                |
| regression, fixed                                                    |
+----------------------------------------------------------------------+

  ENVIRONMENT VARIABLES
+----------------------------------------------------------------------+
| BUILD_NUMBER, BUILD_ID, BUILD_URL, JOB_NAME, NODE_NAME              |
| WORKSPACE, BRANCH_NAME, GIT_COMMIT, STAGE_NAME, EXECUTOR_NUMBER    |
+----------------------------------------------------------------------+

  CLI COMMANDS
+----------------------------------------------------------------------+
| java -jar jenkins-cli.jar -s URL build job                          |
| java -jar jenkins-cli.jar -s URL list-jobs                          |
| curl -X POST URL/job/job/build --user user:token                    |
+----------------------------------------------------------------------+

+======================================================================+
|  PRO TIPS: Always use Pipeline jobs (not freestyle).                 |
|  Store Jenkinsfile in SCM. Use Shared Library.                      |
|  Use Configuration as Code plugin. Back up JENKINS_HOME.            |
|  Use agents for builds, master for scheduling only.                 |
+======================================================================+
```
