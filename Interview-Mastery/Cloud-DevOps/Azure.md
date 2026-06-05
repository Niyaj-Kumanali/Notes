# Azure

---

## Overview

- **Definition:** Microsoft Azure is the second-largest cloud platform offering 200+ services across compute, storage, databases, AI, and DevOps with deep enterprise integration.
- **Why It Exists:** Azure provides the largest regional footprint (60+ regions), deep Microsoft ecosystem integration (Active Directory, SQL Server, .NET), and hybrid cloud capabilities via Azure Arc, making it the preferred cloud for many enterprises.
- **Key Concepts:** **Regions** (60+ globally), **Region Pairs** (built-in DR pairing within geography), **Availability Zones** (1-3 per region), **Resource Hierarchy** (Management Group > Subscription > Resource Group > Resource), **Azure Resource Manager (ARM)** for deployment and management

---

## Core Services

- **Virtual Machines (Compute):** IaaS VMs for any workload. Use VM Scale Sets (VMSS) for auto-scaling. Types: General Purpose (D-series), Compute (F-series), Memory (E-series).
- **App Service (PaaS):** Managed web app hosting with auto-scaling, deployment slots, and CI/CD. Supports .NET, Java, Node.js, Python, PHP.
- **AKS (Containers):** Managed Kubernetes with Azure AD integration, managed identity, and Azure Container Registry (ACR).
- **Azure Functions (Serverless):** Event-driven compute with HTTP, Queue, Timer, Cosmos DB triggers. Supports multiple languages. Plans: Consumption, Premium, Dedicated.
- **Azure SQL Database (Databases):** Managed SQL Server with built-in HA, automatic backups, serverless compute. Cosmos DB for globally distributed NoSQL.
- **Blob Storage (Storage):** Object storage with LRS, GRS, RA-GRS, ZRS redundancy. Hot/Cool/Archive tiers. Data Lake Storage Gen2 for analytics.
- **Virtual Network (Networking):** Isolated network with subnets, NSGs, peering, VPN Gateway, ExpressRoute, Azure Firewall, Azure Front Door.
- **Azure AD (Identity):** Identity and access with Conditional Access, Managed Identities, RBAC, Privileged Identity Management.

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

---

## Common Mistakes

- **Default VNet** — Security risk; use custom VNet with proper segmentation
- **Open NSG rules (0.0.0.0/0)** — Vulnerability; use least privilege
- **No cost alerts** — Bill shock; set up Azure Budgets
- **Single-region deployment** — No DR; use multi-region with Traffic Manager
- **No RBAC** — Over-permissioned; use Azure AD RBAC with least privilege
- **No resource tagging** — No cost tracking; enforce tagging via Azure Policy
- **Public PaaS endpoints** — Security risk; use Private Link for PaaS services
- **No auto-scaling** — Performance issues; configure autoscale rules
- **No managed identities** — Credential burden; use Managed Identity
- **No Azure Policy** — Governance gaps; enforce compliance via policies

---

## Key Design Considerations

- **Landing Zone** — Enterprise-scale architecture with management groups (Platform + Workloads), subscription vending via Terraform/ARM, pre-configured policies, RBAC, and budget allocation
- **High Availability** — Availability Sets (99.95%) for VMs, Availability Zones (99.99%), Load Balancer/Traffic Manager, Azure Site Recovery for DR, geo-redundant storage
- **Hybrid Cloud** — Azure Arc for managing on-premise and multi-cloud resources. ExpressRoute for dedicated private connectivity. VPN Gateway for encrypted tunnels. Azure Stack Edge for edge computing
- **Security** — Azure AD Conditional Access, Managed Identities (no credentials in code), Defender for Cloud, Azure Policy, NSGs on all subnets, Private Link for PaaS, Azure Sentinel for SIEM
- **Cost Optimization** — Reserved Instances (30-72%), Spot VMs (60-90%), Azure Hybrid Benefit (40-50%), right-sizing via Azure Advisor, auto-shutdown dev/test environments
- **Well-Architected Framework** — Cost Optimization, Operational Excellence, Performance Efficiency, Reliability, Security pillars

---

## Real-World Scenarios

**Scenario 1: Enterprise Migration from On-Premise to Azure**
A financial services company migrating 200+ workloads from on-premise datacenters to Azure. Use Azure Migrate for discovery and assessment. Azure Site Recovery for replication. ExpressRoute for private connectivity during migration. Group workloads into waves, validate each wave, cutover during maintenance windows. Use Azure Cost Management to track migration ROI. Implement Azure Landing Zone with management groups for governance.

```powershell
# Deploy Azure Site Recovery for VM replication
$asrVault = New-AzRecoveryServicesVault -Name "ASR-Vault" -ResourceGroupName "Migration-RG" -Location "EastUS"
Set-ASRVaultContext -Vault $asrVault

# Enable replication for each VM
$vm = Get-AzVM -ResourceGroupName "OnPrem-RG" -Name "WebServer01"
New-AzRecoveryServicesAsrProtectionContainerMapping `
  -Name "PrimaryToDR" `
  -Policy $asrPolicy `
  -PrimaryProtectionContainer $primaryContainer `
  -RecoveryProtectionContainer $recoveryContainer
```

**Scenario 2: Real-Time Fraud Detection with Azure Stream Analytics**
A payment processing company needs to detect fraudulent transactions in real-time (<100ms latency) from 50K transactions/second. Use Event Hubs for ingestion (throughput units auto-inflate). Azure Stream Analytics with reference data join for known fraud patterns. Output to Azure SQL for alerts. Use Azure Functions for webhook notifications. Cosmos DB with Cassandra API for historical pattern lookups. Power BI for real-time dashboard.

**Scenario 3: Multi-Tenant SaaS Platform on AKS**
A SaaS company building a multi-tenant application needs tenant isolation, cost tracking per tenant, and zero-downtime deployments. Use AKS with namespace-per-tenant and network policies for isolation. Azure AD B2C for tenant authentication. Azure Policy to enforce resource quotas per namespace. Azure Cost Management with tags per tenant. Helm charts for tenant-specific deployments. Flux v2 for GitOps.

---

## Scenario-Based Questions

1. **Q: Your healthcare application needs to store PHI data on Azure and comply with HIPAA. How do you design the architecture?**
   A: Use Azure Policy to enforce HIPAA controls (audit if encryption disabled, enforce TLS). Azure SQL with Transparent Data Encryption and Always Encrypted for column-level encryption. Key Vault with managed HSM for FIPS 140-2 Level 3 keys. Private Link for all PaaS services. Defender for Cloud with regulatory compliance dashboard. Azure AD Conditional Access requiring MFA for all PHI access. Audit logs in Log Analytics with 7-year retention.

2. **Q: You have a 40 TB SQL Server database on-premise that must move to Azure with < 1 hour downtime. What migration approach do you use?**
   A: Use Azure DMS with continuous CDC replication. Set up Azure SQL Managed Instance as target (closest compatibility). Run full load during week, then during maintenance window stop writes, let CDC catch up, redirect connection string. For 40 TB, consider Azure Data Box for initial seed transfer to reduce full load time. Validate data integrity post-migration with row counts and checksums.

3. **Q: Your AKS cluster experiences noisy neighbor problems where one team's workloads impact others. How do you enforce resource isolation?**
   A: Implement namespace-per-team with resource quotas (CPU/memory limits per namespace). Use Azure Policy to enforce Pod resource requests and limits. Network policies to restrict inter-namespace traffic. Deploy team-specific node pools with taints and tolerations (critical workloads on dedicated nodes). Use Azure Cost Analysis with namespace labels for chargeback. Enable Container Insights for per-namespace monitoring.

4. **Q: Your CI/CD pipeline deploys to 20+ Azure regions but takes 4 hours. How do you optimize multi-region deployments?**
   A: Use Azure DevOps deployment groups with parallel deployment jobs (max 5 regions at a time). Implement ACR geo-replication so each region pulls from a local registry. Use Azure Front Door global load balancer — deploy to one region, validate, then proceed. Terraform workspaces for region-specific config. Use a ring-based deployment strategy: Ring 0 (canary 1 region), Ring 1 (3 regions), Ring 2 (10 regions), Ring 3 (all regions), with automated validation at each ring.

5. **Q: Your Azure bill is unexpectedly high due to dev/test VMs running 24/7. How do you implement cost governance?**
   A: Enforce Azure Policy to auto-shutdown VMs based on tags (Environment=Dev shuts down at 7 PM). Use Dev/Test pricing (Azure Dev/Test subscription — no licensing costs). Right-size VMs via Azure Advisor recommendations. Implement Azure budgets with API-based alerts that trigger webhooks to auto-stop resources. Use reserved instances for baseline VMs. Enforce tagging via Azure Policy to identify orphaned resources.

6. **Q: You need to authenticate users across multiple Azure tenants (B2B) for a partner portal. How do you design identity?**
   A: Use Azure AD B2B collaboration — invite external users as guest accounts in your tenant. For partner-managed identities, use Azure AD B2C with custom identity providers (Google, Microsoft, Facebook). For enterprise partners, configure cross-tenant federation. Enable Conditional Access policies for external users (require MFA, device compliance). Use Azure AD Identity Protection for risky sign-in detection across tenants.

7. **Q: Your multi-tier app must meet an SLA of 99.99% on Azure. How do you design for this reliability target?**
   A: Deploy across at least 2 Availability Zones (99.99% SLA). Use Azure Front Door for global load balancing and failover. App Service in Premium plan with zone redundancy. Azure SQL Business Critical tier with zone-redundant deployment. Cosmos DB multi-region writes for database. Traffic Manager for DNS-level failover. Design for degradation: cache (Azure Redis) to handle backend failures. Test failover monthly.

8. **Q: Your development team accidentally deployed a VM with a wide-open NSG. How do you prevent insecure configurations?**
   A: Use Azure Policy with Deny effect for specific insecure rules (e.g., deny NSG rules allowing RDP/SSH from 0.0.0.0/0). Use Azure Blueprints to enforce baseline NSG configurations per environment. Implement Just-In-Time VM access via Microsoft Defender for Cloud (eliminates always-open management ports). Add Azure Firewall for centralized egress control. Use NSG flow logs analyzed in Traffic Analytics.

9. **Q: You are building a real-time dashboard processing 100K events/second from IoT devices. How do you design the ingestion pipeline?**
   A: IoT Hub for device ingestion with device-to-cloud partitions. Event Hubs with auto-inflate throughput units for spike handling. Azure Stream Analytics with sliding window aggregations for real-time processing. Output to Azure SQL Hyperscale for dashboards. Hot path: streaming output to Power BI for sub-second latency. Cold path: Archive raw events to Data Lake Storage Gen2 for historical analysis via Azure Synapse.

10. **Q: How would you migrate a monolithic .NET Framework app to Azure without a full rewrite?**
    A: Use the Replatform (Lift and Shift) approach: migrate to Azure App Service (Windows) with Azure SQL Database. Use Azure App Migration Assistant to assess compatibility. Break the monolith using the Strangler Fig pattern — gradually route endpoints to new microservices on AKS. Use Azure API Management as the facade. Redis Cache for session state (was in-process). Use deployment slots for zero-downtime during incremental migration.

---

## Interview Questions

1. **What is the difference between Azure RBAC and Azure Policy?**
   A: RBAC controls who can do what (permissions on resources — built-in roles: Owner, Contributor, Reader). Azure Policy enforces rules on resource configurations (e.g., only allow specific VM sizes). RBAC is for access control, Policy is for compliance.

2. **What is Azure Landing Zone?**
   A: A scalable, governance-compliant Azure environment following best practices. Includes management group hierarchy, subscription architecture, Azure Policy assignments, RBAC, networking hub-spoke topology, and monitoring. Deployed via Azure Blueprints or Terraform.

3. **Explain Availability Sets vs Availability Zones.**
   A: Availability Sets distribute VMs across fault domains (different racks) and update domains within a single datacenter — 99.95% SLA. Availability Zones distribute VMs across physically separate datacenters within a region — 99.99% SLA. Use Zones for higher SLA, Sets for legacy apps that can't use Zones.

4. **What is Azure Arc?**
   A: Azure Arc extends Azure management to any infrastructure — on-premise, multi-cloud, edge. Provides Azure Resource Manager-based management, Azure Policy, Defender for Cloud, and GitOps for Kubernetes clusters running outside Azure.

5. **What is the difference between Managed Identity and Service Principal?**
   A: Managed Identity is an Azure resource identity automatically managed by Azure AD — no credential rotation needed. Service Principal is a manual application registration in Azure AD requiring client secret or certificate management. Always prefer Managed Identity.

6. **How does Azure Front Door differ from Traffic Manager?**
   A: Front Door is Layer 7 (HTTP/HTTPS) with global load balancing, SSL offload, WAF integration, URL-based routing, and caching. Traffic Manager is Layer 4 (DNS-based) with no SSL or WAF. Use Front Door for web apps, Traffic Manager for global DNS failover.

7. **What is Azure Policy's effect 'Deny' vs 'Audit' vs 'DeployIfNotExists'?**
   A: Deny blocks non-compliant resource creation. Audit logs non-compliant resources (no block). DeployIfNotExists auto-remediates non-compliant resources. Use Audit first to understand compliance, then Deny, then automated remediation.

8. **What is the difference between Azure SQL Database and SQL Managed Instance?**
   A: Azure SQL Database is a PaaS single database with built-in HA, automated backups, serverless compute. SQL Managed Instance provides full SQL Server instance compatibility (SQL Agent, cross-database queries, CLR) for lift-and-shift migrations. Choose SQL DB for new apps, Managed Instance for existing app migrations.

9. **How do you implement disaster recovery for Azure VMs?**
   A: Use Azure Site Recovery — replicates VMs to secondary region continuously. RPO as low as 30 seconds. Test failover in isolated network. Planned failover for zero data loss. Failback to primary after recovery. Alternative: native replication with Azure Backup + geo-restore.

10. **What is Azure Bicep and how does it compare to ARM templates?**
    A: Bicep is a declarative DSL for Azure deployments — simpler syntax than ARM JSON, modular, supports loops and conditionals, transpiles to ARM JSON. Benefits: cleaner syntax, type safety, no state file (unlike Terraform). Limits: Azure-only (unlike Terraform).

---

## Developer Recommendations

- **Use Managed Identities instead of service principals** — Managed Identities eliminate credential rotation, hardcoded secrets, and expiry management. An Azure resource (VM, App Service, Function) gets an automatic Azure AD identity. No secrets to store, rotate, or leak. Use system-assigned for single resource, user-assigned for consistent identity across resources.

- **Always use Private Link for PaaS services** — Exposing Azure SQL, Storage, or Key Vault to the internet is unnecessary risk. Private Link maps the PaaS service to a private IP in your VNet. Combined with Azure Firewall, no public traffic ever reaches your data. Trade-off: extra cost for Private Endpoints ($0.01/hr) and connection limits (10K per endpoint).

- **Implement Azure Policy early, enforce with Deny** — Policy is the single most important governance tool. Start with Audit to measure compliance, then move to Deny for critical controls (allowed locations, prohibited SKUs, required tags). Use Policy initiatives for grouping related policies. Exemptions support legitimate exceptions with expiry dates.

- **Design for cost from day one** — Use Azure Pricing Calculator before deploying. Right-size: most production VMs are over-provisioned 40%. Use Advisor recommendations. Set budgets with alerts at 50%, 90%, 100%. Enable auto-shutdown for non-production. Use Spot VMs for batch jobs (60-90% savings). Hybrid Benefit saves 40-50% on Windows Server/SQL Server licensing.

- **Use deployment slots for all App Service apps** — Zero-downtime deployments are a swap operation away. Staging slot runs your new version, validate with smoke tests, then swap. Auto-swap for CI/CD. Slot-sticky settings (connection strings, env variables) don't change during swap. Warm-up requests ensure the new slot is ready before traffic arrives.

- **Prefer Bicep or Terraform over ARM templates** — ARM JSON is verbose and error-prone. Bicep is simpler and Azure-native. Terraform is cloud-agnostic with richer state management. For Azure-only stacks, use Bicep. For multi-cloud, use Terraform. Both enable code review, version control, and repeatable deployments. Avoid clicking in the portal for anything production.

- **Monitor with Application Insights, not just infrastructure metrics** — Infrastructure metrics (CPU, memory) tell you something is wrong. Application Insights tells you what and why. Track request rates, failure rates, dependency call times, and custom events. Use distributed tracing for microservices. Set smart detection alerts for anomalies. Trade-off: extra cost for data ingestion, but invaluable for troubleshooting.