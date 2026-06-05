# Azure Study Guide

## 1. Executive Summary

Microsoft Azure is the second-largest cloud computing platform, offering 200+ services across compute, storage, databases, AI, and DevOps. With the largest regional footprint (60+ regions), deep enterprise integration (Active Directory, SQL Server, .NET), and hybrid cloud capabilities (Azure Arc), Azure is the preferred cloud for many enterprises. Key services include Virtual Machines, App Service, AKS (Kubernetes), Azure Functions (serverless), Azure SQL Database, and Azure Active Directory.

## 2. Core Theory

### 2.1 Azure Global Infrastructure

```
Azure Global Infrastructure
  +-- Geography (Americas, Europe, Asia Pacific, ME/Africa)
       +-- Region (paired within geography)
       |    +-- Availability Zone (1-3 per region)
       |    +-- Datacenter
       |
       +-- Resource Hierarchy:
            Management Group -> Subscription -> Resource Group -> Resource
```

### 2.2 Region Pairs

| Primary | Paired | Use Case |
|---------|--------|----------|
| North Europe (Ireland) | West Europe (Netherlands) | DR |
| East US (Virginia) | West US (California) | DR |
| UK South | UK West | In-country DR |
| Southeast Asia | East Asia | Regional DR |

### 2.3 Service Categories

| Category | Services |
|----------|----------|
| Compute | VMs, VMSS, App Service, AKS, Container Instances, Functions, Batch |
| Storage | Blob, Disk, Files, Archive Storage, Data Lake |
| Database | SQL DB, Cosmos DB, MySQL, PostgreSQL, SQL MI, Synapse |
| Networking | VNet, Load Balancer, App Gateway, CDN, DNS, VPN Gateway |
| Security | Azure AD, Key Vault, Security Center, Sentinel, Defender |
| DevOps | Boards, Repos, Pipelines, Artifacts, Test Plans |
| AI/ML | Cognitive Services, ML Service, Bot Service, OpenAI |

## 3. Under-the-Hood Deep Dive

### 3.1 Azure Resource Manager

```
Request -> Azure Resource Manager (ARM)
              +-- Authorization (Azure AD / RBAC)
              +-- Resource Provider Registration
              +-- Deployment (portal/CLI/PowerShell/API/SDK)
              +-- Resource Group placement
              +-- Provisioning
```

### 3.2 Virtual Network Architecture

```yaml
VNet:
  Address Space: 10.0.0.0/16
  Subnets:
    - Frontend: 10.0.1.0/24 [NSG: Web-NSG]
    - Backend:  10.0.2.0/24 [NSG: App-NSG]
    - Data:     10.0.3.0/24 [NSG: DB-NSG, ServiceEndpoints: Storage, SQL]
  Peering:
    - hub-vnet (hub-and-spoke)
    - onprem-vnet (VPN/ExpressRoute)
```

### 3.3 Storage Account Types

| Type | Performance | Redundancy | Use Case |
|------|-------------|------------|----------|
| Standard LRS | HDD | Local | Dev/test |
| Standard GRS | HDD | Regional | Geo-redundant |
| Standard RA-GRS | HDD | Regional + read | DR read access |
| Premium LRS | SSD | Local | Production VMs |
| Premium ZRS | SSD | Zone | HA production |

## 4. Production Code Examples

### 4.1 ARM Template for Web App + SQL

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environment": {
      "type": "string",
      "defaultValue": "production",
      "allowedValues": ["production", "staging", "development"]
    },
    "sqlAdminPassword": {
      "type": "securestring"
    }
  },
  "variables": {
    "appName": "[format('app-{0}', parameters('environment'))]",
    "sqlServerName": "[format('sql-{0}-{1}', parameters('environment'), uniqueString(resourceGroup().id))]",
    "storageAccountName": "[format('st{0}{1}', parameters('environment'), uniqueString(resourceGroup().id))]",
    "hostingPlanName": "[format('plan-{0}', parameters('environment'))]"
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2023-01-01",
      "name": "[variables('storageAccountName')]",
      "location": "[resourceGroup().location]",
      "sku": {
        "name": "[if(equals(parameters('environment'), 'production'), 'Standard_GRS', 'Standard_LRS')]"
      },
      "kind": "StorageV2",
      "properties": {
        "minimumTlsVersion": "TLS1_2",
        "supportsHttpsTrafficOnly": true
      }
    },
    {
      "type": "Microsoft.Sql/servers",
      "apiVersion": "2023-02-01-preview",
      "name": "[variables('sqlServerName')]",
      "location": "[resourceGroup().location]",
      "properties": {
        "administratorLogin": "sqladmin",
        "administratorLoginPassword": "[parameters('sqlAdminPassword')]",
        "minimalTlsVersion": "1.2"
      }
    },
    {
      "type": "Microsoft.Sql/servers/databases",
      "apiVersion": "2023-02-01-preview",
      "name": "[format('{0}/{1}', variables('sqlServerName'), 'myapp')]",
      "location": "[resourceGroup().location]",
      "dependsOn": ["[resourceId('Microsoft.Sql/servers', variables('sqlServerName'))]"],
      "sku": {
        "name": "[if(equals(parameters('environment'), 'production'), 'S2', 'S0')]"
      }
    },
    {
      "type": "Microsoft.Web/serverfarms",
      "apiVersion": "2023-01-01",
      "name": "[variables('hostingPlanName')]",
      "location": "[resourceGroup().location]",
      "sku": {
        "name": "[if(equals(parameters('environment'), 'production'), 'P1v3', 'F1')]"
      }
    },
    {
      "type": "Microsoft.Web/sites",
      "apiVersion": "2023-01-01",
      "name": "[variables('appName')]",
      "location": "[resourceGroup().location]",
      "dependsOn": ["[resourceId('Microsoft.Web/serverfarms', variables('hostingPlanName'))]"],
      "properties": {
        "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', variables('hostingPlanName'))]",
        "httpsOnly": true,
        "siteConfig": {
          "minTlsVersion": "1.2",
          "ftpsState": "FtpsOnly"
        }
      }
    }
  ],
  "outputs": {
    "appUrl": {
      "type": "string",
      "value": "[format('https://{0}.azurewebsites.net', variables('appName'))]"
    }
  }
}
```

### 4.2 Terraform for Azure

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
  }
}

provider "azurerm" {
  features {
    key_vault {
      purge_soft_delete_on_destroy = true
    }
  }
}

resource "azurerm_resource_group" "main" {
  name     = "rg-${var.environment}-${var.project}"
  location = var.location
  tags = {
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "Terraform"
  }
}

resource "azurerm_virtual_network" "main" {
  name                = "vnet-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_subnet" "frontend" {
  name                 = "snet-frontend"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_subnet" "backend" {
  name                 = "snet-backend"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.2.0/24"]
  service_endpoints    = ["Microsoft.Storage", "Microsoft.Sql"]
}

resource "azurerm_kubernetes_cluster" "main" {
  name                = "aks-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "aks-${var.environment}"

  default_node_pool {
    name       = "default"
    node_count = var.environment == "production" ? 5 : 2
    vm_size    = "Standard_D2s_v3"
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin = "azure"
    network_policy = "calico"
  }
}

resource "azurerm_container_registry" "main" {
  name                = "acr${var.project}${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Premium"
  admin_enabled       = false

  georeplications {
    location = var.secondary_location
  }
}

resource "azurerm_key_vault" "main" {
  name                = "kv-${var.project}-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  tenant_id           = data.azurerm_client_config.current.tenant_id
  sku_name            = "standard"

  soft_delete_retention_days = 90
  purge_protection_enabled   = var.environment == "production"
}

resource "azurerm_log_analytics_workspace" "main" {
  name                = "log-${var.project}-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "PerGB2018"
  retention_in_days   = var.environment == "production" ? 365 : 30
}
```

### 4.3 Deployment Slots (App Service)

```powershell
# Create staging slot
az webapp deployment slot create \
  --resource-group rg-myapp \
  --name app-myapp \
  --slot staging \
  --configuration-source app-myapp

# Deploy to staging
az webapp deployment source config-zip \
  --resource-group rg-myapp \
  --name app-myapp \
  --slot staging \
  --src deploy.zip

# Swap slots
az webapp deployment slot swap \
  --resource-group rg-myapp \
  --name app-myapp \
  --slot staging \
  --target-slot production

# Auto-swap configuration (Azure DevOps)
# - task: AzureWebApp@1
#   inputs:
#     azureSubscription: 'sc-prod'
#     appName: 'app-myapp'
#     package: '$(Pipeline.Workspace)/drop/**/*.zip'
#     deployToSlotOrASE: true
#     slotName: 'staging'
```

### 4.4 Managed Identity (C#)

```csharp
[FunctionName("ProcessOrder")]
public async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req,
    ILogger log)
{
    string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
    var order = JsonSerializer.Deserialize<Order>(requestBody);

    // Use managed identity to access Key Vault
    var client = new SecretClient(
        new Uri(Environment.GetEnvironmentVariable("KEY_VAULT_URL")),
        new DefaultAzureCredential());

    KeyVaultSecret connectionString = await client.GetSecretAsync("SqlConnectionString");

    // Use managed identity to access Storage
    var blobClient = new BlobContainerClient(
        new Uri(Environment.GetEnvironmentVariable("STORAGE_URL")),
        new DefaultAzureCredential());

    await blobClient.CreateIfNotExistsAsync();
    return new OkObjectResult(new { order.Id, Status = "Processed" });
}
```

## 5. Real-World Scenarios

### 5.1 Hub-and-Spoke Network

```yaml
hub:
  - Azure Firewall
  - Azure Bastion
  - Log Analytics Workspace
  - Key Vault
  - ExpressRoute/VPN Gateway

spokes:
  production:
    - VNet: 10.1.0.0/16
    - AKS cluster, App Service, Azure SQL
    - Peered to hub
  development:
    - VNet: 10.2.0.0/16
    - VM Scale Set, dev databases
    - Peered to hub
```

### 5.2 Hybrid Cloud with Arc

```bash
# Connect on-premise to Azure Arc
az cmc connect --resource-group rg-hybrid \
  --location eastus --resource-name webserver-01

# Apply Azure Policy to on-premise resources
az policy assignment create \
  --policy "Ensure Linux SSH access is restricted" \
  --resource-group rg-hybrid
```

## 6. Performance

### 6.1 Cost Optimization

| Strategy | Savings | Effort |
|----------|---------|--------|
| Reserved Instances | 30-72% | Medium |
| Spot VMs | 60-90% | Low |
| Right-sizing (Advisor) | 20-50% | Medium |
| Auto-shutdown dev/test | 30-60% | Low |
| Storage lifecycle | 40-80% | Low |
| Azure Hybrid Benefit | 40-50% | Low |

### 6.2 Monitoring

```bash
# Application Insights query
az monitor app-insights query \
  --app app-insights-prod \
  --analytics-query "requests | where timestamp > ago(1h) | summarize count(), avg(duration) by resultCode"

# VM metrics
az monitor metrics list \
  --resource /subscriptions/.../Microsoft.Compute/virtualMachines/vm-prod \
  --metric "Percentage CPU" --interval PT1H
```

## 7. Security

### 7.1 Security Best Practices

- Enable Azure AD Conditional Access
- Use Managed Identities (no credentials in code)
- Enable Defender for Cloud
- Use Azure Policy for compliance
- NSGs on all subnets
- Private Link for PaaS services
- Azure Sentinel for SIEM

### 7.2 Key Vault

```bash
# Store and retrieve secrets
az keyvault secret set --vault-name kv-myapp --name "DbPassword" --value "P@ssw0rd!"

# Grant managed identity access
az keyvault set-policy --name kv-myapp \
  --object-id $(az vm show -n vm-app -g rg-myapp --query identity.principalId -o tsv) \
  --secret-permissions get list

# Enable soft-delete and purge protection
az keyvault update --name kv-myapp \
  --enable-soft-delete true \
  --enable-purge-protection true
```

## 8. Common Mistakes

| Mistake | Impact | Solution |
|---------|--------|----------|
| Default VNet | Security risk | Custom VNet |
| Open NSG rules | Vulnerability | Least privilege |
| No cost alerts | Bill shock | Azure Budgets |
| Single-region | No DR | Multi-region |
| No RBAC | Over-permissioned | Azure AD RBAC |
| No tags | No cost tracking | Enforce tagging policy |
| Public PaaS endpoints | Security risk | Private Link |
| No auto-scaling | Performance issues | Configure autoscale |
| No managed identities | Credential mgmt | Managed Identity |

## 9. Senior Engineer Perspective

### 9.1 Azure Landing Zone

```yaml
landing_zone:
  management_groups:
    - Root
      - Platform (Management, Connectivity, Identity)
      - Workloads (Production, Non-production, Sandbox)
  subscription_vending:
    - Automated via Terraform/ARM
    - Pre-configured policies
    - Budget allocation
    - RBAC assignment
```

### 9.2 Well-Architected Framework

1. Cost Optimization: Right-size, reserved instances, budgets
2. Operational Excellence: Automation, monitoring, IaC
3. Performance Efficiency: Scaling, CDN, caching
4. Reliability: Multi-region, HA, backup, DR
5. Security: Zero trust, encryption, least privilege

## 10. Interview Questions (Easy)

1. What is Microsoft Azure?
2. What is a Resource Group?
3. What is the difference between a VM and App Service?
4. What is Azure Blob Storage?
5. What is Azure Active Directory?
6. What is a Virtual Network?
7. What is Azure Functions?
8. What is the Azure Portal?
9. What is a storage account?
10. What is an availability zone?

## 10. Interview Questions (Medium)

11. Azure SQL DB vs SQL Managed Instance?
12. How does Azure RBAC work?
13. What is Azure DevOps?
14. How do you implement HA in Azure?
15. What is Azure Policy?
16. NSG vs ASG?
17. How does Azure Site Recovery work?
18. What is App Service plan?
19. LRS vs GRS?
20. How do you secure Azure resources?

## 11. Advanced Interview Questions (Hard)

1. Design multi-region HA with automatic failover on Azure.
2. Migrate on-premise .NET app to Azure.
3. Design hybrid cloud architecture with Azure Arc.
4. Governance across 100+ Azure subscriptions.
5. Zero-trust security model for Azure.
6. DR solution with Azure Site Recovery.
7. Cost-optimized AKS with spot node pools.
8. Secrets management across Azure environments.
9. Serverless event-driven architecture on Azure.
10. Blue-green deployments on App Service.

## 11. Advanced Interview Questions (System Design)

11. Global e-commerce platform on Azure.
12. Data analytics pipeline using Azure Synapse.
13. Real-time IoT platform on Azure.
14. SaaS multi-tenant solution on Azure.
15. CI/CD platform for 500+ services.
16. HPC solution on Azure.
17. Media streaming platform on Azure.
18. Financial services with regulatory compliance.
19. AI/ML platform using Azure Machine Learning.
20. Backup/DR for 1000+ workloads.

## 12. Expert-Level Interview Questions (Architect)

1. Design Azure landing zone for 10,000+ subscriptions with governance, security, cost management.

2. Architect migration of 5,000+ VMs from VMware to Azure using Azure Migrate with minimal downtime.

3. Design global active-active across 5 regions with Cosmos DB multi-master, Traffic Manager, Front Door.

4. Zero-trust network eliminating all public IPs, using Private Link exclusively.

5. Data platform with Azure Synapse, Data Lake, Power BI serving 50,000+ users sub-second.

6. Azure SOC using Sentinel, Defender, custom analytics for 500+ apps.

7. Multi-cloud strategy with Azure Arc managing AWS, GCP, and on-premise.

8. High-frequency trading platform with sub-millisecond latency.

9. Governance automation with Azure Policy, Blueprints for regulated industries.

10. Cost management platform auto-optimizing 10,000+ workloads across subscriptions.

## 13. Debugging & Troubleshooting

### 13.1 Common Issues

```bash
# VM boot diagnostics
az vm boot-diagnostics get-boot-log --name vm-prod --resource-group rg-prod
az vm repair create --name vm-prod --resource-group rg-prod

# Network connectivity
az network watcher test-connectivity \
  --source-resource vm-prod --dest-address 10.0.2.4 --dest-port 443

# AKS issues
az aks get-credentials --resource-group rg-prod --name aks-prod
kubectl describe pod <pod>
kubectl logs <pod> --previous

# App Service logs
az webapp log tail --resource-group rg-prod --name app-prod

# Storage
az storage blob show --account-name stprodauth --container-name data --name file.txt
```

## 14. Comparison Section

### Azure vs AWS

| Service | Azure | AWS |
|---------|-------|-----|
| Compute | VM, VMSS, App Service | EC2, ASG, Elastic Beanstalk |
| Containers | AKS, Container Instances | EKS, ECS, Fargate |
| Serverless | Functions, Logic Apps | Lambda, Step Functions |
| Database | SQL DB, Cosmos DB | RDS, DynamoDB |
| Storage | Blob, Files, Disk | S3, EFS, EBS |
| Identity | Azure AD | IAM, Cognito |
| DevOps | Azure DevOps | CodePipeline |
| Monitoring | Monitor, App Insights | CloudWatch, X-Ray |

## 15. Revision Notes

```
AZURE HIERARCHY
Management Group > Subscription > Resource Group > Resource

CORE SERVICES
Compute: VM, VMSS, App Service, AKS, Functions
Storage: Blob, Disk, Files, Archive, Data Lake
Database: SQL DB, Cosmos DB, MySQL, PostgreSQL
Network: VNet, LB, App GW, VPN, ExpressRoute, Front Door
Security: Azure AD, Key Vault, Sentinel, Defender, Policy

AZURE SLA
- Compute: 99.9% (single VM), 99.99% (availability set)
- App Service: 99.95%
- SQL DB: 99.99% (business critical)
- Blob: 99.9% (LRS), 99.99% (RA-GRS)
```

## 16. Cheat Sheet

```text
+======================================================================+
|                     AZURE CHEAT SHEET                                |
+======================================================================+

  AZURE CLI
+----------------------------------------------------------------------+
| az login                              | Authenticate                 |
| az group create -n rg -l eastus       | Create resource group        |
| az vm create -n vm -g rg             | Create VM                    |
| az webapp create -n app -g rg        | Create App Service           |
| az aks create -n aks -g rg          | Create AKS cluster           |
| az storage account create -n st     | Create storage account       |
| az keyvault create -n kv -g rg      | Create Key Vault             |
| az sql server create -n sql -g rg   | Create SQL Server            |
| az functionapp create -n func -g rg  | Create Function App          |
+----------------------------------------------------------------------+

  POWERSHELL
+----------------------------------------------------------------------+
| Get-AzResourceGroup                  | List resource groups          |
| New-AzVM -ResourceGroupName rg      | Create VM                    |
| Get-AzVM -Status                     | Get VM status                |
| New-AzAksCluster -Name aks -RG rg   | Create AKS                   |
| Stop-AzVM -Name vm -RG rg           | Stop VM                      |
| Remove-AzResourceGroup -Name rg     | Delete resource group        |
+----------------------------------------------------------------------+

  SECURITY
+----------------------------------------------------------------------+
| Azure AD  | Identity and authentication                             |
| RBAC      | Role-based access control                               |
| NSG       | Network security groups (L4 firewall)                    |
| ASG       | Application security groups                             |
| Policy    | Governance enforcement                                  |
| Defender  | Cloud security posture                                  |
| Sentinel  | SIEM + SOAR                                             |
| Key Vault | Secrets, keys, certificates                             |
| Private Link | Private PaaS access                                 |
+----------------------------------------------------------------------+

  DISASTER RECOVERY
+----------------------------------------------------------------------+
| Site Recovery  | VM-level replication to paired region               |
| Geo-redundant storage | Storage auto-replicated                     |
| Cosmos DB multi-master | Global database replication                |
| Traffic Manager | Global DNS-based traffic routing                  |
+----------------------------------------------------------------------+

+======================================================================+
|  PRO TIPS: Use managed identities, never connection strings.         |
|  Tag everything for cost allocation. Use Azure Policy for gov.      |
|  Enable Defender for Cloud. Use Private Link for PaaS.              |
|  Implement Azure landing zone for enterprise scale.                 |
+======================================================================+
```
