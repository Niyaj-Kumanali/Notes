# AWS

---

## Overview

- **Definition:** Amazon Web Services (AWS) is the leading cloud computing platform offering 200+ services across compute, storage, databases, networking, machine learning, and more.
- **Why It Exists:** AWS enables on-demand, pay-as-you-go access to scalable infrastructure, eliminating the need for upfront hardware investment and enabling rapid global deployment.
- **Key Concepts:** **Regions** (geographic areas with 3+ AZs), **Availability Zones** (isolated data centers within a region), **Edge Locations** (CDN endpoints), **Shared Responsibility Model** (AWS secures the cloud, customer secures what's in the cloud), **Well-Architected Framework** (6 pillars: Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, Sustainability)

---

## Core Services

- **EC2 (Compute):** Virtual machines with instance families — General (t3, m6i), Compute (c6i), Memory (r6i), Storage (i3), GPU (p4d). Supports On-Demand, Reserved, Spot, and Savings Plan pricing.
- **Lambda (Serverless):** Run code without provisioning servers. Supports multiple languages, auto-scales, billed per invocation. Max 15-min timeout, 10 GB memory.
- **ECS / EKS (Containers):** ECS is AWS-native container orchestration; EKS is managed Kubernetes. Fargate provides serverless containers without managing nodes.
- **S3 (Storage):** Object storage with 11 9s durability. Classes: STANDARD, INTELLIGENT_TIERING, STANDARD_IA, ONEZONE_IA, GLACIER, DEEP_ARCHIVE.
- **RDS / DynamoDB (Databases):** RDS provides managed relational DBs (PostgreSQL, MySQL, Aurora). DynamoDB is managed NoSQL with single-digit-millisecond latency at any scale.
- **VPC (Networking):** Isolated network with public/private subnets, Internet/NAT Gateways, route tables, security groups, NACLs, VPC Peering, Transit Gateway.
- **IAM (Security):** Identity and Access Management — users, groups, roles, policies. Follow least privilege. Use roles for applications, not long-lived keys.

```yaml
VPC:
  CIDR: 10.0.0.0/16
  Subnets:
    Public:  [10.0.1.0/24, 10.0.2.0/24]
    Private: [10.0.3.0/24, 10.0.4.0/24]
  Gateways:
    - Internet Gateway (public subnets)
    - NAT Gateway (private subnets egress)
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
        "IpAddress": { "aws:SourceIp": "10.0.0.0/16" }
      }
    }
  ]
}
```

---

## Common Mistakes

- **Default VPC** — Security risk; always use custom VPCs
- **Open security groups (0.0.0.0/0)** — Vulnerable; restrict by IP/CIDR
- **No IAM roles for EC2** — Credential management headache; use instance profiles
- **No CloudTrail** — No audit trail; enable in all regions
- **Single-AZ deployment** — No high availability; use Multi-AZ
- **No Auto Scaling** — Manual scaling; use ASG with ELB
- **Public S3 buckets** — Data exposure; block public access by default
- **Root account daily use** — Security risk; use IAM users with MFA
- **No cost budgets** — Bill shock; set up AWS Budgets and alerts
- **Over-provisioned resources** — Wasted money; right-size instances

---

## Key Design Considerations

- **VPC Architecture** — Design with multiple AZs, public/private subnet isolation, NAT gateways for private egress, and transit gateway for multi-VPC connectivity
- **High Availability** — Use ALB across AZs, Auto Scaling Groups with min 2 instances, RDS Multi-AZ, read replicas for scale, ElastiCache cluster mode
- **Disaster Recovery** — Strategies: Backup & Restore, Pilot Light, Warm Standby, Multi-Site Active-Active. Target RPO/RTO based on business requirements
- **Security** — IAM least privilege, encryption in transit (TLS 1.2+) and at rest (KMS), CloudTrail for auditing, AWS Config for compliance, GuardDuty for threat detection
- **Cost Optimization** — Spot instances (60-90% savings), Reserved Instances/Savings Plans (30-72%), S3 lifecycle policies, right-sizing via CloudWatch metrics
- **Infrastructure as Code** — Use CloudFormation or Terraform for reproducible deployments. Tag all resources for cost allocation and automation

---

## Real-World Scenarios

**Scenario 1: Multi-Region Active-Active E-Commerce Platform**
A global e-commerce company needs sub-second response times worldwide with 99.99% availability. Use Route 53 latency-based routing to direct users to the nearest region. Deploy ALB + Auto Scaling Groups in each region with RDS Aurora Global Database for cross-region replication. DynamoDB Global Tables for session state. S3 Cross-Region Replication for product images. CloudFront with multiple origins for static/dynamic content. Failover via Route 53 health checks.

```yaml
# Route 53 DNS failover with primary/secondary regions
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
      - Name: api.example.com
        Type: A
        SetIdentifier: secondary
        Region: eu-west-1
        AliasTarget:
          DNSName: !GetAtt SecondaryALB.DNSName
          HostedZoneId: !GetAtt SecondaryALB.CanonicalHostedZoneID
        Failover: SECONDARY
```

**Scenario 2: Cost-Optimized CI/CD Infrastructure at Scale**
A startup runs 50+ microservices with dev, staging, and prod environments. EC2 costs are $40K/month. Migrate stateless workloads to ECS Fargate (no EC2 management). Use Spot Instances (60-90% cheaper) via Spot Capacity Providers. Add Compute Savings Plans for baseline compute (30-40% discount). S3 Lifecycle policies move logs to Glacier after 30 days. Auto-scaling policies scale to zero in dev during non-business hours.

```bash
# Spot instance interruption handling script
curl -s http://169.254.169.254/latest/meta-data/spot/termination-time
if [ $? -eq 0 ]; then
  systemctl stop my-service
  aws autoscaling complete-lifecycle-action \\
    --lifecycle-hook-name spot-termination \\
    --auto-scaling-group-name my-asg \\
    --lifecycle-action-result CONTINUE \\
    --instance-id $(curl -s http://169.254.169.254/latest/meta-data/instance-id)
fi
```

**Scenario 3: Secure Data Lake for Healthcare Analytics**
A healthcare company needs a HIPAA-compliant data lake ingesting 5 TB/day from IoT devices. Use S3 with SSE-KMS encryption and automatic key rotation. Block Public Access + bucket policies restricting access to specific VPC endpoints. Glue for ETL with IAM roles scoped to specific databases. Lake Formation for fine-grained column/row-level access control. CloudTrail + Config for audit trail. Athena + QuickSight for analytics. VPC Endpoints ensure data never traverses the public internet.

---

## Scenario-Based Questions

1. **Q: You are building a multi-region e-commerce platform with RPO < 1 second and RTO < 5 minutes. How do you design the database layer?**
   A: Use RDS Aurora Global Database — primary region handles writes, up to 5 secondary regions with <1 second replication lag. For session/cart data, use DynamoDB Global Tables (multi-master, single-digit-millisecond latency). Implement application-level retry with exponential backoff for replication conflicts. Aurora Global Database failover completes in <1 minute. Add read replicas in each region for read scaling. Test failover quarterly with GameDays.

2. **Q: Your production EC2 instance running a critical batch job crashed at 3 AM. How do you design for automatic recovery?**
   A: Configure the instance in an Auto Scaling Group with min=1, max=1. Use an ALB health check probing the application endpoint. After 3 consecutive failures, ASG terminates and relaunches. Store job state in SQS or DynamoDB so the new instance can resume. Use CloudWatch alarms for SNS notifications. Use termination lifecycle hooks for draining connections.

3. **Q: Your team accidentally exposed an S3 bucket with customer PII data. How do you prevent this going forward?**
   A: Implement a multi-layered defense: (1) S3 Block Public Access at account level — overrides any bucket settings. (2) Bucket policies with explicit Deny for public access. (3) AWS Config rules `s3-bucket-public-read-prohibited` and `s3-bucket-public-write-prohibited` with auto-remediation via SSM. (4) IAM policy requiring `s3:PutBucketPolicy` with condition key for approved templates. (5) GuardDuty for S3 threat detection. (6) CloudTrail alerts on `PutBucketPolicy` events.

4. **Q: You are designing a serverless data pipeline processing 10 million events/hour with variable traffic spikes. How do you handle throttling?**
   A: Use Kinesis Data Streams with resharding or Kinesis Firehose for near-real-time loading to S3. Lambda consumers with reserved concurrency to avoid throttling other functions. SQS as a dead-letter queue for failed events. If Lambda throttles (burst limit exceeded), events stay in Kinesis for up to 7 days. Pre-provision Lambda concurrency for extreme spikes.

5. **Q: You need to migrate a 5 TB on-premise SQL Server database to RDS with < 30 minutes downtime. What migration strategy do you use?**
   A: Use AWS DMS with ongoing change data capture (CDC). Phase 1: Full load migration to RDS. Phase 2: DMS continuously replicates changes via log-based CDC. Phase 3: During maintenance window, stop writes to source, let CDC catch up, redirect application connection string. Use a DNS CNAME swap for instant traffic redirect. Rollback plan: keep source writable, reverse replication if issues arise.

6. **Q: Your VPC architecture is running out of IP addresses. How do you design for CIDR exhaustion without disrupting existing workloads?**
   A: Add secondary CIDR blocks to the existing VPC (up to 5 total). Create new subnets in the secondary CIDR and migrate workloads gradually. Use Transit Gateway to connect secondary VPCs if primary VPC CIDR is exhausted. For future-proofing: use /16 CIDRs, reserve /20 per AZ, leave 50% unallocated. Implement IPv6 — every instance gets a globally unique IPv6 address, eliminating NAT need.

7. **Q: Your CI/CD pipeline builds are slow because Docker layers aren't cached between runs. How do you solve this for ECS/Fargate deployments?**
   A: Use a self-hosted build agent with persistent Docker cache. In CodeBuild, use `--cache-from` with cached image in ECR. Structure Dockerfile to maximize layer caching: OS packages first, system deps, app deps (package.json), then app code last. Use BuildKit inline cache. For GitHub Actions, use `docker/build-push-action` with `cache-from: type=gha`.

8. **Q: Your organization has 50 AWS accounts with no cost visibility. How do you implement cost governance?**
   A: Use AWS Organizations with consolidated billing and SCPs to restrict expensive services. Tag all resources with CostCenter, Environment, Owner — enforce via AWS Config and SCPs. Set AWS Budgets at account and OU level with alerts at 50%, 80%, 100%. Use Cost Explorer for reporting. Implement chargeback: monthly cost reports per CostCenter, published to S3, visualized in QuickSight.

9. **Q: You are building a PCI-DSS compliant workload on AWS. What specific controls do you implement?**
   A: Key controls: (1) VPC with no public subnets for cardholder data — use PrivateLink. (2) CloudTrail in all regions with log file validation and encryption. (3) IAM with explicit Deny for cardholder data outside approved roles. (4) KMS with automatic key rotation. (5) TLS 1.2+ enforced via ALB listener policies. (6) AWS Config rules for PCI-DSS. (7) GuardDuty + Security Hub for monitoring. (8) SSM Patch Manager with maintenance windows. (9) SCPs preventing disablement of CloudTrail, Config, or GuardDuty. (10) Quarterly vulnerability scans via Amazon Inspector.

10. **Q: Your Lambda function times out at 15 minutes for a long-running data processing job. How do you redesign?**
    A: Decompose the job into smaller chunks using SQS — each Lambda invocation processes one batch within the timeout. Use Step Functions with a distributed map state — up to 10,000 parallel executions. Use ECS Fargate tasks via Step Functions (`RunTask`) for jobs needing >15 minutes (Fargate has no time limit). For cold start issues, use Lambda SnapStart.

---

## Interview Questions

1. **What is the difference between a Security Group and a NACL?**
   A: Security Groups are stateful (return traffic auto-allowed), operate at instance level, support allow rules only. NACLs are stateless, operate at subnet level, support allow and deny rules. Use SGs for primary security, NACLs as secondary layer.

2. **What is an S3 Transfer Acceleration?**
   A: Uses AWS edge locations to accelerate uploads over long distances. Files upload to an edge location, then transfer over AWS's private network to S3. Useful for global users uploading to a single bucket region. Extra cost applies.

3. **Explain horizontal vs vertical scaling on AWS.**
   A: Horizontal scaling adds more instances (ASG adds EC2 behind ALB). Vertical scaling increases instance size (t3.micro to t3.large). Horizontal is preferred for HA and unlimited scaling. Vertical has instance size limits and requires downtime.

4. **What is an Internet Gateway and when do you need a NAT Gateway?**
   A: IGW allows VPC resources with public IPs to communicate with internet. NAT Gateway allows private subnet resources to initiate outbound connections while preventing inbound connections. IGW for public resources, NAT Gateway for private resources needing outbound-only access.

5. **How does AWS Shield protect against DDoS attacks?**
   A: AWS Shield Standard is free, protects against common L3/L4 attacks (SYN floods, UDP floods). Shield Advanced ($3,000/month) provides enhanced protection, DDoS cost protection, 24/7 DRT access, detailed diagnostics. For L7 protection, pair with WAF.

6. **What is the difference between an ALB, NLB, and CLB?**
   A: ALB — Layer 7, content-based routing, path/host/query routing, WebSocket, HTTP/2. NLB — Layer 4, ultra-low latency, static IP, TCP/UDP, millions of requests/sec. CLB — legacy, limited features. Use ALB for HTTP workloads, NLB for extreme performance or static IP needs.

7. **Explain the AWS Well-Architected Framework.**
   A: Six pillars: Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, Sustainability. Use the Well-Architected Tool to review workloads against these pillars.

8. **What is the difference between EBS and Instance Store?**
   A: EBS is persistent block storage — survives instance stop/terminate, can be detached, supports snapshots, up to 16 TB. Instance Store is ephemeral — physically attached to host, highest IOPS, data lost on stop/terminate. Use EBS for databases, Instance Store for caches.

9. **How do you implement blue-green deployment on AWS?**
   A: Deploy new version alongside old (two ASGs, two target groups). Update ALB listener rule to route traffic from blue to green. Run smoke tests. If healthy, keep green. If issues, switch back to blue. Use CodeDeploy for automation. For Lambda, use aliases with weighted routing.

10. **What is CloudFormation and how does it differ from Terraform?**
    A: CloudFormation is AWS-native IaC — YAML/JSON templates, drift detection, StackSets for multi-account, no state file management. Terraform is cloud-agnostic — HCL syntax, remote state backends, broader providers, more flexible logic. Choose CloudFormation for AWS-only stacks, Terraform for multi-cloud.

---

## Developer Recommendations

- **Use IAM roles over access keys** — Long-lived access keys are a top breach vector. IAM roles with STS temporary credentials (15 min-12 hours) eliminate key rotation overhead. For EC2, use instance profiles. For CI/CD, use OIDC federation. For local dev, use AWS SSO with `aws sso login`.

- **Tag everything, enforce tagging with SCPs** — Without tags, cost allocation and resource identification become impossible. Define mandatory tag set (CostCenter, Environment, Owner, Application). Enforce via SCPs that deny resource creation without required tags. Use tag-based IAM policies for fine-grained access control.

- **Use S3 Intelligent-Tiering for unpredictable access patterns** — Manually choosing storage classes leads to overpaying or unexpected costs. Intelligent-Tiering auto-moves objects between access tiers based on usage. Only extra cost is $0.0025/1,000 objects. Best for data lakes, logs, and content repositories.

- **Prefer managed services over self-managed — know the trade-off** — RDS, ElastiCache, DynamoDB reduce operational burden but cost more and reduce control. RDS costs 2-3x more than EC2 + self-managed MySQL but saves 10+ hours/week of ops. Decision: use managed where operational complexity > cost premium; self-manage where specific configs are needed.

- **Implement cost-aware development practices** — Developers should see cost implications of their choices. Use Cost Explorer with resource-level tags. Set AWS Budgets that alert the team. Implement cost gates in CI/CD — pipeline fails if estimated monthly cost increase > 10%. Use `aws-nuke` for ephemeral environments that self-destruct.

- **Design for failure from day one** — Build multi-AZ from the start, even in dev. Implement circuit breakers, retries with exponential backoff, bulkheads. Use AWS Fault Injection Simulator to test failure modes. Every dependency should have a fallback. Single-AZ production is not cost optimization — it's deferred risk.

- **Use Infrastructure as Code for everything** — CloudFormation/Terraform for resources, SSM Parameter Store for config, AppConfig for feature flags. Never manually configure SGs, IAM, or VPCs. Use linting tools (cfn-lint, tfsec, checkov) in CI/CD. Every manual change creates drift — drift is the enemy of reliability.

- **Monitor the right metrics, not everything** — Collecting every CloudWatch metric creates noise and cost. Focus on the Four Golden Signals (latency, traffic, errors, saturation) per service. Set composite alarms to reduce false positives. Use CloudWatch Contributor Insights for high-cardinality analysis.