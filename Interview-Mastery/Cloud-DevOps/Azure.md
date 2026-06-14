# Azure

---

## Overview

- **Definition:** Microsoft Azure is the second-largest cloud platform by market share, offering over 200 services across compute, storage, databases, artificial intelligence, machine learning, IoT, and DevOps with deep enterprise integration.
  - Operates on a pay-as-you-go pricing model with reserved instances, spot pricing, and hybrid benefit discounts
  - Serves Fortune 500 enterprises, government agencies, and startups with particular strength in regulated industries
  - Comprehensive compliance portfolio covering over 100 offerings including SOC, ISO, PCI DSS, HIPAA, and FedRAMP
  - Tight integration with Microsoft ecosystem — Active Directory, Office 365, Dynamics 365, SQL Server, and .NET framework

- **Why It Exists:** Azure provides the largest regional footprint of any cloud provider with more than 60 regions globally, deep integration with the Microsoft software ecosystem, and hybrid cloud capabilities.
  - Enterprises running Microsoft workloads had limited cloud migration options before Azure — running SQL Server and Windows Server on AWS at a licensing disadvantage or building private clouds
  - Azure solved this with native Active Directory federation, SQL Server managed instances, .NET hosting, and licensing portability through Azure Hybrid Benefit
  - Hybrid cloud strategy anchored by Azure Arc, Azure Stack Hub, and Azure Stack Edge enables organizations to adopt cloud services at their own pace

- **Key Concepts:**
  - **Regions** — Azure operates more than 60 regions globally, each paired with another region within the same geography for disaster recovery and planned maintenance
  - **Availability Zones** — Physically separate locations within a region with independent power, cooling, and networking; most regions offer three zones with 99.99% SLA
  - **Region Pairs** — Each region is paired with another in the same geography at least 300 miles away for built-in disaster recovery; Azure serializes platform updates across pairs
  - **Resource Hierarchy** — Management Group > Subscription > Resource Group > Resource; management groups enable enterprise-scale policy and RBAC inheritance
  - **Azure Resource Manager (ARM)** — Unified deployment and management layer with consistent APIs, RBAC, tagging, and policy enforcement across services

---

## Core Services

- **Virtual Machines (Compute):** Azure Virtual Machines provide IaaS compute capacity for any workload from legacy enterprise applications to modern distributed systems.
  - General Purpose D-series and B-series provide balanced CPU-to-memory ratios for development and web servers
  - Compute-optimized F-series deliver high CPU-to-memory ratios for batch processing and gaming servers
  - Memory-optimized E-series and M-series support large in-memory databases with M-series offering up to 12 TB of RAM
  - Storage-optimized L-series and Ebsv5 provide high disk throughput and IOPS for data warehouses and big data analytics
  - GPU-optimized NC-series and NV-series provide NVIDIA GPU acceleration for ML training, HPC, and virtual desktop infrastructure
  - VM Scale Sets (VMSS) enable automatic scaling based on CPU, memory, or custom metrics with scheduled and predictive autoscale
  - Availability Sets distribute VMs across fault domains (physical racks) and update domains (planned maintenance) for 99.95% SLA; Availability Zones provide 99.99% SLA

- **App Service (PaaS):** Azure App Service is a fully managed PaaS offering for hosting web applications, REST APIs, and mobile backends with built-in auto-scaling, deployment slots, and CI/CD.
  - Supports ASP.NET, ASP.NET Core, Java, Node.js, Python, PHP, and Ruby with automatic patching
  - Tiers from Free/Shared through Basic/Standard/Premium to Isolated (dedicated App Service Environments in VNet)
  - Deployment slots provide zero-downtime deployments with staging validation and instant swap
  - Slot-sticky settings (connection strings, environment variables) remain with their slots during swap
  - Built-in authentication providers, custom domains with TLS/SSL, and automatic backup and restore

- **AKS (Containers):** Azure Kubernetes Service is a managed Kubernetes offering handling master node management, health monitoring, maintenance, and upgrades.
  - Integrates with Azure AD for Kubernetes RBAC authentication using existing corporate credentials
  - Managed identities allow pods to authenticate to Key Vault, Azure SQL, and Storage without managing secrets
  - Azure Container Registry (ACR) provides private image storage with geo-replication and integrated security scanning
  - Advanced networking with Azure CNI for pod-level VNet integration and Network Policies for micro-segmentation
  - Cluster autoscaling and horizontal pod autoscaler; GitOps workflows through Flux v2

- **Azure Functions (Serverless):** Azure Functions provides serverless compute where code executes in response to events without infrastructure management.
  - Trigger types: HTTP requests, Queue storage, Timer, Cosmos DB change feed, Event Hub, Service Bus, Blob storage events
  - Supported languages: C#, JavaScript, Python, Java, PowerShell, TypeScript, and Go
  - Consumption plan (pay-per-execution, 1M free executions/month), Premium plan (no cold starts, 60-minute timeout), Dedicated plan (on existing App Service plan)
  - Durable Functions provides stateful orchestration with fan-out/fan-in, function chaining, and human interaction workflows

- **Azure SQL Database (Databases):** Azure SQL Database is a fully managed relational database with built-in high availability (99.99% SLA), automated backups, and serverless compute.
  - Service tiers: General Purpose (balanced), Business Critical (lowest latency, zone-redundant), Hyperscale (up to 100 TB)
  - Advanced data security: Transparent Data Encryption, Always Encrypted, Dynamic Data Masking, Azure Defender for SQL
  - Elastic pools for cost-effective management of databases with variable usage patterns
  - SQL Managed Instance provides near 100% on-premise compatibility for lift-and-shift migrations
  - Cosmos DB complements Azure SQL for globally distributed NoSQL workloads with multiple APIs (SQL, MongoDB, Cassandra, Gremlin, Table)

- **Blob Storage (Storage):** Azure Blob Storage provides massively scalable object storage for unstructured data with three resource types: storage accounts, containers, and blobs.
  - Redundancy: LRS (3 copies in single datacenter), ZRS (3 copies across zones), GRS (LRS + 3 copies in paired region), RA-GRS (adds read access to secondary)
  - Storage tiers: Hot (frequent access), Cool (30-day minimum), Cold (90-day minimum), Archive (180-day minimum, hours retrieval)
  - Lifecycle management policies automatically move data between tiers based on age
  - Data Lake Storage Gen2 adds hierarchical namespace for POSIX-compatible access control for analytics workloads

- **Virtual Network (Networking):** Azure Virtual Network provides isolated networking enabling private IP address spaces, subnet segmentation, and hybrid connectivity.
  - Network Security Groups (NSGs) provide distributed stateful firewalling at subnet and NIC level
  - Application Security Groups (ASGs) simplify NSG management by grouping VMs by application role
  - VNet Peering connects VNets within same region or across regions using Microsoft's backbone network
  - VPN Gateway provides encrypted site-to-site and point-to-site connectivity; ExpressRoute offers dedicated private connectivity from 50 Mbps to 100 Gbps
  - Azure Firewall provides managed stateful firewall-as-a-service with built-in HA, auto-scaling, and Azure Monitor integration
  - Azure Front Door provides global Layer 7 load balancing with SSL offloading, WAF, URL-based routing, and caching

```yaml
hub-and-spoke:
  hub:
    - Azure Firewall
    - Azure Bastion
    - Log Analytics Workspace
    - Key Vault
    - ExpressRoute/VPN Gateway
  spokes:
    production:
      VNet: 10.1.0.0/16
      Services: AKS, App Service, Azure SQL
    development:
      VNet: 10.2.0.0/16
      Services: VM Scale Set, dev databases
```

The hub-and-spoke topology is the most widely adopted networking pattern for enterprise Azure deployments. The hub VNet acts as a central connectivity point to on-premise networks through ExpressRoute or VPN Gateway and hosts shared services. Azure Firewall inspects all traffic between spokes and to the internet. Azure Bastion provides secure RDP/SSH access without public IPs. Key Vault stores secrets centrally. Each spoke connects to the hub through VNet peering.

---

## Common Mistakes

- **Using the default VNet for production workloads** — Default VNets have generic CIDR ranges that can overlap with on-premise networks, making hybrid connectivity impossible without complex NAT configurations.
  - **Why it looks correct:** The default VNet is created automatically and works out of the box — if it weren't suitable, Azure wouldn't create it by default.

- **Opening NSG rules to 0.0.0.0/0 for management ports** — Allowing inbound RDP (3389) or SSH (22) from any source IP exposes infrastructure to continuous scanning and brute force attacks.
  - **Why it looks correct:** Opening port 22 to the internet is the simplest way to SSH into a VM, and the constant scanning from foreign IPs isn't visible unless you check the NSG logs.

- **Failing to configure cost alerts and budgets** — Without proactive monitoring, organizations experience bill shock from GPU-intensive VMs left running over weekends or misconfigured autoscale policies.
  - **Why it looks correct:** Azure resources are provisioned as needed, and costs seem straightforward — the bill shock only comes at the end of the month.

- **Deploying resources in a single region without disaster recovery planning** — A regional outage due to natural disasters, power failures, or network issues can render the entire application unavailable.
  - **Why it looks correct:** Deploying in one region is simpler and cheaper, and Azure's 99.99% SLA seems sufficient — until a power failure takes out an entire region.

- **Operating without Azure RBAC** — Without role-based access control, users receive excessive permissions through subscription-level Contributor roles or shared administrative credentials.
  - **Why it looks correct:** Giving everyone Contributor access is the fastest way to get them working until someone accidentally runs a destructive command on production.

- **Not implementing resource tagging** — Without mandatory tags, organizations cannot track cloud spending by cost center, department, project, or environment.
  - **Why it looks correct:** Tags feel like metadata with no functional impact — resources work fine without them, so tagging seems like bureaucratic overhead.

- **Exposing PaaS services through public endpoints** — Azure SQL, Storage, Key Vault, and Cosmos DB are often deployed with public network access enabled by default.
  - **Why it looks correct:** PaaS services have built-in authentication and TLS, so a public endpoint behind a strong password seems adequately protected.

- **Running workloads without autoscaling configured** — Without autoscaling, workloads are either over-provisioned (wasting money) or under-provisioned (degrading performance during spikes).
  - **Why it looks correct:** Manually setting instance counts gives precise control and predictable costs — autoscaling feels like giving up control to an algorithm.

- **Managing credentials manually instead of using managed identities** — Service principals require manual creation, rotation, and secure storage of secrets.
  - **Why it looks correct:** Creating a service principal and pasting its secret into the application config is the standard approach — the security risk isn't obvious until it leaks.

- **Deploying resources without Azure Policy governance** — Organizations that skip Azure Policy experience configuration drift where teams violate security baselines and compliance requirements.
  - **Why it looks correct:** Policies restrict developers and seem like an obstacle to agility until an audit reveals widespread compliance violations.

---

## Key Design Considerations

- **Landing Zone** — An Azure Landing Zone is a scalable, governance-compliant environment following Cloud Adoption Framework best practices.
  - Management group hierarchy organizing subscriptions into Platform and Workloads structures
  - Subscription vending automation through Terraform or ARM templates for self-service provisioning
  - Platform management group contains subscriptions for connectivity, identity, and management
  - Workload subscriptions connect to platform through hub-spoke networking with forced tunneling
  - Must account for regulatory compliance, geographic distribution, and organizational structure

- **High Availability** — Azure provides multiple layers of HA that should be combined to meet application uptime requirements.
  - Availability Sets (99.95% SLA): distribute VMs across fault and update domains within a single datacenter
  - Availability Zones (99.99% SLA): distribute resources across three physically separated locations
  - Zone-redundant services: Azure SQL Business Critical, Premium Blob Storage automatically replicate across zones
  - Multi-region DR: Azure Site Recovery (RPO as low as 30 seconds), Azure SQL active geo-replication
  - Azure Front Door provides global load balancing with health probe-based failover within 10 seconds

- **Hybrid Cloud** — Azure's hybrid capabilities are the most comprehensive among major cloud providers.
  - Azure Arc extends ARM-based management to any infrastructure — on-premise, multi-cloud, and edge
  - Arc-enabled Kubernetes provides GitOps configuration management and Azure Policy enforcement
  - ExpressRoute provides dedicated private connectivity from 50 Mbps to 100 Gbps with 99.95% availability
  - Azure Stack Hub provides on-premise Azure services; Azure Stack Edge brings Azure to remote locations
  - Azure File Sync enables caching Azure file shares on Windows Servers for low-latency access

- **Security** — Azure follows a defense-in-depth strategy across identity, network, data, and application layers.
  - Azure AD Conditional Access evaluates user, device, location, and risk signals for granular access controls
  - Managed Identities eliminate credential management with automatically rotated Azure AD identities
  - Microsoft Defender for Cloud provides unified security management with vulnerability scanning and JIT VM access
  - Azure Policy enforces security configurations with automated remediation of non-compliant resources
  - Network security layers: NSGs, Azure Firewall, Private Link for securing PaaS access
  - Microsoft Sentinel provides cloud-native SIEM with built-in security analytics
  - Azure Key Vault centrally manages encryption keys, secrets, and certificates with HSM support

- **Cost Optimization** — Azure offers multiple pricing and purchasing options to reduce costs when properly configured.
  - Reserved Instances: 30-72% discount for 1-3 year commitment with size flexibility
  - Spot VMs: 60-90% discount for interruptible workloads like batch processing and CI/CD
  - Azure Hybrid Benefit: 40-50% savings on Windows Server and SQL Server by applying existing licenses
  - Azure Advisor provides personalized cost optimization recommendations
  - Auto-shutdown policies for non-production environments enforced through Azure Policy
  - Azure Budgets with action groups provide automated responses to cost anomalies

- **Well-Architected Framework** — The Microsoft Azure Well-Architected Framework provides architectural best practices organized into five pillars.
  - Cost Optimization: right-sizing, reserved capacity, consumption-based model
  - Operational Excellence: infrastructure as code, comprehensive observability, safe deployment practices
  - Performance Efficiency: appropriate resource selection, horizontal scaling, caching, performance testing
  - Reliability: design for failure, graceful degradation, automated health probes and failover, DR testing
  - Security: least privilege, defense in depth, encryption at rest and in transit, continuous monitoring
  - Each pillar includes review questions assessed quarterly for critical workloads

---

## Real-World Scenarios

- **Scenario 1: Enterprise Migration from On-Premise to Azure**
  - **Context:** A financial services company needs to migrate 200+ workloads from on-premise datacenters to Azure while maintaining regulatory compliance, minimizing downtime, and ensuring data integrity.
  - **Assessment phase:** Use Azure Migrate to discover servers, configurations, dependencies, and performance metrics. Dependency visualization maps inter-server communications. Performance data collected over two weeks provides accurate sizing recommendations.
  - **Migration phase:** Use Azure Site Recovery for continuous VM replication with 30-second RPO. ExpressRoute provides dedicated 1 Gbps connectivity. Each wave begins with test failover to validate functionality. Cutover stops writes on source, performs final sync, fails over, updates DNS, and validates application health.
  - **Post-migration:** Use Azure Cost Management to track ROI. Azure Advisor provides right-sizing recommendations. Landing Zone architecture with management groups, Azure Policy, and RBAC ensures consistent governance.

```powershell
# Step 1: Create a Recovery Services Vault for Azure Site Recovery
$asrVault = New-AzRecoveryServicesVault `
  -Name "ASR-Vault-Prod" `
  -ResourceGroupName "Migration-RG" `
  -Location "EastUS"

# Step 2: Configure the vault context for subsequent operations
Set-ASRVaultContext -Vault $asrVault

# Step 3: Enable replication for each VM in the migration wave
$vm = Get-AzVM -ResourceGroupName "OnPrem-RG" -Name "WebServer01"
New-AzRecoveryServicesAsrProtectionContainerMapping `
  -Name "PrimaryToDR" `
  -Policy $asrPolicy `
  -PrimaryProtectionContainer $primaryContainer `
  -RecoveryProtectionContainer $recoveryContainer

# Step 4: Perform a test failover in an isolated network
$testFailoverJob = Start-AzRecoveryServicesAsrTestFailoverJob `
  -RecoveryPlan $recoveryPlan `
  -Direction "PrimaryToRecovery" `
  -AzureVMNetworkId $drVnet.Id

# Step 5: Monitor the test failover job and clean up when complete
Get-AzRecoveryServicesAsrJob -Name $testFailoverJob.Name
```

- **Scenario 2: Real-Time Fraud Detection with Azure Stream Analytics**
  - **Context:** A payment processing company processing 50,000 transactions per second needs to detect fraud in real-time with end-to-end latency under 100 milliseconds and 99.99% availability.
  - **Ingestion:** Azure Event Hubs with auto-inflate throughput units handles traffic spikes up to 200,000 events/second during peak periods like Black Friday.
  - **Processing:** Azure Stream Analytics uses sliding window aggregations over 10-second windows to detect velocity-based fraud patterns. Queries join incoming events against reference data from Azure SQL Database containing blacklisted merchants, suspicious IP ranges, and compromised card numbers.
  - **Output sinks:** Power BI for real-time dashboards, Azure SQL Database for alert persistence, Azure Functions for webhook notifications to security operations center.

```sql
-- Azure Stream Analytics query for real-time fraud detection
WITH GeoVelocityCheck AS (
  SELECT
    TransactionId,
    CardNumber,
    MerchantCity,
    EventEnqueuedUtcTime,
    LAG(EventEnqueuedUtcTime) OVER (
      PARTITION BY CardNumber
      LIMIT DURATION(minute, 10)
    ) AS PreviousTransactionTime,
    LAG(MerchantCity) OVER (
      PARTITION BY CardNumber
      LIMIT DURATION(minute, 10)
    ) AS PreviousMerchantCity
  FROM PaymentEventHub TIMESTAMP BY EventEnqueuedUtcTime
)
SELECT
  TransactionId,
  CardNumber,
  MerchantCity,
  EventEnqueuedUtcTime,
  'VelocityAnomaly' AS FraudType,
  DATEDIFF(minute, PreviousTransactionTime, EventEnqueuedUtcTime) AS MinutesSinceLastTx
INTO FraudAlertOutput
FROM GeoVelocityCheck
WHERE
  PreviousMerchantCity IS NOT NULL
  AND PreviousMerchantCity != MerchantCity
  AND DATEDIFF(minute, PreviousTransactionTime, EventEnqueuedUtcTime) <= 10
```

- **Scenario 3: Multi-Tenant SaaS Platform on AKS**
  - **Context:** A SaaS company building a multi-tenant enterprise application requires tenant isolation, per-tenant cost tracking, zero-downtime deployments, and compliance with data residency requirements across multiple geographic regions.
  - **Tenant isolation:** AKS uses a namespace-per-tenant architecture with ResourceQuota, Network Policies, and RBAC preventing cross-tenant access. Azure Policy for AKS enforces resource requests and limits.
  - **Authentication:** Azure AD B2C handles tenant authentication with tenant-specific branding, self-service registration, social identity providers, and enterprise federation via SAML and OpenID Connect.
  - **Cost tracking:** Mandatory labels on all Kubernetes resources identify tenant and environment, enabling accurate cost allocation. Helm charts parameterize tenant-specific configurations. Flux v2 implements GitOps for continuous deployment.

```yaml
# Kubernetes ResourceQuota for tenant namespace isolation
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  namespace: tenant-abc-corp
spec:
  hard:
    requests.cpu: "8"
    requests.memory: "32Gi"
    limits.cpu: "16"
    limits.memory: "64Gi"
    requests.nvidia.com/gpu: "0"
    persistentvolumeclaims: "5"
    pods: "20"
---
# NetworkPolicy to isolate tenant namespace traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-isolation
  namespace: tenant-abc-corp
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: tenant-abc-corp
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
      ports:
        - protocol: TCP
          port: 443
```

## Use Cases

- **Enterprise application hosting** — running line-of-business applications with Azure App Service or AKS
  - App Service provides managed hosting with auto-scaling, staging slots, and integrated CI/CD. AKS provides Kubernetes orchestration for containerized microservices.
  - **Avoid when:** the application is a simple static site — Azure Static Web Apps or Storage static website hosting is cheaper and simpler.

- **Hybrid cloud with on-premise integration** — extending on-premise data centers to the cloud
  - Azure Arc provides unified management across on-prem and cloud. ExpressRoute establishes dedicated private connections. Azure Stack HCI runs Azure services on-prem.
  - **Avoid when:** all workloads can migrate to the cloud — a fully cloud-native architecture avoids hybrid complexity.

- **Data and AI workloads** — building machine learning pipelines, data warehouses, and real-time analytics
  - Azure Synapse Analytics for data warehousing. Azure Databricks for big data processing. Azure Machine Learning for model training and deployment.
  - **Avoid when:** data volume fits in a single SQL Server database — simpler to use Azure SQL Database with integrated analytics features.

- **Identity and access management** — centralized authentication and authorization across applications
  - Azure AD provides SSO, MFA, Conditional Access, and device management. Managed identities eliminate credential management for Azure resources.
  - **Avoid when:** all users are external to the organization (B2C scenarios) — Azure AD B2C is designed for consumer identity but adds configuration complexity.

- **Compliance and governance** — enforcing organizational policies and regulatory compliance at scale
  - Azure Policy enforces resource configuration rules. Blueprints deploy compliant environments. Microsoft Defender for Cloud provides compliance dashboards and remediation recommendations.
  - **Avoid when:** there is no regulatory requirement and the organization is small — Azure Policy adds operational overhead; manual review may be acceptable.

---

## Scenario-Based Questions

- **Q: Your healthcare application needs to store Protected Health Information (PHI) data on Azure and comply with HIPAA regulations. How do you design the architecture?**
  - **A:** The architecture must implement multiple layers of security and compliance. Azure Policy should be configured with the HIPAA regulatory compliance built-in initiative that continuously audits resource configurations against HIPAA controls. Azure SQL Database must use both Transparent Data Encryption and Always Encrypted for column-level encryption of sensitive fields. Key Vault with managed HSM stores encryption keys at FIPS 140-2 Level 3. Private Link must be configured for all PaaS services. Microsoft Defender for Cloud provides the regulatory compliance dashboard. Azure AD Conditional Access requires MFA for all access to PHI data. Log Analytics workspace with seven-year retention captures all audit logs.
  - **Interview follow-up:** How would you handle a scenario where a healthcare provider needs to share PHI with a partner organization that does not have Azure AD — what identity federation mechanism would you use to maintain HIPAA compliance across organizational boundaries?

- **Q: You have a 40 TB SQL Server database on-premise that must move to Azure with less than one hour of downtime. What migration approach do you use?**
  - **A:** Use Azure Database Migration Service (DMS) with continuous CDC replication. Use Azure Data Box for offline seed transfer of the 40 TB. SQL Managed Instance is the recommended target for near 100% compatibility. During the week before the maintenance window, CDC continuously synchronizes changes. During maintenance, stop writes, let CDC catch up, validate data integrity, and update the connection string. A dry run should be performed at least two weeks before production cutover.
  - **Interview follow-up:** What would you do if the dry run reveals that CDC replication cannot keep up with the write volume on the source, and the replication lag keeps growing beyond acceptable limits during the final sync window?

- **Q: Your AKS cluster experiences noisy neighbor problems where one team's workloads consume excessive resources and degrade performance for other teams. How do you enforce resource isolation?**
  - **A:** Use a multi-layered approach: dedicated namespaces with ResourceQuota objects defining hard limits on CPU, memory, storage, and pod counts. LimitRange objects set default requests and limits for pods without them. Azure Policy for AKS enforces that every pod specifies CPU and memory requests with the Deny effect. Network Policies restrict inter-namespace traffic. For critical workloads, dedicated node pools with taints and tolerations provide hardware-level isolation. Azure Cost Management with namespace labels provides per-team chargeback reporting.

- **Q: Your CI/CD pipeline deploys to 20+ Azure regions but currently takes four hours to complete. How do you optimize multi-region deployments?**
  - **A:** Deploy to multiple regions in parallel with a limit of five simultaneous deployments. Enable ACR geo-replication so each regional AKS cluster pulls from a local registry replica. Use Terraform or Bicep with state management for idempotent updates. Azure Front Door routes traffic only to regions that have completed health validation. Use a ring-based deployment strategy: Ring 0 (single canary region), Ring 1 (3 additional regions), Ring 2 (10 regions), Ring 3 (all remaining regions) with automated rollback at each ring.

```yaml
# Azure DevOps deployment pipeline with ring-based multi-region strategy
stages:
  - stage: Build
    jobs:
      - job: BuildAndPush
        steps:
          - script: docker build -t app:$(Build.BuildId) .
          - script: docker push acr.azurecr.io/app:$(Build.BuildId)

  - stage: Ring0_Canary
    displayName: 'Ring 0 - Single Region Canary'
    dependsOn: Build
    jobs:
      - deployment: DeployToEastUS
        environment: 'ring0-eastus'
        strategy:
          runOnce:
            deploy:
              steps:
                - script: helm upgrade app ./chart --set image.tag=$(Build.BuildId)
                - script: |
                    curl -f http://app-eastus.azurefd.net/health
```

- **Q: Your Azure bill is unexpectedly high because development and test virtual machines are running 24/7 including nights and weekends. How do you implement cost governance?**
  - **A:** Use Azure Policy to enforce auto-shutdown schedules based on Environment tag. Use Azure Dev/Test subscriptions for Dev/Test pricing benefits (no licensing costs for Windows Server and SQL Server). Review Azure Advisor recommendations weekly for right-sizing. Set Azure Budgets with action groups triggering automated workflows. Purchase Reserved Instances for baseline production capacity. Use Spot VMs for dev/test workloads.

- **Q: You need to authenticate users across multiple Azure tenants for a partner portal where external organizations access your applications. How do you design the identity solution?**
  - **A:** For partners using Azure AD, use Azure AD B2B collaboration with guest accounts for seamless authentication using existing corporate credentials. For partners without Azure AD or consumer scenarios, use Azure AD B2C with customizable user interfaces and social identity providers. For enterprise partners with strict security, configure direct federation. Apply Conditional Access policies specifically to external users with stricter requirements including MFA.

- **Q: Your multi-tier application must meet a service level agreement of 99.99 percent uptime on Azure. How do you design for this reliability target?**
  - **A:** Deploy compute tier across at least two Availability Zones with zone-redundant App Service Premium v3 or VM Scale Sets. Use Azure Front Door for global load balancing with 5-second health probes. Database tier: Azure SQL Business Critical with zone-redundant deployment (three synchronous replicas, failover under 30 seconds), Cosmos DB with multi-region writes. Use Azure Redis Cache Premium with zone redundancy. Implement circuit breaker and retry patterns with exponential backoff. Conduct monthly DR testing using Azure Chaos Studio.
  - **Interview follow-up:** During a Chaos Studio experiment simulating a full zone outage, your application survives but traffic concentrates on remaining zones and overwhelms the database connection pool — how would you design connection pooling and throttling to prevent this?

- **Q: Your development team accidentally deployed a VM with a wide-open NSG allowing RDP access from the entire internet. How do you prevent this from recurring?**
  - **A:** Use Azure Policy with Deny effect blocking creation of NSG rules allowing inbound RDP/SSH from 0.0.0.0/0. Use Microsoft Defender for Cloud's Just-In-Time VM access to eliminate always-open management ports. Deploy Azure Firewall as centralized egress control with forced tunneling. Enable NSG flow logs analyzed through Traffic Analytics. Enforce that all VMs use Azure Bastion for management access through Azure Policy.

- **Q: You are building a real-time dashboard that processes 100,000 events per second from IoT devices deployed across multiple geographic regions. How do you design the ingestion and processing pipeline?**
  - **A:** Use Azure IoT Hub for device ingestion with per-device authentication and built-in message routing to Event Hubs. Use Azure Stream Analytics with sliding window aggregations over 5-second tumbling windows for real-time temperature averages and anomaly detection. Hot path sends processed metrics to Power BI for sub-second dashboard updates. Cold path archives raw events to Azure Data Lake Storage Gen2 in Parquet format, with Azure Synapse Analytics for historical analysis and ML model training. Azure Functions triggered from Event Hubs handles device command processing.

```sql
-- Stream Analytics query for IoT device telemetry processing
WITH TemperatureReadings AS (
  SELECT
    DeviceId,
    Temperature,
    EventEnqueuedUtcTime,
    DeviceLocation
  FROM IoTHub TIMESTAMP BY EventEnqueuedUtcTime
  WHERE Temperature IS NOT NULL
)
SELECT
  DeviceId,
  AVG(Temperature) AS AvgTempLastMinute,
  MAX(Temperature) AS MaxTempLastMinute,
  MIN(Temperature) AS MinTempLastMinute,
  COUNT(*) AS ReadingCount,
  System.Timestamp AS WindowEnd
INTO HotPathDashboard
FROM TemperatureReadings
GROUP BY
  DeviceId,
  TumblingWindow(minute, 1)
```

- **Q: How would you migrate a monolithic .NET Framework application to Azure without performing a full rewrite?**
  - **A:** Use the Replatform/Lift and Shift approach. Azure App Service on Windows provides managed hosting with the Azure App Migration Assistant analyzing compatibility issues. Azure SQL Database replaces on-premise SQL Server via DMS. Use the Strangler Fig pattern for gradual modernization through Azure API Management. Azure Redis Cache replaces in-process session state for multi-instance scaling. Deployment slots provide zero-downtime deployments. Application Insights provides distributed tracing across both legacy monolith and new microservices.

---

## Interview Questions

- **What is the difference between Azure RBAC and Azure Policy?**
  - **A:** Azure RBAC controls who can perform actions on Azure resources by assigning roles at a specific scope — it answers "who can do what." Azure Policy evaluates resource configurations against defined rules and can Deny, Audit, or DeployIfNotExists — it answers "what resources are allowed." RBAC provides access control; Policy provides compliance enforcement. For example, RBAC grants Contributor access to a resource group, while Policy blocks creating VMs in non-approved regions.

- **What is an Azure Landing Zone and why is it critical for enterprise cloud adoption?**
  - **A:** An Azure Landing Zone is a scalable, governance-compliant Azure environment following Cloud Adoption Framework best practices. It includes management group hierarchy, subscription vending automation, Azure Policy assignments, RBAC roles, hub-spoke network topology, and centralized monitoring. It enforces consistent security and governance from day one, prevents configuration drift, provides self-service capabilities, and reduces time to onboard new workloads.

- **Explain the difference between Availability Sets and Availability Zones in Azure.**
  - **A:** Availability Sets provide redundancy within a single datacenter by distributing VMs across fault domains (physical racks) and update domains (planned maintenance groups), achieving 99.95% SLA. Availability Zones provide redundancy across physically separate datacenters within a region with independent power, cooling, and networking, achieving 99.99% SLA. Use Availability Zones for new stateless applications; use Availability Sets for legacy applications with tight latency requirements.

- **What is Azure Arc and how does it extend Azure's management capabilities?**
  - **A:** Azure Arc extends Azure Resource Manager-based management to any infrastructure outside Azure — on-premise datacenters, other public clouds, and edge locations. Arc-enabled servers provide consistent management including inventory, policy enforcement, and patch compliance. Arc-enabled Kubernetes provides GitOps configuration management and Azure Policy enforcement. Arc-enabled data services extend Azure SQL Managed Instance to any Kubernetes cluster. It creates a single control plane for managing distributed infrastructure.

- **What is the difference between Azure Managed Identity and a Service Principal?**
  - **A:** Managed Identity is automatically managed by Azure and tied to the lifecycle of an Azure resource — credentials are rotated every 45 days automatically with no secrets exposed to code. Service Principal is a manual Azure AD registration requiring explicit management of client secrets or certificates. Always prefer Managed Identities for Azure-resident workloads. Use Service Principals with certificate-based authentication for workloads outside Azure where Managed Identities aren't available.

- **How does Azure Front Door differ from Azure Traffic Manager?**
  - **A:** Azure Front Door operates at Layer 7 with HTTP/HTTPS global load balancing, SSL offloading, WAF, URL-based routing, session affinity, and caching. Azure Traffic Manager operates at Layer 4 using DNS-based load balancing with performance, geographic, weighted, and priority routing methods but does not inspect HTTP traffic or provide SSL termination. Use Front Door for web applications requiring advanced routing and WAF. Use Traffic Manager for DNS-level failover or non-HTTP protocols.

- **What are the differences between Azure Policy effects Deny, Audit, and DeployIfNotExists?**
  - **A:** Deny blocks creation of non-compliant resources before they are created — for critical controls like allowed regions or required encryption. Audit creates a warning event but does not block — used during initial governance rollout to measure compliance. DeployIfNotExists automatically remediates non-compliant resources by deploying a defined template — for automatically enabling diagnostic logging. Start with Audit on all policies, then move critical controls to Deny, and use DeployIfNotExists for automated remediation.

- **What is the difference between Azure SQL Database and Azure SQL Managed Instance?**
  - **A:** Azure SQL Database is a fully managed PaaS database with built-in HA, automated backups, serverless compute, and Hyperscale tier up to 100 TB — lowest administrative overhead but some compatibility limitations (no SQL Agent, cross-database queries, CLR). SQL Managed Instance provides near 100% on-premise SQL Server compatibility including SQL Agent, cross-database queries, CLR, Service Broker, and linked servers — runs in a dedicated VNet subnet. Choose Azure SQL Database for new cloud-native apps; choose SQL Managed Instance for lift-and-shift migrations.

- **How do you implement disaster recovery for Azure VMs, and what RPO/RTO can you achieve?**
  - **A:** Azure Site Recovery (ASR) provides continuous block-level replication to a secondary region with RPO as low as 30 seconds and typical RTO ranging from minutes for stateless servers to 30+ minutes for multi-tier applications. Recovery plans define VM startup order and automate script execution. Test failovers in isolated networks validate recovery without impacting production. For critical databases, combine ASR with Azure SQL Database active geo-replication for RPO of zero and RTO under 30 seconds.

- **What is Azure Bicep and how does it compare to ARM templates and Terraform?**
  - **A:** Azure Bicep is a domain-specific language for deploying Azure resources with a declarative syntax that is 50-70% more concise than ARM JSON. It provides modules, loops, conditional deployment, and automatic dependency detection. Bicep transpiles to ARM JSON with zero runtime overhead and no state file to manage. Compared to ARM JSON, Bicep is significantly more readable and modular. Compared to Terraform, Bicep is Azure-only but requires no state management. Choose Bicep for Azure-only environments; choose Terraform for multi-cloud deployments.

---

## Developer Recommendations

- **Use Managed Identities instead of service principals for all Azure-resident workloads** — Managed Identities completely eliminate credential rotation, secret storage, and expiry management because Azure AD automatically creates and rotates identity credentials. System-assigned identities are tied to a single resource lifecycle. User-assigned identities persist independently and can be shared across multiple resources. The Azure Identity SDK automatically handles token acquisition using Managed Identity when running in Azure. Refactor any code retrieving client secrets to use the Azure Identity SDK. This eliminates the most common credential leak scenarios and simplifies compliance audits.
  - **Production story:** One team stored a service principal secret in an App Service app setting — when a junior developer pushed a debug endpoint to production that returned all environment variables, the leaked credential gave an attacker read access to every storage container in the subscription for 72 hours before the compromise was detected.

- **Always use Azure Private Link for PaaS services to eliminate public internet exposure** — Exposing Azure SQL, Blob Storage, Key Vault, Cosmos DB, or ACR through public endpoints creates unnecessary security risk. Private Link maps the PaaS service to a private IP address within your VNet, ensuring all traffic traverses the Microsoft backbone network. Combined with Azure Firewall and NSGs, this ensures no data traffic can be intercepted. Private Endpoints cost approximately $0.01 per hour each plus data processing charges. Azure Policy should enforce Private Link usage for all supported PaaS services.

- **Implement Azure Policy governance early and progressively enforce with Deny** — Azure Policy provides preventative controls that stop insecure resources from being deployed. Start with the Audit effect on critical policies (allowed regions, required tags, encryption, NSG rules) to measure compliance. After 2-4 weeks, introduce the Deny effect for highest-priority security and cost controls. Use Policy Initiatives grouping related policies (Security Baseline, Cost Management, Compliance). Policy exemptions require expiration dates and justification fields. Regular compliance reviews identify new governance requirements.
  - **Production story:** A financial services firm skipped Azure Policy entirely during their first year on Azure, and during a SOC 2 audit discovered that 40% of their storage accounts had public network access enabled with no authenticated access logging — it took six months and $200,000 in remediation costs.

- **Design for cost optimization from day one and continuously monitor spending against budgets** — Use the Azure Pricing Calculator to estimate costs before deploying. Most production VMs are over-provisioned by approximately 40%. Set Azure Budgets at management group, subscription, and resource group levels with alerts at 50%, 90%, and 100%. Enable auto-shutdown for all non-production environments. Use Spot VMs for interruptible workloads for 60-90% savings. Apply Azure Hybrid Benefit for 40-50% licensing savings. Purchase Reserved Instances for baseline capacity. Enforce mandatory resource tagging for cost allocation accuracy.

- **Use deployment slots for all Azure App Service applications to enable zero-downtime deployments** — The staging slot allows comprehensive validation before any production traffic is directed. The swap operation completes in seconds regardless of application size because it only updates routing rules. Slot-sticky settings ensure staging connects to staging databases and production to production databases. Auto-swap enables continuous delivery with warm-up. Deployment slots also support A/B testing by routing a percentage of traffic to staging, and instant rollback by swapping back.

- **Prefer Bicep or Terraform over ARM templates for all infrastructure as code on Azure** — ARM JSON templates are excessively verbose for anything beyond simple deployments. Bicep provides syntax approximately 50% shorter than ARM JSON with first-class modules, loops, conditional deployment, and automatic dependency detection. Bicep transpiles to ARM JSON with zero runtime overhead and full compatibility with all ARM features. Terraform provides cloud-agnostic IaC with richer state management features. For Azure-only environments, Bicep is recommended. For multi-cloud environments, Terraform provides consistent tooling.

- **Monitor with Application Insights, not just infrastructure metrics, to understand application health and performance** — Infrastructure metrics (CPU, memory, disk IOPS) tell you something is wrong but not what or why. Application Insights provides request rates, failure rates, dependency call times, exceptions, and distributed tracing across microservices. Smart detection automatically identifies anomalies using machine learning. Availability tests ping your application from multiple global locations. Configure sampling to manage data ingestion costs — retain standard metrics for 90 days, sample detailed transaction traces at 10-50%.
