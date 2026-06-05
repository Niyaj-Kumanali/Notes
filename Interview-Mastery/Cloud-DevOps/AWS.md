# AWS Study Guide

## 1. Executive Summary

Amazon Web Services (AWS) is the leading cloud computing platform offering 200+ services across compute, storage, databases, networking, machine learning, and more. AWS follows a pay-as-you-go model with global infrastructure spanning 30+ regions and 90+ availability zones. Key services include EC2 (compute), S3 (storage), RDS (databases), Lambda (serverless), EKS (Kubernetes), and IAM (security). Understanding AWS is critical for cloud architects, DevOps engineers, and platform engineers.

## 2. Core Theory

### 2.1 AWS Global Infrastructure

```
AWS Global Infrastructure
  +-- Region (us-east-1, eu-west-1, ap-southeast-1)
  |     +-- Availability Zone 1a
  |     +-- Availability Zone 1b
  |     +-- Availability Zone 1c
  |     +-- Edge Locations (CDN)
  |
Each region >= 3 AZs, isolated with low-latency links
```

### 2.2 Shared Responsibility Model

```
Customer responsible FOR the cloud:
  - Customer data, platform, apps, IAM
  - OS, network, firewall configuration
  - Client-side encryption

AWS responsible OF the cloud:
  - Hardware, software, networking
  - Facilities, physical security
  - Hypervisor, compute, storage, database
```

### 2.3 Core Service Categories

| Category | Services |
|----------|----------|
| Compute | EC2, Lambda, ECS, EKS, Fargate, Batch |
| Storage | S3, EBS, EFS, Glacier, Storage Gateway |
| Database | RDS, DynamoDB, Aurora, Redshift, ElastiCache |
| Networking | VPC, CloudFront, Route 53, ELB, API Gateway |
| Security | IAM, KMS, Shield, WAF, Cognito, GuardDuty |
| DevOps | CodePipeline, CodeBuild, CloudFormation |
| Monitoring | CloudWatch, X-Ray, CloudTrail, Config |

## 3. Under-the-Hood Deep Dive

### 3.1 VPC Architecture

```yaml
VPC:
  CIDR: 10.0.0.0/16
  Subnets:
    Public:  [10.0.1.0/24 (AZ-a), 10.0.2.0/24 (AZ-b)]
    Private: [10.0.3.0/24 (AZ-a), 10.0.4.0/24 (AZ-b)]
    DB:      [10.0.5.0/24 (AZ-a), 10.0.6.0/24 (AZ-b)]
  Gateways:
    - Internet Gateway (public subnets)
    - NAT Gateway (private subnets egress)
    - VPC Peering / Transit Gateway
```

### 3.2 EC2 Instance Families

| Family | Use Case | Examples |
|--------|----------|----------|
| General | Web servers, dev | t3, m6i, m7g |
| Compute | Batch, HPC | c6i, c7g, hpc7a |
| Memory | DBs, caching | r6i, r7g, x2iedn |
| Storage | Data warehousing | i3, i4i, d3 |
| GPU | ML, rendering | p4d, p5, g5 |

### 3.3 S3 Storage Classes

| Class | Durability | Availability | Retrieval | Use Case |
|-------|-----------|-------------|-----------|----------|
| STANDARD | 11 9s | 99.99% | Immediate | Frequent access |
| INTELLIGENT_TIERING | 11 9s | 99.9% | Immediate | Auto-cost optimization |
| STANDARD_IA | 11 9s | 99.9% | Immediate | Infrequent access |
| ONEZONE_IA | 11 9s | 99.5% | Immediate | Non-critical |
| GLACIER | 11 9s | 99.99% | Minutes | Archival |
| DEEP_ARCHIVE | 11 9s | 99.99% | Hours | Long-term archival |

## 4. Production Code Examples

### 4.1 CloudFormation VPC

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Production VPC with public and private subnets'

Parameters:
  EnvironmentName:
    Type: String
    Default: production
    AllowedValues: [production, staging, development]

Mappings:
  EnvironmentConfig:
    production:
      instanceType: t3.small
      desiredCapacity: 3
    staging:
      instanceType: t3.micro
      desiredCapacity: 2

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-vpc"

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-igw"

  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs ""]
      MapPublicIpOnLaunch: true

  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [0, !GetAZs ""]

  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      VPCZoneIdentifier: [!Ref PrivateSubnet1]
      LaunchConfigurationName: !Ref LaunchConfig
      MinSize: !FindInMap [EnvironmentConfig, !Ref EnvironmentName, desiredCapacity]
      MaxSize: 10
      DesiredCapacity: !FindInMap [EnvironmentConfig, !Ref EnvironmentName, desiredCapacity]
      HealthCheckType: ELB
      TargetGroupARNs: [!Ref TargetGroup]

  LaunchConfig:
    Type: AWS::AutoScaling::LaunchConfiguration
    Properties:
      ImageId: ami-0c55b159cbfafe1f0
      InstanceType: !FindInMap [EnvironmentConfig, !Ref EnvironmentName, instanceType]
      SecurityGroups: [!Ref WebSecurityGroup]

  WebSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web server SG
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0

  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Subnets: [!Ref PublicSubnet1]
      SecurityGroups: [!Ref WebSecurityGroup]
      Scheme: internet-facing
      Type: application

  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Port: 80
      Protocol: HTTP
      VpcId: !Ref VPC
      HealthCheckPath: /health
      HealthCheckIntervalSeconds: 30

Outputs:
  LoadBalancerDNS:
    Value: !GetAtt ApplicationLoadBalancer.DNSName
```

### 4.2 Terraform for AWS

```hcl
provider "aws" {
  region = var.region
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.1"

  name = "${var.environment}-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["${var.region}a", "${var.region}b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.3.0/24", "10.0.4.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = var.environment != "production"

  tags = { Environment = var.environment }
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "19.16.0"

  cluster_name    = "${var.environment}-eks"
  cluster_version = "1.28"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  eks_managed_node_groups = {
    main = {
      desired_size   = var.environment == "production" ? 5 : 2
      min_size       = var.environment == "production" ? 3 : 1
      max_size       = 10
      instance_types = ["t3.medium"]
    }
  }
}

resource "aws_s3_bucket" "app_data" {
  bucket        = "${var.environment}-app-data-${data.aws_caller_identity.current.account_id}"
  force_destroy = var.environment != "production"
}

resource "aws_s3_bucket_versioning" "app_data" {
  bucket = aws_s3_bucket.app_data.id
  versioning_configuration {
    status = var.environment == "production" ? "Enabled" : "Suspended"
  }
}

resource "aws_db_instance" "main" {
  identifier     = "${var.environment}-db"
  engine         = "postgres"
  engine_version = "16.1"
  instance_class = var.environment == "production" ? "db.r6g.large" : "db.t3.small"

  allocated_storage     = var.environment == "production" ? 100 : 20
  max_allocated_storage = 500

  db_name  = "app"
  username = "appuser"
  password = random_password.db_password.result

  backup_retention_period = var.environment == "production" ? 30 : 7
  storage_encrypted       = true
  deletion_protection     = var.environment == "production"
}

resource "random_password" "db_password" {
  length  = 24
  special = false
}
```

### 4.3 Lambda with API Gateway (SAM)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 10
    MemorySize: 256
    Runtime: python3.12
    Tracing: Active

Resources:
  ApiGateway:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      Auth:
        DefaultAuthorizer: CognitoAuthorizer
        Authorizers:
          CognitoAuthorizer:
            UserPoolArn: !GetAtt CognitoUserPool.Arn
      MethodSettings:
        - LoggingLevel: INFO
          MetricsEnabled: true
          ResourcePath: "/*"
          HttpMethod: "*"

  GetItemFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: handlers.get_item
      Events:
        GetItem:
          Type: Api
          Properties:
            RestApiId: !Ref ApiGateway
            Path: /items/{id}
            Method: GET
      Policies:
        - DynamoDBReadPolicy:
            TableName: !Ref DataTable

  DataTable:
    Type: AWS::DynamoDB::Table
    Properties:
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
        - AttributeName: createdAt
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
        - AttributeName: createdAt
          KeyType: RANGE
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: true
      SSESpecification:
        SSEEnabled: true
```

### 4.4 IAM Policy

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
    },
    {
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:Query"],
      "Resource": "arn:aws:dynamodb:*:*:table/MyAppTable*"
    },
    {
      "Effect": "Deny",
      "Action": "iam:*",
      "Resource": "*"
    }
  ]
}
```

## 5. Real-World Scenarios

### 5.1 Multi-AZ HA

```yaml
high_availability:
  compute:
    - ASG with min 2 instances across 2 AZs
    - ALB cross-zone enabled
    - Spot instances with on-demand fallback
  database:
    - RDS Multi-AZ synchronous standby
    - Read replicas for scaling
    - ElastiCache cluster mode
  storage:
    - S3 with cross-region replication
    - EBS snapshots via lifecycle manager
```

### 5.2 Disaster Recovery

```yaml
disaster_recovery:
  strategy: Pilot Light
  RPO: 15 minutes
  RTO: 1 hour
  replication:
    - RDS cross-region read replica
    - S3 cross-region replication
    - DynamoDB global tables
  failover:
    - Route 53 DNS failover
    - Auto Scaling in DR region
    - CloudFormation infrastructure
  testing:
    - Quarterly DR drills
    - Documented runbook
```

## 6. Performance

### 6.1 EC2 Optimization

```bash
# Right-sizing with CloudWatch
aws cloudwatch get-metric-statistics \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-02-01T00:00:00Z \
  --period 3600 --statistics Average \
  --dimensions Name=InstanceId,Values=i-1234567890abcdef0
```

### 6.2 Cost Optimization

| Strategy | Savings | Effort |
|----------|---------|--------|
| Spot instances | 60-90% | Low |
| Reserved instances | 30-72% | Medium |
| Savings Plans | 30-66% | Low |
| Auto Scaling | Variable | Medium |
| S3 Lifecycle policies | 40-80% | Low |
| Right-sizing | 20-50% | Medium |

## 7. Security

### 7.1 IAM Best Practices

- Least privilege principle
- Use roles, not users for applications
- Enable MFA for all users
- Use IAM Access Analyzer
- Regular access key rotation
- Use managed policies where possible

### 7.2 Encryption

```yaml
encryption:
  in_transit:
    - TLS 1.2+ for all endpoints
    - VPC peering encryption
  at_rest:
    - S3 SSE-S3, SSE-KMS, SSE-C
    - EBS encrypted by default
    - RDS encryption at rest
  key_management:
    - AWS KMS with automatic rotation
    - Separate keys per environment
```

## 8. Common Mistakes

| Mistake | Impact | Solution |
|---------|--------|----------|
| Default VPC | Security risk | Custom VPC |
| Open security groups (0.0.0.0/0) | Vulnerable | Restrict by IP/CIDR |
| No IAM roles for EC2 | Credential mgmt | Instance profiles |
| No CloudTrail | No audit trail | Enable all regions |
| Single-AZ | No HA | Multi-AZ minimum |
| No ASG | Manual scaling | ASG with ELB |
| S3 bucket public | Data exposure | Block public access |
| Root account daily use | Security risk | IAM users |
| No cost budgets | Bill shock | AWS Budgets |
| Over-provisioned | Wasted money | Right-size |

## 9. Senior Engineer Perspective

### 9.1 Architecture Patterns

```yaml
web_application:
  compute: ECS Fargate + ALB
  database: Aurora Serverless + ElastiCache
  storage: S3 + CloudFront
  auth: Cognito
  api: API Gateway + Lambda

microservices:
  compute: EKS (Kubernetes)
  messaging: SQS + SNS + EventBridge
  api: API Gateway + VPC Link
  monitoring: Prometheus + Grafana

data_pipeline:
  ingestion: Kinesis
  processing: Glue ETL
  storage: S3 data lake
  query: Athena + Redshift
  orchestration: Step Functions
```

## 10. Interview Questions (Easy)

1. What is AWS and what services does it offer?
2. What is the difference between a Region and AZ?
3. What is EC2 and how is it used?
4. What is S3 and its storage classes?
5. What is IAM?
6. What is a VPC?
7. Security group vs NACL?
8. What is Elastic Load Balancing?
9. What is Auto Scaling?
10. What is the AWS Free Tier?

## 10. Interview Questions (Medium)

11. How does the shared responsibility model work?
12. EC2 vs Lambda differences?
13. Design a multi-tier architecture on AWS.
14. CloudFormation vs Terraform?
15. How do you secure an S3 bucket?
16. RDS vs DynamoDB?
17. Route 53 routing policies?
18. NAT Gateway vs Internet Gateway?
19. How do you monitor AWS resources?
20. EBS vs EFS?

## 11. Advanced Interview Questions (Hard)

1. Design multi-region active-active architecture.
2. Migrate 10TB database with minimal downtime.
3. Serverless data pipeline handling millions of events/day.
4. Implement HIPAA compliance on AWS.
5. Cost-optimized ML training infrastructure.
6. Architect PCI DSS compliant system.
7. DR solution with RPO <5 min, RTO <15 min.
8. Cross-account access in multi-account org.
9. Global content delivery with CloudFront.
10. Blue-green deployments on ECS/EKS.

## 11. Advanced Interview Questions (System Design)

11. Multi-tenant SaaS platform on AWS.
12. Real-time analytics for IoT data.
13. Data lake architecture on AWS.
14. Hybrid cloud with AWS and on-premise.
15. CI/CD platform using AWS Code services.
16. Real-time chat application.
17. Video transcoding pipeline.
18. Financial trading system with millisecond latency.
19. Global auth system with Cognito.
20. Cost monitoring and optimization system.

## 12. Expert-Level Interview Questions (Architect)

1. Design a multi-region, multi-account AWS org for Fortune 500 with 500+ apps, 50+ business units, and PCI/HIPAA/SOC2/FedRAMP compliance.

2. Architect a serverless platform processing 1M events/second with ordering, exactly-once, and sub-second latency.

3. Design global network across 5 regions with Direct Connect, VPN failover, transit gateway, centralized inspection.

4. Zero-trust security model spanning accounts, VPCs, services, and on-premise with continuous verification.

5. Cost management platform auto-optimizing 10,000+ instances across 500+ accounts with real-time allocation.

6. Data platform handling 100PB+ with serverless querying, automated lifecycle, and cross-region replication.

7. AWS Control Tower customizations enforcing security policies across 1,000+ accounts.

8. Migration strategy for 10,000 servers from on-premise to AWS with minimal disruption.

9. Observability platform for 10,000+ microservices across 100+ accounts with real-time anomaly detection.

10. Chaos engineering platform safely testing resilience across accounts, regions, and services.

## 13. Debugging & Troubleshooting

### 13.1 Common Issues

```bash
# EC2 connection issues
aws ec2 describe-instance-status --instance-ids i-123
aws ec2 get-console-output --instance-id i-123
aws ec2 reboot-instances --instance-ids i-123

# S3 access denied
aws s3api get-bucket-policy --bucket my-bucket
aws s3api get-public-access-block --bucket my-bucket

# VPC connectivity
aws ec2 describe-route-tables --filters Name=vpc-id,Values=vpc-123

# Lambda errors
aws logs filter-log-events \
  --log-group-name /aws/lambda/my-function \
  --filter-pattern "ERROR"
```

## 14. Comparison Section

### AWS vs Azure vs GCP

| Feature | AWS | Azure | GCP |
|---------|-----|-------|-----|
| Compute | EC2, Lambda | VMs, Functions | GCE, Cloud Functions |
| Containers | ECS, EKS, Fargate | AKS, Container Instances | GKE, Cloud Run |
| Serverless | Lambda + API GW | Functions + API Mgmt | Cloud Functions |
| Database | RDS, DynamoDB, Aurora | SQL DB, Cosmos DB | Cloud SQL, Firestore, Spanner |
| Storage | S3, EBS, EFS | Blob, Disk, Files | Cloud Storage, PD |
| Regions | 30+ | 60+ | 40+ |
| Market Share | ~32% | ~23% | ~11% |

## 15. Revision Notes

```
AWS INFRASTRUCTURE
Region >= 3 AZs >= 1 Edge Location

CORE SERVICES
Compute: EC2, Lambda, ECS, EKS, Fargate
Storage: S3, EBS, EFS, Glacier
Database: RDS, DynamoDB, Aurora, ElastiCache
Network: VPC, CloudFront, Route 53, ELB
Security: IAM, KMS, WAF, Shield, GuardDuty

WELL-ARCHITECTED FRAMEWORK
1. Operational Excellence  2. Security
3. Reliability             4. Performance Efficiency
5. Cost Optimization       6. Sustainability
```

## 16. Cheat Sheet

```text
+======================================================================+
|                      AWS CHEAT SHEET                                 |
+======================================================================+

  CORE SERVICES
+----------------------------------------------------------------------+
| EC2           | Virtual machines                                     |
| Lambda        | Serverless functions                                 |
| S3            | Object storage (11 9s durability)                    |
| RDS           | Managed relational databases                         |
| DynamoDB      | Managed NoSQL database                               |
| VPC           | Isolated cloud network                               |
| IAM           | Identity and access management                       |
| CloudFront    | CDN                                                  |
| Route 53      | DNS service                                          |
| CloudWatch    | Monitoring and logging                               |
+----------------------------------------------------------------------+

  EC2 PURCHASE OPTIONS
+----------------------------------------------------------------------+
| On-Demand     | Pay per hour, no commitment                          |
| Reserved      | 1-3 year, up to 72% savings                         |
| Savings Plan  | Flexible compute, 1-3 year                          |
| Spot          | Up to 90% off, can be terminated                    |
| Dedicated     | Physical server isolation                           |
+----------------------------------------------------------------------+

  S3 STORAGE CLASSES
+----------------------------------------------------------------------+
| STANDARD      | Frequent, 99.99% avail                              |
| IA            | Infrequent, lower cost                              |
| ONEZONE_IA    | Single AZ, non-critical                             |
| INTELLIGENT   | Auto-tiering by access                             |
| GLACIER       | Archival, min retrieval                            |
| DEEP_ARCHIVE  | Long-term, hour retrieval                          |
+----------------------------------------------------------------------+

  IAM BEST PRACTICES
+----------------------------------------------------------------------+
| - Least privilege                                                    |
| - Roles not users for apps                                           |
| - MFA for all human users                                            |
| - Regular access key rotation                                        |
| - IAM Access Analyzer                                                |
| - Service control policies (SCPs)                                    |
| - CloudTrail for audit                                               |
+----------------------------------------------------------------------+

  WELL-ARCHITECTED PILLARS
+----------------------------------------------------------------------+
| 1. Operational Excellence | Automate, monitor                        |
| 2. Security               | Protect data, manage access             |
| 3. Reliability            | Recover, scale, handle failures        |
| 4. Performance Efficiency | Right-size, optimize                    |
| 5. Cost Optimization      | Pay for what you use                   |
| 6. Sustainability         | Minimize environmental impact          |
+----------------------------------------------------------------------+

  COMMON CLI
+----------------------------------------------------------------------+
| aws ec2 describe-instances           | List EC2                       |
| aws s3 ls s3://bucket                | List S3 bucket                 |
| aws s3 sync local/ s3://bucket       | Sync to S3                    |
| aws rds describe-db-instances        | List databases                 |
| aws lambda invoke --function-name fn | Invoke Lambda                 |
| aws cloudformation deploy            | Deploy CloudFormation         |
+----------------------------------------------------------------------+

+======================================================================+
|  PRO TIPS: Tag everything for cost allocation. Enable termination    |
|  protection. Use Config rules for compliance. Set up AWS Budgets.   |
|  Use IaC (CloudFormation/Terraform). Enable S3 versioning.          |
+======================================================================+
```
