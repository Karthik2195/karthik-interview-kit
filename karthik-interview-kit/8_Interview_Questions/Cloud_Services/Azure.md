# Azure - Senior Level Interview Questions & Answers

## Table of Contents
1. Architecture & Azure Services
2. Cost Management & Optimization
3. Security & Compliance
4. Multi-cloud Strategy
5. Enterprise Integration

---

## 1. Architecture & Azure Services

### What is Azure and how is it different from AWS?

You've worked with AWS. Now we're migrating to Azure. What are the fundamental differences, and how would you approach this migration?

Great question. Both are hyperscalers, but they have different philosophical approaches. AWS has been around longer and is more mature - they've had 15+ years to build services. Azure came later but it's deeply integrated with enterprise Microsoft software - Office 365, SQL Server, Active Directory. That's huge if you're already a Microsoft shop. From an architectural perspective, AWS uses regions and availability zones like most cloud providers. Azure has regions and availability zones too, but they also introduced Availability Sets which are older and less reliable than zones. AWS is more 'cloud-native' focused - containers, serverless, microservices. Azure is more 'enterprise datacenter as a cloud' - better for lift-and-shift migrations. The pricing model is different too. AWS charges on-demand, savings plans, or reserved instances. Azure has this concept of 'commitment-based discounts' which can be better if you have predictable workloads. From a networking perspective, Azure's Virtual Networks are similar to AWS VPCs but the terminology is confusing - Network Security Groups vs Security Groups, etc. The biggest difference is compliance. Azure excels at government and regulated industries because Microsoft has deep relationships with governments worldwide. If you're doing FedRAMP, government, healthcare, Azure is often the easier choice. From a personal perspective, I find AWS more intuitive for infrastructure engineers, but Azure better for enterprises with existing Microsoft infrastructure."

---

### Real-time scenario: Migrate 500 VMs from on-premises to Azure

Your boss asks: 'How long? How much?' You have 10 seconds to answer.

Okay, quick math. 500 VMs is substantial. Assessment phase: 4-8 weeks. We'd use Azure Migrate to scan your environment, understand dependencies, estimate costs. Migration phase: depends on network bandwidth. If you have 100 Mbps internet, you're looking at weeks of data transfer. Realistically 3-6 months. Cost is tricky. If you're replacing aging hardware, Azure might be cheaper day one. If your on-prem infrastructure is amortized, you're probably not saving money immediately - maybe $200k-500k/year in operational costs, but that depends on current licensing. Total project cost: $500k-2M depending on scope. My recommendation: Start with a pilot - migrate 20-30 critical applications first. Use Azure Site Recovery for replication - it's actually really good for this. Don't try to migrate everything at once. Also, your current licenses - Software Assurance licenses? You can often bring those to Azure for discounts. That's a hidden 40% savings most teams don't leverage."

---

### Q: Design a multi-region Azure infrastructure with disaster recovery

**A:**
```yaml
# Azure Multi-Region HA Architecture

# Primary Region: East US
resource "azurerm_resource_group" "primary" {
  name     = "rg-primary-eastus"
  location = "East US"
}

# Secondary Region: West US
resource "azurerm_resource_group" "secondary" {
  name     = "rg-secondary-westus"
  location = "West US"
}

# Azure Traffic Manager (DNS-level routing)
resource "azurerm_traffic_manager_profile" "main" {
  name                   = "tm-prod"
  resource_group_name    = azurerm_resource_group.primary.name
  traffic_routing_method = "Geographic"
  
  # Failover to secondary if primary unhealthy
  monitor_config {
    protocol                = "HTTPS"
    port                    = 443
    path                    = "/health"
    interval_in_seconds     = 10
    tolerated_number_of_failures = 3
  }
}

# Primary endpoint (East US)
resource "azurerm_traffic_manager_azure_endpoint" "primary" {
  name               = "primary-endpoint"
  profile_name       = azurerm_traffic_manager_profile.main.name
  resource_group_name = azurerm_resource_group.primary.name
  target_resource_id = azurerm_app_service.primary.id
  
  geo_mapping = ["US", "CA"]
}

# Secondary endpoint (West US)
resource "azurerm_traffic_manager_azure_endpoint" "secondary" {
  name               = "secondary-endpoint"
  profile_name       = azurerm_traffic_manager_profile.main.name
  resource_group_name = azurerm_resource_group.primary.name
  target_resource_id = azurerm_app_service.secondary.id
  
  geo_mapping = ["*"]  # Fallback for all others
}

# App Service - Primary
resource "azurerm_app_service" "primary" {
  name                = "app-primary-eastus"
  location            = azurerm_resource_group.primary.location
  resource_group_name = azurerm_resource_group.primary.name
  app_service_plan_id = azurerm_app_service_plan.primary.id
}

# App Service - Secondary
resource "azurerm_app_service" "secondary" {
  name                = "app-secondary-westus"
  location            = azurerm_resource_group.secondary.location
  resource_group_name = azurerm_resource_group.secondary.name
  app_service_plan_id = azurerm_app_service_plan.secondary.id
}

# SQL Database with Geo-Replication
resource "azurerm_sql_server" "primary" {
  name                         = "sqlserver-primary"
  location                     = azurerm_resource_group.primary.location
  resource_group_name          = azurerm_resource_group.primary.name
  administrator_login          = "sqladmin"
  administrator_login_password = random_password.sql_password.result
  version                      = "12.0"
}

resource "azurerm_sql_database" "primary" {
  name                             = "proddb"
  server_id                        = azurerm_sql_server.primary.id
  collation                        = "SQL_Latin1_General_CP1_CI_AS"
  create_mode                      = "Default"
  read_scale_out_enabled           = true
  zone_redundant                   = true
}

# Secondary replica (failover)
resource "azurerm_sql_server" "secondary" {
  name                         = "sqlserver-secondary"
  location                     = azurerm_resource_group.secondary.location
  resource_group_name          = azurerm_resource_group.secondary.name
  administrator_login          = "sqladmin"
  administrator_login_password = random_password.sql_password.result
  version                      = "12.0"
}

resource "azurerm_sql_database" "secondary" {
  name                        = "proddb"
  server_id                   = azurerm_sql_server.secondary.id
  create_mode                 = "Secondary"
  creation_source_database_id = azurerm_sql_database.primary.id
}

# Failover Group
resource "azurerm_sql_failover_group" "failover" {
  name                       = "fg-prod"
  server_name                = azurerm_sql_server.primary.name
  resource_group_name        = azurerm_resource_group.primary.name
  database_ids               = [azurerm_sql_database.primary.id]
  partner_server_id          = azurerm_sql_server.secondary.id
  read_write_endpoint_failover_policy {
    mode          = "Automatic"
    grace_minutes = 60
  }
}
```

---

## 2. Cost Management & Optimization

### Topic: Cost Audit & Optimization

Your Azure bill is $100k/month. Finance wants to know why and how to reduce it. Walk me through your systematic audit process.

First thing - go to Cost Management + Billing. Azure's cost analysis tool actually shows you a breakdown by service. I'd look for these cost drivers: 1) Virtual Machines - probably 30-40% of the bill. The question is: which VMs are actually being used? I'd look at CPU and network metrics. VMs at 5% CPU for months? Shut them down or right-size. 2) Storage - especially blob storage. If you have old backups sitting there, that costs money. I'd set lifecycle policies to move old data to archive tiers, that's 90% cheaper. 3) Data transfer costs - similar to AWS, egress is expensive. Make sure you're using Azure CDN and local caching. 4) Database costs - SQL Database and Cosmos DB are expensive. Consider if you're over-provisioning. Reserved Capacity is huge here - if you commit for 3 years, you get 65% discount. 5) Compute sizing - are you running B2s when B1s would work? 6) Unused resources - unattached disks, abandoned app services, old databases. Azure has an 'Advisor' that flags this. Here's my systematic approach: Use Cost Analysis by service, drill into top 3 cost drivers, check if reserved capacity is applicable, look for unused resources via Advisor, consolidate small workloads. My guess is you could cut that bill by 30-40% just by being thoughtful about resource sizing and cleanup."

---

### Q: Design a cost optimization strategy for a large Azure deployment

**A:**
```python
# Azure Cost Optimization Script

import os
from azure.identity import DefaultAzureCredential
from azure.mgmt.compute import ComputeManagementClient
from azure.mgmt.storage import StorageManagementClient
from datetime import datetime, timedelta

class AzureCostOptimizer:
    def __init__(self):
        credential = DefaultAzureCredential()
        self.subscription_id = os.environ['AZURE_SUBSCRIPTION_ID']
        self.compute_client = ComputeManagementClient(credential, self.subscription_id)
        self.storage_client = StorageManagementClient(credential, self.subscription_id)
    
    def find_underutilized_vms(self):
        """Find VMs with low CPU utilization"""
        underutilized = []
        
        for resource_group in self.compute_client.resource_groups.list():
            for vm in self.compute_client.virtual_machines.list(resource_group.name):
                # Check metrics
                metrics = self.get_vm_metrics(resource_group.name, vm.name)
                
                if metrics['cpu_utilization'] < 5:
                    underutilized.append({
                        'name': vm.name,
                        'resource_group': resource_group.name,
                        'vm_size': vm.hardware_profile.vm_size,
                        'cpu_utilization': metrics['cpu_utilization'],
                        'recommendation': 'Downsize or delete',
                        'estimated_savings': self.estimate_savings(vm.hardware_profile.vm_size)
                    })
        
        return underutilized
    
    def optimize_storage_lifecycle(self):
        """Set lifecycle policies for storage optimization"""
        storage_optimizations = []
        
        for resource_group in self.storage_client.resource_groups.list():
            for storage_account in self.storage_client.storage_accounts.list_by_resource_group(resource_group.name):
                # Recommend lifecycle policy
                storage_optimizations.append({
                    'storage_account': storage_account.name,
                    'recommendation': 'Enable lifecycle management',
                    'policy': {
                        'hot_to_cool': '30 days',  # Move to Cool tier after 30 days
                        'cool_to_archive': '90 days',  # Move to Archive tier after 90 days
                        'delete': '365 days'  # Delete after 1 year
                    },
                    'estimated_savings': '60% reduction in storage costs'
                })
        
        return storage_optimizations
    
    def reserve_capacity_recommendations(self):
        """Recommend reserved capacity for consistent workloads"""
        recommendations = [
            {
                'resource_type': 'VM',
                'current_cost': '$50,000/month',
                'reserved_capacity_3yr': '$18,000/month',
                'savings': '64% = $32,000/month',
                'commitment': '3 years'
            },
            {
                'resource_type': 'SQL Database',
                'current_cost': '$15,000/month',
                'reserved_capacity_3yr': '$5,250/month',
                'savings': '65% = $9,750/month',
                'commitment': '3 years'
            }
        ]
        return recommendations
    
    def find_unused_resources(self):
        """Identify unused resources"""
        unused = []
        
        # Unused disks
        for resource_group in self.compute_client.resource_groups.list():
            for disk in self.compute_client.disks.list_by_resource_group(resource_group.name):
                if not disk.managed_by:  # Unattached disk
                    unused.append({
                        'type': 'Disk',
                        'name': disk.name,
                        'cost': '$5/month',
                        'action': 'Delete'
                    })
        
        return unused
    
    def estimate_savings(self, vm_size):
        vm_pricing = {
            'Standard_D4s_v3': 195,
            'Standard_D2s_v3': 98,
            'Standard_B1ms': 12
        }
        
        current = vm_pricing.get(vm_size, 100)
        downsize = current * 0.5
        return current - downsize

# Usage
optimizer = AzureCostOptimizer()
print("Underutilized VMs:", optimizer.find_underutilized_vms())
print("Storage Optimization:", optimizer.optimize_storage_lifecycle())
print("Reserved Capacity Savings:", optimizer.reserve_capacity_recommendations())
print("Unused Resources:", optimizer.find_unused_resources())
```

---

## 3. Security & Compliance

### Topic: Zero-Trust Security Architecture

Your company wants zero-trust architecture on Azure. Walk me through implementation.

Zero-trust is about 'never trust, always verify' - assume every request is untrusted. In Azure, the foundation is Azure Active Directory (AAD) - this is where authentication and authorization happen. Every access attempt goes through AAD. First, enforce multi-factor authentication on everything. No more passwords. Hardware keys preferred, then authenticator apps. Second, use Conditional Access policies. This is Azure's policy engine - you can say 'if user logs in from China, require extra verification' or 'if device isn't compliant, block access'. Third, use Azure AD B2B for external users - fine-grained permissions, no shared accounts. Fourth, network level - use Network Security Groups and Azure Firewall to filter traffic. No open ports, everything filtered. Fifth, data level - use Azure Key Vault for all secrets, never hardcoded credentials. Sixth, monitoring - Azure Sentinel for threat detection, audit all access attempts. In practice: AAD for identity, Conditional Access for policies, Network Security Groups for network filtering, Key Vault for secrets, Sentinel for monitoring. That's your zero-trust stack. Most enterprises do 80% of this and claim zero-trust - real zero-trust requires all five layers."

---

### Q: Design enterprise-grade identity and access management on Azure

**A:**
```yaml
# Azure AD Zero-Trust Architecture

# 1. Multi-factor Authentication
resource "azurerm_mfa_server" "main" {
  name     = "mfa-server"
  location = "East US"
}

# 2. Conditional Access Policies
resource "azuread_conditional_access_policy" "geographic_restriction" {
  display_name = "Block High-Risk Countries"
  state        = "enabled"
  
  conditions {
    client_app_types    = ["browser", "mobileAppsAndDesktopClients"]
    sign_in_risk_levels = ["medium", "high"]
    
    locations {
      included_locations = ["all"]
      excluded_locations = ["US", "CA", "GB"]
    }
  }
  
  grant_controls {
    operator          = "OR"
    built_in_controls = ["block"]
  }
}

# 3. Role-Based Access Control (RBAC)
resource "azurerm_role_assignment" "example" {
  scope              = azurerm_resource_group.example.id
  role_definition_name = "Contributor"
  principal_id       = azuread_group.developers.object_id
  
  # Time-limited access
  not_before = timestamp()
  not_after  = timeadd(timestamp(), "8760h")  # 1 year
}

# 4. Azure Key Vault for Secrets
resource "azurerm_key_vault" "main" {
  name                        = "kv-prod"
  location                    = azurerm_resource_group.main.location
  resource_group_name         = azurerm_resource_group.main.name
  enabled_for_disk_encryption = true
  enabled_for_secret_retrieval = true
  purge_protection_enabled    = true
  soft_delete_retention_days  = 90
  
  access_policy {
    tenant_id = data.azurerm_client_config.current.tenant_id
    object_id = azuread_group.admins.object_id
    
    secret_permissions = [
      "Get", "List", "Set", "Delete", "Recover", "Backup", "Restore"
    ]
  }
}

# 5. Azure Sentinel for Threat Detection
resource "azurerm_sentinel_alert_rule" "brute_force" {
  name                       = "Detect Brute Force"
  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  
  display_name = "Multiple failed login attempts"
  severity     = "High"
  enabled      = true
  
  query = <<-QUERY
    SigninLogs
    | where ResultType != 0
    | summarize FailureCount=count() by UserPrincipalName, ClientAppUsed
    | where FailureCount > 10
  QUERY
}
```

---

## 4. Multi-cloud Strategy

### Topic: Multi-Cloud Architecture & Strategy

You want to run workloads on AWS, Azure, and Google Cloud simultaneously for resilience. How do you manage this complexity?

Multi-cloud is complex but viable. The key principle is workload independence - never build something that requires all three clouds working. Here's how I'd approach it: First, identity management - use a cloud-agnostic identity provider like Okta or Auth0, not built into any single cloud. Second, containerization - everything in Kubernetes. Kubernetes runs the same on AWS (EKS), Azure (AKS), and Google Cloud (GKE). You can schedule workloads across clouds. Third, data - this is tricky. Use cloud-agnostic databases like PostgreSQL, not cloud-specific ones like DynamoDB. Or use multi-master replication if you need the same data in all three clouds. Fourth, networking - use a service mesh like Istio or Linkerd that abstracts away cloud-specific networking. Fifth, CI/CD - single pipeline that deploys to all three clouds. Terraform or Pulumi for infrastructure. Sixth, monitoring - use Datadog or similar that aggregates across clouds. Seventh, cost - this is your biggest pain point. Each cloud has different pricing, different reserved instance terms. You need a cost allocation model so you know which business unit is paying for what across all three clouds. Real talk: multi-cloud is about 30% more expensive operationally than single-cloud, but your risk is distributed. If AWS has an outage, you have GCP and Azure as fallback."

---

## 5. Enterprise Integration

### Topic: Enterprise Legacy System Migration

Your company uses SAP, Oracle ERP, and Salesforce on-premises. How do you migrate these to Azure?

Enterprise migrations are complex. SAP and Oracle are the toughest because they're mission-critical. My strategy: Phase 1 (Months 1-2): Assessment. SAP Rapid Cloud Migration (RCM) program helps assess. Oracle has their own cloud assessment tools. Understand your licensing - SAP can use their Azure offer, Oracle has hybrid agreements. Phase 2 (Months 3-6): Pilot migration. Migrate a non-critical system first - maybe a dev or test environment. Use Azure Site Recovery for replication. Phase 3 (Months 6-12): Production cutover. You'll need a big-bang cutover weekend - SAP can't be partially migrated. Phase 4 (Month 12+): Optimization. Move to cloud-native where possible. Key considerations: 1) Licensing - huge cost factor. Check if you have Software Assurance - that gives you Azure discounts. 2) Performance - cloud is fast but latency matters for SAP. Put it in same region as users. 3) Support - SAP and Oracle support changes when you move to cloud. 4) Network - dedicated ExpressRoute connection for stability. 5) High Availability - use availability sets or zones depending on SLA. Timeline: 12-18 months for large enterprise. Cost: $5-10M depending on scale and complexity. My advice: Hire a Microsoft Premier Partner with SAP expertise. This isn't a DIY project."

---

## Summary

**Critical Senior Azure Skills:**
1. ✅ Multi-region HA architecture
2. ✅ Azure services ecosystem
3. ✅ Cost optimization strategies
4. ✅ Zero-trust security model
5. ✅ Multi-cloud orchestration
6. ✅ Enterprise migrations
7. ✅ Compliance & governance
8. ✅ Hybrid cloud with on-premises

*Created by Karthik Reddy*
