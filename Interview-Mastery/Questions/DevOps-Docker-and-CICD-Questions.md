# DevOps, Docker, and CI/CD Questions

## Questions

1. What is Git?
2. What is Git branch?
3. Difference between merge and rebase.
4. What is pull request?
5. How do you resolve merge conflict?
6. What is CI/CD?
7. What is Jenkins?
8. How do you create Jenkins pipeline?
9. What are Jenkins stages?
10. How do you manage Jenkins credentials?
11. How do you deploy Spring Boot using Jenkins?
12. How did Jenkins reduce deployment time in your project?
13. What is Docker?
14. Why use Docker?
15. What is Docker image?
16. What is Docker container?
17. What is Dockerfile?
18. What is Docker Compose?
19. Difference between image and container.
20. What is multi-stage Docker build?
21. How do you reduce Docker image size?
22. How do you pass environment variables to Docker?
23. How do you debug container startup failure?
24. What is logging in Docker?
25. What is health check?
26. What is deployment rollback?
27. What is blue-green deployment?
28. What is canary deployment?
29. What is infrastructure as code?
30. What are common deployment issues?

---

## Answers

1. What is Git?
   - **Answer:**
      - Git is a distributed version control system that tracks source code changes, enabling collaboration and rollback
      - We used Git with GitHub, feature branches for development, and pull requests for code review before merging to main
      - We used a simplified Git flow branching strategy with feature branches off main, short-lived branches for fixes, and release branches for deployment readiness

2. What is Git branch?
   - **Answer:**
      - A Git branch is a parallel version of the codebase for independent work without affecting the main code
      - In our workflow, each feature or bug fix had its own branch, and after review, it was merged to main via a pull request
      - Branches enable parallel development by letting multiple developers work on different features simultaneously; we resolved merge conflicts by merging the target branch into the feature branch

3. Difference between merge and rebase.
   - **Answer:**
      - Merge creates a commit combining two branch histories, preserving the complete timeline
      - Rebase rewrites commit history by applying commits onto another branch linearly
      - I prefer merge for feature branches to preserve history and rebase for cleaning up local commits before pushing
      - A key rule is to never rebase shared branches that others are working on, as it rewrites commit history and causes conflicts for collaborators

4. What is pull request?
   - **Answer:**
      - A pull request (PR) lets developers request review of their branch changes before merging
      - In our team, every change went through a PR - at least one reviewer checked logic, performance, and testing before merge
      - PR best practices include keeping PRs small and focused, including clear descriptions of changes, and ensuring CI passes before requesting review

5. How do you resolve merge conflict?
   - **Answer:**
      - I pull the latest target branch, run `git merge` to see conflicts, then edit conflicting files keeping the correct code
      - I use VS Code's merge editor for a side-by-side comparison, then run tests after resolution before committing
      - I communicate with the other developer if the conflict involves their code, and ensure the final commit is clean and tested

6. What is CI/CD?
   - **Answer:**
      - CI/CD automates building, testing, and deploying code changes
      - In our Jenkins pipeline, every push to main triggered automatic build, test, Docker image creation, and deployment to EC2
      - CI catches issues early, and CD reduces manual deployment errors
      - CI focuses on frequent code merges and automated testing to catch issues early; CD extends this by automatically deploying to production after successful CI

7. What is Jenkins?
   - **Answer:**
      - Jenkins is an open-source automation server for CI/CD pipelines
      - We used declarative pipelines in a Jenkinsfile that checked out code, ran Maven build and tests, created a Docker image, pushed to Docker Hub, and deployed to EC2 via SSH
      - Jenkins has a rich plugin ecosystem for extending functionality; pipeline as code stores Jenkinsfile in the repo for versioning; and agents can be configured for parallel build stages

8. How do you create Jenkins pipeline?
   - **Answer:**
      - I create declarative pipelines using a Jenkinsfile in the repo root with stages: checkout, build (mvn clean install), test (mvn test), Dockerize (docker build and push), deploy (SSH to EC2, docker pull, docker-compose up)
      - Each stage has clear success/failure conditions
      - Environment-specific configuration uses Jenkins credentials to securely manage secrets, and parameterized builds allow manual triggers with custom parameters

9. What are Jenkins stages?
   - **Answer:**
      - Jenkins stages are logical pipeline steps representing build phases
      - Our pipeline stages: Checkout, Build, Test (parallel unit and integration), Docker Build, Docker Push, Deploy to Staging, Smoke Test, and Deploy to Production (with manual approval)
      - Stages can run in parallel with each having its own agent, and they make pipeline failures visible at a glance by clearly showing which step broke

10. How do you manage Jenkins credentials?
    - **Answer:**
       - We used Jenkins' built-in credential store for Docker Hub credentials, EC2 SSH keys, and database passwords - referenced in the Jenkinsfile using `credentialsId`
       - This avoided hardcoding secrets in the pipeline code or repository
       - Environment variables are used for non-sensitive configuration, and secrets should never be logged or printed in pipeline output to prevent accidental exposure

11. How do you deploy Spring Boot using Jenkins?
    - **Answer:**
       - Our Jenkins pipeline built the Spring Boot app with Maven, created a Docker image from the JAR, pushed it to Docker Hub, then SSHed into EC2 to pull the new image and restart containers using Docker Compose
       - Fully automated from commit to deployment
       - Docker Compose works well for single-host deployments; for multi-host deployments with load balancing, AWS ECS manages container orchestration across multiple instances

12. How did Jenkins reduce deployment time in your project?
    - **Answer:**
       - Jenkins reduced deployment from manual 30-minute SSH-and-copy sessions to automated 5-7 minute pipelines
       - Before Jenkins, developers manually copied JARs to EC2 and restarted services
       - Now a webhook triggers the full pipeline, eliminating human error and reducing downtime
       - Manual deployment took 30 minutes with risks of configuration mistakes, while the automated Jenkins pipeline completed in 7 minutes with consistent, repeatable steps

13. What is Docker?
    - **Answer:**
       - Docker packages applications with all dependencies into lightweight, portable containers
       - In our projects, each service (Spring Boot API, Kafka, InfluxDB, Redis) ran in its own Docker container, ensuring consistent environments across dev, staging, and production on EC2
       - VMs run a full OS per instance with dedicated resources, while containers share the host OS kernel with isolated processes — making them lighter and faster; Docker improved our EC2 resource utilization by running multiple services per instance instead of one per VM

14. Why use Docker?
    - **Answer:**
       - Docker eliminates environment inconsistencies by packaging the application with all dependencies
       - In the cold-chain project, Docker ensured Spring Boot, Kafka, and InfluxDB versions were identical everywhere, and container startup and recovery was fast
       - Each container has its own filesystem, network, and process space providing strong isolation; Docker Compose simplified local development by starting all services with a single command

15. What is Docker image?
    - **Answer:**
       - A Docker image is a read-only template with instructions for creating a container, containing the application code, runtime, libraries, and configuration
       - We built images from a Dockerfile using `docker build` and stored them on Docker Hub for EC2 distribution
       - Image layering means each Dockerfile instruction creates a layer, and layers are cached so subsequent builds only rebuild changed layers, speeding up the build process

16. What is Docker container?
    - **Answer:**
       - A Docker container is a running instance of a Docker image - an isolated process with its own filesystem and network
       - Each container (Spring Boot, Kafka, Redis) shared the EC2 host's kernel but had isolated environments, making them portable across any Docker host
       - Containers follow a lifecycle of create, start, stop, restart, and remove; Docker's health check feature monitors container health and automatically restarts containers that fail

17. What is Dockerfile?
    - **Answer:**
       - A Dockerfile is a script with instructions to build a Docker image
       - Our Spring Boot Dockerfile used multi-stage build: first stage compiled code with Maven, second stage used a slim OpenJDK image to run the JAR, keeping the image around 150MB
       - Our Spring Boot Dockerfile: `FROM maven:3.8 AS build` for compilation, then `FROM openjdk:17-jre-slim` for runtime, `COPY --from=build` to transfer the built JAR, and `ENTRYPOINT ["java", "-jar", "app.jar"]` to start the application

18. What is Docker Compose?
    - **Answer:**
       - Docker Compose defines multi-container applications using a YAML file
       - In the cold-chain project, our `docker-compose.yml` defined Spring Boot API, Kafka, Zookeeper, Redis, and InfluxDB - all starting in order with one `docker-compose up` command
       - The compose file defines service dependencies with `depends_on` for startup order, environment variables for configuration, port mappings to expose services, and volume mounts for persistent data

19. Difference between image and container.
    - **Answer:**
       - An image is a static, read-only template (like a class), while a container is a running instance (like an object)
       - Images are stored in registries and versioned; containers are ephemeral, can be started, stopped, and deleted without affecting the image
       - An image is like a recipe and a container is like the cooked meal; multiple containers can run from the same image, each with its own independent state

20. What is multi-stage Docker build?
    - **Answer:**
       - Multi-stage build uses multiple FROM statements, copying artifacts from intermediate stages into the final stage
       - Our Spring Boot Dockerfile used this: build compiled with Maven, runtime only had the JAR and JDK, reducing the image from 700MB to ~150MB
       - Multi-stage builds improve security by excluding build tools like Maven and compilers from the final image, reducing the attack surface in production

21. How do you reduce Docker image size?
    - **Answer:**
       - I use multi-stage builds, choose slim base images (openjdk:17-jre-slim), clean package manager caches, and minimize layers
       - Our Spring Boot image went from 700MB to 150MB
       - Using `.dockerignore` excludes unnecessary files like logs and .git
       - Tools like `dive` analyze image layer contents to identify space wastage and optimize layer composition

22. How do you pass environment variables to Docker?
    - **Answer:**
       - We passed environment variables through Docker Compose's `environment` section or `.env` files
       - For production, Jenkins injected them at container runtime
       - Spring Boot's properties were overridden using environment variables like `SPRING_DATASOURCE_URL` mapped in the compose file
       - Following the 12-factor app methodology, configuration lives in environment variables instead of code, with separate .env files for dev, staging, and production environments

23. How do you debug container startup failure?
    - **Answer:**
       - I check `docker logs <container>` for startup errors, verify environment variables, inspect the Dockerfile for entry point issues, and check if required services are available
       - Spring Boot startup failures were often due to database connection issues or missing env vars
       - `docker exec -it <container> /bin/bash` provides interactive shell access for diagnostics; resource limits like memory and CPU can prevent startup, and port conflicts on the host must be verified

24. What is logging in Docker?
    - **Answer:**
       - Docker containers output logs to stdout/stderr, collected by the logging driver
       - We used the JSON-file driver locally and CloudWatch agent for centralized log collection on EC2
       - Each container's logs were accessible through `docker logs` and aggregated in CloudWatch Logs
       - Docker supports logging drivers like syslog, fluentd, and awslogs for different backends; structured JSON logging improves searchability and enables filtering by fields in log aggregation tools

25. What is health check?
    - **Answer:**
       - A health check verifies that a container is running correctly beyond just the process being alive
       - In our Spring Boot containers, we used Actuator's `/actuator/health` endpoint
       - Docker restarted containers that failed health checks, and the load balancer removed them from the target group
       - Liveness probes check if the container is alive and should be restarted if failing; readiness probes check if the container is ready to accept traffic. Spring Boot separates these with dedicated Actuator endpoints `/actuator/health/liveness` and `/actuator/health/readiness`

26. What is deployment rollback?
    - **Answer:**
       - Deployment rollback reverts to a previous stable version when a deployment causes issues
       - We kept the last known-good Docker image tag, and rollback meant re-tagging the previous image and redeploying
       - We documented and tested the rollback procedure
       - Code rollback is safe only if database schema changes are also backward-compatible; rolling back code while the DB has new schema can cause failures, so schema changes must be designed to work with both old and new code versions

27. What is blue-green deployment?
    - **Answer:**
       - Blue-green deployment runs two identical environments (blue=current, green=new) and switches traffic after testing
       - We did not implement this on EC2 due to cost, but for zero-downtime, I would use AWS ALB target group switching to route traffic between environments
       - Blue-green reduces downtime to just the traffic switch time and enables instant rollback by switching back to the previous environment, but requires double the infrastructure during the transition period

28. What is canary deployment?
    - **Answer:**
       - Canary deployment routes a small percentage of traffic to the new version while keeping most on the old version, monitoring before full rollout
       - On EC2, we could use ALB weighted target groups - send 10% traffic to new version, monitor for 15 minutes, then gradually increase to 100%
       - During canary deployment, monitor error rate, latency, and business metrics; set up automated rollback triggers that revert to the stable version if error rates exceed thresholds or latency degrades

29. What is infrastructure as code?
    - **Answer:**
       - Infrastructure as Code (IaC) manages infrastructure through configuration files instead of manual setup
       - We used Docker Compose as basic IaC for container configuration
       - For advanced needs, Terraform or CloudFormation define EC2 instances, load balancers, and security groups as code
       - IaC enables version-controlled infrastructure stored in Git, ensuring every environment change is auditable and repeatable; it reduces configuration drift between dev, staging, and production by using the same code to provision all environments

30. What are common deployment issues?
    - **Answer:**
       - Common issues: environment-specific config mistakes, database schema mismatch, dependency version conflicts, resource exhaustion (disk full, memory leak), and health check failures
       - We once had a Spring Boot container fail on EC2 because of timezone mismatch causing JWT validation errors
       - Mitigations include comprehensive smoke tests before full rollout, canary deployments for risky changes, monitoring dashboards for early anomaly detection, and documented runbooks for quick recovery
