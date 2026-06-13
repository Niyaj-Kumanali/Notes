# AWS

---

## Overview

- **Definition:** Amazon Web Services (AWS) is the leading cloud computing platform, offering over 200 services across compute, storage, databases, networking, machine learning, security, and analytics.
  - Operates on a pay-as-you-go pricing model — customers only pay for the resources they actually consume, with no upfront commitments or long-term contracts required
  - Serves millions of customers globally, from startups to enterprises to government agencies
  - Holds an estimated 33% market share of the cloud infrastructure market as of 2025
  - Massive scale allows it to offer lower prices than most competitors while maintaining industry-leading breadth and depth of services

- **Why It Exists:** AWS enables on-demand, pay-as-you-go access to scalable infrastructure, eliminating the need for upfront hardware investment and enabling rapid global deployment.
  - Before AWS, launching a new application required purchasing servers, networking equipment, and data center space — a process that could take weeks or months
  - With AWS, developers can provision virtual servers, databases, and networking in minutes through a web console, CLI, or API
  - This paradigm shift has democratized access to enterprise-grade infrastructure, allowing startups to compete with established companies without massive capital expenditure
  - Global infrastructure spans 33+ regions and 105+ availability zones

- **Key Concepts:**
  - **Regions** — AWS's geographic areas that each contain 3 or more availability zones; each region is completely independent and isolated from other regions
  - **Availability Zones (AZs)** — Physically isolated data centers within a region, each with independent power, cooling, and networking; connected through low-latency, high-bandwidth fiber links
  - **Edge Locations** — CDN endpoints for Amazon CloudFront that cache content close to end users; over 600+ edge locations globally
  - **Shared Responsibility Model** — AWS secures the cloud (physical security, hardware, networking, hypervisor); customers secure what's in the cloud (OS updates, application code, IAM configuration, data encryption)
  - **Well-Architected Framework** — Six pillars: Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, and Sustainability

---

## Core Services

- **EC2 (Compute):** Elastic Compute Cloud provides virtual machines with a vast selection of instance families optimized for different workloads.
  - General-purpose instances like t3 and m6i offer a balanced mix of compute, memory, and networking for standard applications
  - Compute-optimized instances (c6i, c7g) are designed for CPU-intensive workloads such as batch processing, scientific modeling, and gaming servers, delivering up to 30% better price-performance
  - Memory-optimized instances (r6i, x2idn) support in-memory databases like Redis and SAP HANA with up to 24 TB of RAM
  - Storage-optimized instances (i3, i4i) provide high, low-latency SSD storage for databases and data warehousing, with up to 30 TB of NVMe SSD storage
  - GPU instances (p4d, g5) deliver computational power for machine learning training, video rendering, and graphics-intensive applications, with p4d featuring 8 NVIDIA A100 GPUs and 400 Gbps networking
  - Flexible pricing: On-Demand (pay by hour/second with no commitment), Reserved Instances (up to 72% discount for 1-3 year terms), Spot Instances (up to 90% discount for fault-tolerant workloads), and Savings Plans
  - EC2 Auto Scaling automatically adjusts the number of instances based on demand
  - Key design decisions include choosing the right instance family, selecting appropriate storage (EBS vs. Instance Store), and configuring security groups and VPC placement

- **Lambda (Serverless):** AWS Lambda allows you to run code without provisioning or managing servers. Upload code, configure triggers, and Lambda handles all scaling, patching, and capacity management automatically.
  - Supports multiple languages including Node.js, Python, Java, Go, .NET, and Ruby
  - Scales automatically from zero to thousands of concurrent executions based on incoming traffic
  - Billed per invocation ($0.20 per 1M requests) and per duration (rounded to nearest millisecond, $0.0000166667 per GB-second)
  - Maximum 15-minute execution timeout, 10 GB of memory, 250 MB deployment package, 1,000 concurrent executions (soft limit)
  - Cold starts add 200ms-1s of latency; SnapStart reduces Java/.NET startup to under 200ms
  - Ideal for event-driven architectures, data processing pipelines, web API backends, and glue code connecting AWS services

- **ECS / EKS (Containers):** Amazon ECS is AWS's native container orchestration service; Amazon EKS is managed Kubernetes.
  - ECS uses task definitions (JSON describing containers) and services for scheduling; integrates with IAM, VPC, CloudWatch, ALB
  - ECS supports EC2 launch type (manage cluster) and Fargate (serverless — AWS manages infrastructure)
  - Fargate eliminates the need to provision, configure, or scale clusters of VMs for running containers
  - EKS handles Kubernetes control plane (etcd, API server, scheduler); supports standard kubectl, Helm, and Kubernetes tooling
  - EKS supports Istio, Prometheus, Fluentd, ArgoCD; integrates via IAM roles for service accounts (IRSA) and pod identity
  - ECS is simpler and more AWS-integrated; EKS is more portable with a larger ecosystem

- **S3 (Storage):** Amazon Simple Storage Service provides highly durable object storage with 99.999999999% (11 nines) durability across multiple AZs within a region.
  - Stores data as objects within buckets; each object consists of data, a key, and metadata; objects range from 0 bytes to 5 TB
  - Storage classes: STANDARD (frequent access), INTELLIGENT_TIERING (auto-moves between tiers), STANDARD_IA (infrequent access), ONEZONE_IA (single AZ), GLACIER (archive retrieval in minutes), GLACIER_DEEP_ARCHIVE (lowest cost, 12-hour retrieval)
  - Features include versioning, object lock, replication, server-side encryption (SSE-S3, SSE-KMS, SSE-C), event notifications, and static website hosting
  - Access control through bucket policies, IAM policies, ACLs (legacy), and Access Points
  - Supports multipart uploads for large objects and range GETs for partial object retrieval
  - Common use cases include backup and restore, data lakes, static website hosting, content distribution, and application data storage

- **RDS / DynamoDB (Databases):** Amazon RDS provides managed relational databases; DynamoDB is a fully managed NoSQL key-value and document database.
  - RDS supports PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, and Amazon Aurora
  - RDS automates hardware provisioning, database setup, patching, and backups with point-in-time recovery
  - RDS Multi-AZ deployments create a standby replica in a different AZ with synchronous replication for high availability
  - Amazon Aurora offers 5x MySQL throughput and 3x PostgreSQL throughput; distributed storage replicates data across 3 AZs with 6 copies
  - DynamoDB delivers single-digit-millisecond latency at any scale using a serverless architecture with SSD storage across 3 AZs
  - DynamoDB includes primary keys, secondary indexes (LSI and GSI), Streams, Global Tables, DAX caching, TTL, and on-demand capacity mode

- **VPC (Networking):** Amazon Virtual Private Cloud enables provisioning a logically isolated section of the AWS cloud where you can launch resources in a virtual network you define.
  - Complete control over IP address range selection (CIDR blocks), subnet creation, route table configuration, and network gateway management
  - Key components: Internet Gateways (public internet access), NAT Gateways (private subnet egress), route tables, Network ACLs (stateless subnet firewalls), and Security Groups (stateful instance firewalls)
  - VPC Peering connects two VPCs; Transit Gateway connects thousands of VPCs and on-premises networks through a central hub
  - VPN Gateway enables encrypted connections over internet; Direct Connect provides dedicated private network connections
  - VPC Endpoints (Gateway and Interface types) allow private connectivity to AWS services without traversing the public internet
  - Common design patterns: 3-tier architecture, multi-VPC with Transit Gateway, VPC sharing for centralized management

- **IAM (Identity and Access Management):** AWS Identity and Access Management enables managing access to AWS services and resources securely.
  - Core entities: users (permanent credentials), groups (collections of users), roles (temporary credentials for trusted entities), and policies (JSON documents defining permissions)
  - Best practices: principle of least privilege, IAM roles for applications, enable MFA, use condition keys, rotate credentials regularly
  - IAM Access Analyzer identifies resources shared with external entities
  - IAM Identity Center (formerly AWS SSO) provides centralized workforce access management across multiple accounts
  - IAM Roles for EC2 provide temporary credentials via instance metadata service, eliminating the need to store AWS credentials on instances
  - OIDC federation allows GitHub Actions or GitLab CI to assume IAM roles without storing long-lived AWS keys

```yaml
# VPC Architecture Design
VPC:
  CIDR: 10.0.0.0/16        # Main VPC with 65,536 IP addresses
  Subnets:
    Public:                  # Internet-facing subnets (load balancers, bastion hosts)
      - 10.0.1.0/24         # AZ-a: 256 IPs
      - 10.0.2.0/24         # AZ-b: 256 IPs
    Private:                 # Application and database subnets (no direct internet access)
      - 10.0.3.0/24         # AZ-a application
      - 10.0.4.0/24         # AZ-b application
      - 10.0.5.0/24         # AZ-a database
      - 10.0.6.0/24         # AZ-b database
  Gateways:
    - Internet Gateway       # Attached to public subnets for inbound/outbound internet traffic
    - NAT Gateway            # Deployed in public subnet, enables private subnet egress
    - Transit Gateway        # Optional: connects multiple VPCs and on-prem networks
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::myapp-data-*/*",
      "Condition": {
        "IpAddress": { "aws:SourceIp": "10.0.0.0/16" },
        "Bool": { "aws:SecureTransport": "true" }
      }
    }
  ]
}
```

---

## Common Mistakes

- **Default VPC** — Using the default VPC in any region is a security risk because it has an open internet gateway, unrestricted security groups, and no subnet segmentation.
  - **Why it looks correct:** The default VPC lets you launch instances and reach the internet immediately — a junior engineer sees a working network and assumes it is production-ready, not recognizing the wide-open security surface they inherited.

- **Open security groups (0.0.0.0/0)** — Allowing traffic from any IP on management ports like SSH (22) and RDP (3389) makes instances vulnerable to brute-force attacks and port scanning.
  - **Why it looks correct:** Opening port 22 from anywhere is the fastest way to enable SSH access during development, and the security risk feels abstract until the first brute-force attack actually succeeds.

- **No IAM roles for EC2** — Hardcoding AWS access keys on EC2 instances creates credential management headaches and can leak through log files or source code.
  - **Why it looks correct:** Hardcoding keys in a `.env` file or application config is the simplest approach, and the credentials are never visibly compromised during local testing.

- **No CloudTrail** — CloudTrail records API activity across your AWS account, providing the audit trail needed for security analysis, resource change tracking, and compliance reporting.
  - **Why it looks correct:** The application functions perfectly without CloudTrail enabled, and audit logging feels like a compliance checkbox rather than an operational necessity.

- **Single-AZ deployment** — Deploying critical workloads in a single Availability Zone means a power outage, network failure, or hardware issue in that AZ takes your entire application offline.
  - **Why it looks correct:** The application runs reliably in one AZ during development, and paying for duplicate infrastructure in a second AZ feels like wasted money for a failure mode that has never occurred.

- **No Auto Scaling** — Manually managing instance counts based on demand leads to either over-provisioning (wasting money) or under-provisioning (poor performance, downtime).
  - **Why it looks correct:** Manually sizing instances to handle peak load provides consistent performance during testing, and the complexity of scaling policies seems unnecessary when current capacity appears sufficient.

- **Public S3 buckets** — Publicly accessible S3 buckets have been the source of countless data breaches exposing millions of customer records.
  - **Why it looks correct:** Making a bucket public is the most straightforward way to share content, and the data feels harmless when you personally know what is stored there.

- **Root account daily use** — The root user has unrestricted access to the entire AWS account, including the ability to close the account.
  - **Why it looks correct:** The root account never hits permission errors, making daily work frictionless, and creating IAM users with scoped permissions feels like unnecessary overhead.

- **No cost budgets** — Without proactive cost monitoring, AWS bills can balloon unexpectedly from forgotten resources, unused storage, or elevated data transfer costs.
  - **Why it looks correct:** Each individual service appears inexpensive in isolation, and there is no visible feedback mechanism connecting a deployed resource to its monthly cost.

- **Over-provisioned resources** — Most production workloads use less than 50% of their provisioned resources, leading to significant wasted spending.
  - **Why it looks correct:** Larger instances provide comfortable headroom and prevent any risk of performance degradation, and the wasted capacity never visibly impacts the application.

---

## Key Design Considerations

- **VPC Architecture** — Design with multiple AZs (at least 2, ideally 3 for critical workloads), public/private subnet isolation, NAT gateways in each AZ, and Transit Gateway for connecting multiple VPCs.
  - For large organizations, use a centralized VPC for shared services with VPC Peering or Transit Gateway attachments to workload VPCs
  - Reserve at least 50% of the VPC CIDR for future growth

- **High Availability** — Use Application Load Balancers across AZs with health checks on the application endpoint.
  - Configure Auto Scaling Groups with a minimum of 2 instances across AZs and spread across instance types
  - Use RDS Multi-AZ for automatic failover of relational databases and read replicas in different regions for disaster recovery
  - Use ElastiCache with cluster mode and replica nodes in different AZs; configure Route 53 with health checks and failover routing

- **Disaster Recovery** — Four DR strategies with increasing complexity and cost:
  - Backup & Restore (cheapest, highest RTO — hours/days)
  - Pilot Light (core services running in minimal footprint in DR region, scale up when failover triggers)
  - Warm Standby (scaled-down but fully functional environment in DR region, scales up on failover)
  - Multi-Site Active-Active (both regions handle traffic simultaneously, lowest RTO/RPO — seconds/minutes)
  - Define RPO and RTO with business stakeholders, then choose the appropriate DR strategy

- **Security** — Implement defense in depth across all layers:
  - IAM least privilege: grant only necessary actions, use condition keys for additional restrictions
  - Encryption in transit: TLS 1.2+ enforced via ALB listener policies
  - Encryption at rest: KMS with automatic key rotation, S3 SSE, EBS encryption
  - Network security: security groups, NACLs, WAF for application-layer protection
  - Monitoring: CloudTrail for API audit trail, AWS Config for compliance, GuardDuty for threat detection
  - Secrets management: Secrets Manager for database credentials, Parameter Store for configuration data

- **Cost Optimization** — Use Spot instances for fault-tolerant workloads for 60-90% savings.
  - Purchase Reserved Instances or Savings Plans for baseline compute (30-72% discount)
  - Implement S3 lifecycle policies to automatically transition objects to cheaper storage classes
  - Right-size instances based on CloudWatch metrics and AWS Compute Optimizer
  - Tag everything with cost allocation tags and use AWS Budgets with alerts

- **Infrastructure as Code** — Use AWS CloudFormation or Terraform to define all infrastructure as version-controlled, reviewable code.
  - Tag all resources with consistent tags (Environment, Owner, CostCenter, Application)
  - Use CI/CD to validate and deploy IaC changes — cfn-lint or terraform plan in CI, approval gates for production
  - Use AWS AppConfig for application configuration separate from infrastructure
  - Never manually provision resources through the console for production workloads

---

## Real-World Scenarios

- **Scenario 1: Multi-Region Active-Active E-Commerce Platform with Global Database Replication**
  - **Context:** A global e-commerce company needs sub-second response times worldwide with 99.99% availability. Their single-region deployment in us-east-1 results in 2-3 second page load times for users in Asia and Europe.
  - **Problem:** Designing a multi-region active-active architecture requires solving database writes accepted locally but replicated globally, session state across regions, shopping cart data preservation during failover, and partial failure handling.
  - **Resolution:** Use Route 53 latency-based routing with health checks; deploy ALB and Auto Scaling Group across 3 AZs per region; use Aurora Global Database for relational layer; use DynamoDB Global Tables for session and cart data; use S3 with Cross-Region Replication fronted by CloudFront.

```yaml
# Route 53 DNS failover with primary/secondary regions and health checks
MyDNSRecord:
  Type: AWS::Route53::RecordSetGroup
  Properties:
    HostedZoneId: !Ref HostedZone
    RecordSets:
      - Name: api.example.com
        Type: A
        SetIdentifier: primary
        Region: us-east-1
        AliasTarget:
          DNSName: !GetAtt PrimaryALB.DNSName
          HostedZoneId: !GetAtt PrimaryALB.CanonicalHostedZoneID
        Failover: PRIMARY
        HealthCheckId: !Ref PrimaryHealthCheck
      - Name: api.example.com
        Type: A
        SetIdentifier: secondary
        Region: eu-west-1
        AliasTarget:
          DNSName: !GetAtt SecondaryALB.DNSName
          HostedZoneId: !GetAtt SecondaryALB.CanonicalHostedZoneID
        Failover: SECONDARY
        HealthCheckId: !Ref SecondaryHealthCheck
```

- **Scenario 2: Cost-Optimized CI/CD Infrastructure at Scale with Fargate and Spot**
  - **Context:** A startup runs 50+ microservices with EC2 costs ballooning to $40,000/month, with many instances running 24/7 despite only being utilized during business hours.
  - **Problem:** Build agents are idle ~60% of the time but incur full costs; development environments run full-size instances around the clock; staging has the same capacity as production but receives only 5% of traffic.
  - **Resolution:** Migrate stateless microservices to ECS Fargate with scheduled auto-scaling to zero during non-business hours; use Fargate Spot for 60-70% savings; use EC2 Spot instances for CI/CD build agents that scale based on queue depth; right-size remaining On-Demand instances; purchase Compute Savings Plans.

```bash
# Spot instance interruption handling script for CI/CD agents
SPOT_META="http://169.254.169.254/latest/meta-data/spot/termination-time"
curl -s $SPOT_META
if [ $? -eq 0 ]; then
  echo "Spot termination notice received. Draining jobs..."
  systemctl stop my-service
  aws autoscaling complete-lifecycle-action \
    --lifecycle-hook-name spot-termination \
    --auto-scaling-group-name my-asg \
    --lifecycle-action-result CONTINUE \
    --instance-id $(curl -s http://169.254.169.254/latest/meta-data/instance-id)
  echo "Instance drained, ready for termination"
fi
```

- **Scenario 3: HIPAA-Compliant Secure Data Lake for Healthcare Analytics**
  - **Context:** A healthcare company needs to ingest 5 TB of IoT device data per day from medical monitors and build a HIPAA-compliant data lake for analytics with protected health information (PHI).
  - **Problem:** All PHI data must be encrypted with automatically rotated keys; access restricted by role; data in transit must never traverse public internet; access logs retained for 7 years.
  - **Resolution:** Use S3 with SSE-KMS and automatic key rotation; block all public access at account level; use AWS Glue for ETL with scoped IAM roles; use Lake Formation for column-level and row-level security; set up VPC Endpoints for S3 and Glue; enable CloudTrail with log file validation and AWS Config with HIPAA-security rules.

---

## Scenario-Based Questions

- **Q: You are building a multi-region e-commerce platform with RPO < 1 second and RTO < 5 minutes. How do you design the database layer?**
  - **A:** Use Amazon RDS Aurora Global Database for the relational data layer with a single primary region and up to 5 secondary regions with less than 1 second replication latency. Failover completes in under 1 minute. For session and cart data, use DynamoDB Global Tables with multi-master replication for single-digit-millisecond writes in any region. Implement application-level retry with exponential backoff and test failover quarterly with GameDays.
  - **Interview follow-up:** How do you handle the case where a write is acknowledged to the client in the primary region but not yet replicated to a secondary region when that secondary region is promoted to primary during a failover?

- **Q: Your production EC2 instance running a critical batch job crashed at 3 AM. How do you design for automatic recovery?**
  - **A:** Configure the EC2 instance as part of an Auto Scaling Group with min=1, max=1. Use an ALB with a health check probing the application endpoint. After 3 consecutive health check failures, the ASG terminates the unhealthy instance and launches a new one. Store progress state in SQS or DynamoDB so the new instance can resume. Use termination lifecycle hooks to drain in-flight work and CloudWatch alarms for SNS notifications.

- **Q: Your team accidentally exposed an S3 bucket with customer PII data. How do you prevent this going forward?**
  - **A:** Implement a multi-layered defense: (1) Enable S3 Block Public Access at the account level; (2) Configure bucket policies with explicit Deny; (3) Use AWS Config rules with auto-remediation; (4) Add IAM policy conditions requiring approve-before-create workflows for s3:PutBucketPolicy; (5) Enable GuardDuty with S3 threat detection; (6) Create EventBridge rules for PutBucketPolicy or PutBucketAcl events.

- **Q: You are designing a serverless data pipeline processing 10 million events/hour with variable traffic spikes. How do you handle throttling?**
  - **A:** Use Kinesis Data Streams as the ingestion layer with at least 3 shards. Configure Lambda consumers with reserved concurrency to prevent downstream throttling. Set up an SQS dead-letter queue for failed events. Lambda's burst and account concurrency limits are soft limits that can be increased. Events remain in Kinesis for up to 7 days so no data is lost. Pre-provision Lambda concurrency for the baseline event rate.
  - **Interview follow-up:** What happens when your Lambda reserved concurrency plus burst concurrency exceeds the account-level concurrency limit — and how would you detect this before it causes production throttling?

- **Q: You need to migrate a 5 TB on-premise SQL Server database to RDS with less than 30 minutes of downtime. What migration strategy do you use?**
  - **A:** Use AWS DMS with ongoing change data capture (CDC). Phase 1: Perform full load migration during business hours. Phase 2: DMS continuously replicates ongoing changes via log-based CDC. Phase 3: During maintenance window, stop writes, let CDC catch up, verify data integrity, redirect connection string via DNS CNAME swap. Keep source database writable for rollback.

- **Q: Your VPC architecture is running out of IP addresses. How do you design for CIDR exhaustion without disrupting existing workloads?**
  - **A:** Add secondary CIDR blocks to the existing VPC (up to 5 total). Create new subnets and migrate workloads gradually. If primary VPC CIDR is completely exhausted, use Transit Gateway to connect separate VPCs. For future-proofing, use /16 CIDRs, reserve a /20 per AZ, leave 50% unallocated. Adopt IPv6 with a dual-stack architecture.

- **Q: Your CI/CD pipeline builds are slow because Docker layers aren't cached between runs. How do you solve this for ECS/Fargate deployments?**
  - **A:** For CodeBuild, use `--cache-from` with the previously built image in ECR. Structure Dockerfile to maximize layer caching: OS packages first, then dependencies, then application code. Use BuildKit with inline cache. For GitHub Actions, use `docker/build-push-action` with `cache-from: type=gha` and `cache-to: type=gha,mode=max`.
  - **Interview follow-up:** How does Docker layer caching break when your CI runners are ephemeral containers that do not share a Docker daemon between runs, and what cache export mode prevents the cache from growing unbounded?

- **Q: Your organization has 50 AWS accounts with no cost visibility. How do you implement cost governance?**
  - **A:** Use AWS Organizations with consolidated billing. Implement Service Control Policies (SCPs) to restrict expensive services. Enforce mandatory tagging (CostCenter, Environment, Owner, Application) using AWS Config rules and SCPs. Set AWS Budgets at account and OU level with notifications. Use Cost Explorer for analysis and implement chargeback with AWS Cost and Usage Reports.

- **Q: You are building a PCI-DSS compliant workload on AWS. What specific controls do you implement?**
  - **A:** (1) VPC with no public subnets for cardholder data; (2) CloudTrail in all regions with log file validation and SSE-KMS; (3) IAM with explicit Deny for cardholder data outside approved roles; (4) KMS with automatic key rotation using CMKs; (5) TLS 1.2+ through ALB; (6) AWS Config rules for PCI-DSS controls with auto-remediation; (7) GuardDuty and Security Hub; (8) Systems Manager Patch Manager; (9) SCPs preventing disable of CloudTrail, Config, or GuardDuty; (10) Amazon Inspector for quarterly vulnerability scans.

- **Q: Your Lambda function times out at 15 minutes for a long-running data processing job. How do you redesign?**
  - **A:** Decompose the job into smaller units using SQS to queue work items with each Lambda invocation processing one item. Use AWS Step Functions with a distributed map state for up to 10,000 parallel executions. For jobs needing more than 15 minutes of continuous compute, use ECS Fargate tasks launched via Step Functions with no maximum execution time. Enable Lambda SnapStart for Java/.NET to reduce cold start latency.

---

## Interview Questions

- **What is the difference between a Security Group and a NACL?**
  - **A:** Security Groups are stateful firewalls operating at the instance/ENI level with allow rules only. Network ACLs are stateless operating at the subnet level with both allow and deny rules evaluated in numeric order. Security Groups are evaluated as a whole; NACLs are evaluated in rule number order. Use Security Groups as the primary mechanism and NACLs as a secondary layer for explicit deny rules.

- **What is S3 Transfer Acceleration?**
  - **A:** S3 Transfer Acceleration uses AWS edge locations to accelerate uploads over long geographical distances. Files are uploaded to the nearest edge location, which transfers data to the S3 bucket over AWS's optimized private network. Costs additional per-GB transfer fees. Use the Speed Comparison tool to verify it improves performance for your use case.

- **Explain horizontal vs vertical scaling on AWS.**
  - **A:** Horizontal scaling (scaling out) adds more instances behind an ALB — unlimited theoretical scaling, higher availability, zero downtime. Vertical scaling (scaling up) increases the size of existing instances — hard limits, requires downtime, single point of failure. Horizontal scaling is strongly preferred for production workloads.

- **What is an Internet Gateway and when do you need a NAT Gateway?**
  - **A:** An Internet Gateway (IGW) allows bidirectional communication between a VPC and the internet for resources with public IPs. A NAT Gateway allows resources in private subnets to initiate outbound connections while preventing inbound connections. Use IGW for public-facing resources; use NAT Gateway for private resources needing outbound-only internet access.

- **How does AWS Shield protect against DDoS attacks?**
  - **A:** AWS Shield Standard (free) protects against common Layer 3 and Layer 4 attacks using AWS's global network infrastructure. Shield Advanced ($3,000/month) provides enhanced detection, near real-time visibility, DDoS cost protection, and access to the DDoS Response Team. Pair with AWS WAF for Layer 7 protection.

- **What is the difference between an ALB, NLB, and CLB?**
  - **A:** ALB operates at Layer 7 with content-based routing (path, host, query string), supports WebSocket, HTTP/2, and gRPC. NLB operates at Layer 4 with extreme performance (25M requests/second), ultra-low latency, preserves client source IP, supports static IPs. CLB is the legacy ELB (deprecated). Use ALB for HTTP/HTTPS, NLB for extreme performance or static IPs, avoid CLB entirely.

- **Explain the AWS Well-Architected Framework.**
  - **A:** The framework consists of six pillars: Operational Excellence (run and monitor systems), Security (protect data and systems), Reliability (perform correctly when expected), Performance Efficiency (use resources efficiently), Cost Optimization (deliver value at lowest price), and Sustainability (minimize environmental impact). The Well-Architected Tool provides a consistent approach for reviewing architectures and generating recommendations.

- **What is the difference between EBS and Instance Store?**
  - **A:** EBS provides persistent block storage that survives instance stops and terminations — supports snapshots, encryption, provisioned IOPS, up to 16 TB. Instance Store provides temporary, ephemeral storage physically attached to the host — highest possible IOPS but data is lost if the instance stops or fails. Use EBS for databases and persistent data; use Instance Store for temporary data and caches.

- **How do you implement blue-green deployment on AWS?**
  - **A:** Maintain two identical environments — blue (current) and green (new). Deploy to green, validate with smoke tests, then update the ALB listener to route traffic to green. For rollback, switch traffic back to blue. AWS CodeDeploy automates this. For Lambda, use aliases with weighted routing, gradually shifting traffic in 1% increments.

- **What is CloudFormation and how does it differ from Terraform?**
  - **A:** CloudFormation is AWS's native IaC service with drift detection, StackSets, no state file management, and deep AWS integration. Terraform is cloud-agnostic, uses HCL syntax, supports remote state backends, has a broader provider ecosystem, and offers advanced logic constructs. Choose CloudFormation for AWS-only deployments; choose Terraform for multi-cloud environments.

---

## Developer Recommendations

- **Use IAM roles over access keys** — Long-lived access keys are one of the top breach vectors in AWS. IAM roles with STS temporary credentials eliminate the need to rotate keys or risk leaking them. For EC2, use instance profiles. For CI/CD, use OIDC federation so GitHub Actions, GitLab CI, and CircleCI can assume IAM roles directly. For local development, use AWS SSO with `aws sso login`. Temporary credentials expire within hours; leaked access keys provide indefinite access until manually rotated. In a real incident, a developer accidentally committed AWS access keys to a public GitHub repository — automated scanners launched GPU instances totaling tens of thousands of dollars before the keys could be revoked.

- **Tag everything, enforce tagging with SCPs** — Without a comprehensive tagging strategy, cost allocation becomes guesswork and automation cannot target specific resources. Define mandatory tags: CostCenter, Environment, Owner, Application, DataClassification. Enforce via SCPs that deny resource creation when required tags are missing using `aws:RequestTag` and `aws:ResourceTag` condition keys. Use tag-based IAM policies for fine-grained access control.

- **Use S3 Intelligent-Tiering for unpredictable access patterns** — Manually choosing storage classes leads to overpaying or incurring retrieval fees. S3 Intelligent-Tiering monitors access patterns and automatically moves objects between Frequent, Infrequent, and Archive Instant tiers with a small monitoring fee of $0.0025 per 1,000 objects. Ideal for data lakes, logs, and content repositories with changing access patterns.

- **Prefer managed services over self-managed — know the trade-off** — Managed services like RDS, ElastiCache, and DynamoDB reduce operational burden significantly but cost more. For example, RDS PostgreSQL costs 2-3x more than self-managed but saves 10+ hours per week of operational tasks. Use managed services when the operational complexity outweighs the cost premium. Reserve self-managed for specific configurations not supported by managed services.

- **Implement cost-aware development practices** — Developers are the primary drivers of cloud costs. Use Cost Explorer with resource-level tags, set AWS Budgets with Slack/email alerts, implement cost gates in CI/CD, use `aws-nuke` to clean up ephemeral environments. Make cost visible — developers who see the cost of their choices naturally make more cost-effective decisions.

- **Design for failure from day one** — Building multi-AZ or multi-region architecture is much harder to retrofit. Use at least 2 subnets in development, implement circuit breakers and retries with exponential backoff, use AWS Fault Injection Simulator to test failure modes. Every external dependency should have a fallback. A SaaS company saved $3,000/month in a single AZ, then lost 8 hours of production from a transformer failure — the revenue loss exceeded 60 times the annual savings.

- **Use Infrastructure as Code for everything** — Every piece of infrastructure should be defined in CloudFormation or Terraform. Manual configuration changes produce drift, which is the enemy of reliability. Use IaC linting and security scanning tools (cfn-lint, tfsec, checkov) in CI/CD to catch misconfigurations before deployment.

- **Monitor the right metrics, not everything** — Focus on the Four Golden Signals per service: latency, traffic, errors, and saturation. Use composite CloudWatch alarms that combine multiple conditions before alerting to reduce false positives. Use CloudWatch Contributor Insights for high-cardinality traffic analysis.
