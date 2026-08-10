# Azure Governance Framework

**Version:** 1.0  
**Last Updated:** 2026-08-10  
**Owner:** Cloud Center of Excellence (CCoE)  
**Status:** Active

---

## Executive Summary

This document establishes the comprehensive governance framework for enterprise Azure deployments. The framework provides standardized controls, policies, and procedures to ensure consistency, compliance, security, and cost optimization across all Azure subscriptions and resources. It defines organizational structures, access controls, resource standards, and measurable KPIs to maintain operational excellence.

**Key Principles:**
- **Centralized Control:** Governance applied at the management group level for scalability
- **Defense in Depth:** Layered controls combining policy, RBAC, and networking
- **Least Privilege:** Minimal access rights with just-in-time elevation
- **Compliance by Default:** Resources compliant with policies at creation time
- **Continuous Monitoring:** KPIs tracked and reviewed monthly

---

## 1. Management Groups Architecture

### 1.1 Organizational Structure

The management group hierarchy provides the foundation for policy enforcement and subscription organization.

```
Root Management Group
├── Platform (Platform resources and shared services)
│   ├── Connectivity (Network and security)
│   ├── Shared-Services (Common platform services)
│   ├── Identity (Identity and access management)
│   └── Management (Monitoring and compliance)
├── Workloads (Application and business workloads)
│   ├── Production (Production applications)
│   ├── Non-Production (Development, Test, Staging)
│   └── Sandbox (Experimentation and proof-of-concept)
├── Security (Security operations and governance)
│   ├── Security-Operations (SOC and threat management)
│   └── Compliance (Audit and compliance)
└── Decommissioned (Archived and end-of-life workloads)
```

### 1.2 Management Group Policies

| Management Group | Policy Scope | Key Policies Applied |
|------------------|--------------|---------------------|
| **Root** | Enterprise-wide | Allowed regions, Audit logging, Tagging enforcement |
| **Platform** | Core infrastructure | Network segmentation, Encryption mandates, Service limits |
| **Workloads** | Application layer | Workload-specific policies, Cost controls, Compliance |
| **Production** | Production only | High availability, Backup requirements, DR testing |
| **Security** | Security controls | Defender enforcement, Sentinel integration, Access logging |

### 1.3 Management Group Ownership

| Management Group | Owner | Responsibilities |
|------------------|-------|------------------|
| Root | CCoE Lead | Strategic governance, Global policy alignment |
| Platform | Infrastructure Lead | Network, storage, compute platforms |
| Workloads | Application Lead | Workload policies, cost governance, SLA management |
| Security | Security Lead | Compliance, threat detection, incident response |

---

## 2. Subscription Design

### 2.1 Subscription Taxonomy

**Naming Convention:** `{environment}-{workload}-{region}-{sequence}`  
**Example:** `prod-erp-eastus-001`, `dev-analytics-westeurope-001`

#### Platform Subscriptions

| Name | Purpose | Budget | Owner | SLA |
|------|---------|--------|-------|-----|
| `platform-connectivity-global-001` | Hub VNet, Azure Firewall, Bastion, DNS | $50K/month | Infrastructure | 99.95% |
| `platform-shared-services-001` | Key Vault, ACR, Storage, Identity services | $30K/month | Infrastructure | 99.95% |
| `platform-management-001` | Monitoring, Log Analytics, Automation | $20K/month | Operations | 99.9% |
| `platform-identity-001` | Azure AD, PIM, MFA infrastructure | $15K/month | Security | 99.99% |

#### Workload Subscriptions

| Environment | Naming | Budget | Owner | Retention |
|-------------|--------|--------|-------|-----------|
| **Production** | `prod-{workload}-{region}-001` | 60% of total | App Owner | Permanent |
| **Staging** | `stg-{workload}-{region}-001` | 20% of total | DevOps Lead | 90 days |
| **Development** | `dev-{workload}-{region}-001` | 15% of total | Dev Lead | 30 days |
| **Sandbox** | `sandbox-{team}-{region}-001` | 5% of total | Team Lead | 14 days |

### 2.2 Subscription Limits & Quotas

| Resource | Limit | Monitoring | Alert Threshold |
|----------|-------|-----------|-----------------|
| VNets per subscription | 50 | Monthly | 40 |
| VMs per subscription | 1000 | Weekly | 800 |
| Storage accounts per subscription | 250 | Monthly | 200 |
| App Service Plans per region | 100 | Monthly | 80 |
| Cosmos DB accounts | 50 | Monthly | 40 |

### 2.3 Subscription Access Control

**Subscription Owner:** Application owner with business accountability  
**Subscription Contributors:** Infrastructure teams (limited scope via RBAC)  
**Subscription Readers:** Operations, Compliance, Finance teams  

Access provisioned via Azure AD security groups with:
- 90-day access reviews for all privileged roles
- Just-in-time (JIT) elevation for root-level changes
- Change tickets required for any access modification

---

## 3. Resource Group Standards

### 3.1 Resource Group Naming

**Format:** `{environment}-{workload}-{component}-{region}`

**Examples:**
- `prod-erp-frontend-eastus`
- `dev-analytics-database-westeurope`
- `sandbox-poc-networking-eastus2`

### 3.2 Resource Group Organization

**Maximum resources per Resource Group:** 500  
**Minimum components per Environment:**

```
Production Environment:
  prod-{app}-web-{region}           (Web tier resources)
  prod-{app}-api-{region}           (API tier resources)
  prod-{app}-data-{region}          (Database resources)
  prod-{app}-storage-{region}       (Storage resources)
  prod-{app}-monitoring-{region}    (Monitoring resources)
  prod-{app}-backup-{region}        (Backup resources)

Non-Production Environment:
  dev-{app}-all-{region}            (All resources combined)
```

### 3.3 Resource Group Lifecycle

| Stage | Duration | Action | Owner |
|-------|----------|--------|-------|
| **Created** | Day 1 | Automatic tags applied, policy evaluation | System |
| **Active** | Days 2-89 | Regular monitoring, cost optimization reviews | App Owner |
| **Review** | Day 90 | Access review, cost validation | CCoE |
| **Archived** | Days 91-180 | Move to decommissioned subscription, retain backups | App Owner |
| **Deleted** | Day 181 | Hard delete after backup verification | Infrastructure |

---

## 4. Azure Policy Framework

### 4.1 Policy Categories & Deployment

#### Category 1: Mandatory (Deny) Policies
Enforced at Root Management Group - violations prevent resource creation

| Policy | Scope | Effect | Details |
|--------|-------|--------|---------|
| Allowed Resource Types | Root | Deny | Only approved resource types allowed |
| Allowed Locations | Root | Deny | Deployments only in EU regions (compliance) |
| Allowed VM SKUs | Root | Deny | Only cost-optimized SKUs permitted |
| Encryption at Rest | Root | Deny | Storage and DBs must have encryption enabled |
| TLS/SSL Requirement | Root | Deny | HTTPS/TLS 1.2+ enforced on all services |
| Network Isolation | Platform | Deny | Private endpoints required for PaaS services |

#### Category 2: Audit & Modify Policies
Logged for compliance and automatically remediated

| Policy | Effect | Target | Remediation |
|--------|--------|--------|-------------|
| Enable monitoring on all resources | Audit | All | Create Log Analytics integration |
| Add mandatory tags | Modify | All | Auto-apply default tags |
| Enable Azure Backup | Audit | VMs/Databases | Alert for manual remediation |
| Enable Disk Encryption | Modify | VMs | Enable encryption on unencrypted disks |
| Enable Network Security Groups | Audit | Subnets | Log for review and manual application |

#### Category 3: Governance Policies
Control and standardize resource configuration

| Policy | Scope | Objective |
|--------|-------|-----------|
| Require Cost Center Tag | All | Enable chargeback and cost allocation |
| Enforce Resource Location | Regional | Reduce latency and data residency compliance |
| Limit Resource Size | Production | Prevent over-provisioning and cost overruns |
| Enforce Backup Retention | Production | Ensure data protection and RTO/RPO compliance |
| Require Resource Locks | Production | Prevent accidental deletion or modification |

### 4.2 Policy Initiative: Landing Zone Compliance

**Initiative Name:** `lz-governance-baseline`  
**Target:** Workloads Management Group and below

**Included Policies:**
```
1. Allowed locations (EU-only)
2. Allowed VM SKUs (cost-optimized list)
3. Encryption at rest (mandatory)
4. Network isolation (private endpoints)
5. Logging and monitoring (Log Analytics)
6. Tagging enforcement (cost center, owner)
7. Resource naming compliance
8. Backup requirements (prod only)
9. Disaster recovery testing (prod only)
10. Access logging and audit
```

### 4.3 Exemptions & Exceptions

**Exemption Process:**
1. **Request:** Submit exemption request with business justification
2. **Review:** CCoE Lead approval required (max 90 days)
3. **Documentation:** Record exemption in central registry
4. **Expiration:** Automatic re-evaluation after 90 days
5. **Cleanup:** Delete exemption if no longer needed

**Exemption Registry Location:** Azure Policy > Exemptions > Central tracking log

---

## 5. RBAC (Role-Based Access Control) Model

### 5.1 Custom Roles Definition

#### Role 1: Application Owner
**Scope:** Subscription-level  
**Purpose:** Full operational management of application resources

**Permissions:**
- Create, read, update, delete resources within subscription
- Manage role assignments for Dev and Ops teams
- Cannot delete subscription or create network peering
- Cannot modify Azure Policy or management group settings

**Typical Assignment:** Application product owner, 12-month term

#### Role 2: Infrastructure Administrator
**Scope:** Resource Group-level  
**Purpose:** Infrastructure and platform management

**Permissions:**
- Full control of compute, storage, network resources
- Configure backup and disaster recovery
- Cannot modify identity or security services
- Cannot assign roles above current scope

**Typical Assignment:** Infrastructure team lead, 12-month term

#### Role 3: DevOps Engineer
**Scope:** Resource Group-level  
**Purpose:** CI/CD pipeline operations and deployment automation

**Permissions:**
- Deploy resources using ARM/Bicep templates
- Manage compute and container services (AKS, App Service)
- Read-only access to security and monitoring services
- Cannot modify network or storage policies

**Typical Assignment:** DevOps team members, 12-month term

#### Role 4: Security Analyst
**Scope:** Subscription-level  
**Purpose:** Security monitoring and compliance verification

**Permissions:**
- Read access to all security, monitoring, and audit logs
- Configure Microsoft Defender for Cloud policies
- Create and manage security alerts
- Cannot modify production resources

**Typical Assignment:** Security team members, 12-month term

#### Role 5: Auditor/Compliance Officer
**Scope:** Management Group-level  
**Purpose:** Compliance verification and audit logging

**Permissions:**
- Read-only access to all subscriptions within scope
- Access to audit logs and compliance records
- Cannot perform any resource modifications

**Typical Assignment:** Compliance and audit teams, 12-month term

### 5.2 Built-In Roles Usage

| Built-In Role | Scope | Usage | Restrictions |
|---------------|-------|-------|-------------|
| Owner | Subscription | Platform subscriptions only | Requires MFA + PIM |
| Contributor | Resource Group | Application resources | Time-limited via PIM |
| Reader | Subscription | Broad read access | 6-month review cycle |
| Virtual Machine Contributor | Resource Group | VM operations | Automatic password refresh |
| Network Contributor | Resource Group | Network operations | Requires change ticket |
| Storage Account Contributor | Resource Group | Storage operations | Audit all modifications |

### 5.3 Access Control Matrix

| Role | Scope | Subscription | Resource Group | Duration | MFA | PIM |
|------|-------|--------------|----------------|----------|-----|-----|
| Application Owner | App Subscription | Owner | Contributor | 12 mo | ✅ | ❌ |
| Infrastructure Lead | Platform | Contributor | N/A | 12 mo | ✅ | ✅ |
| DevOps Engineer | App Subscription | N/A | Contributor | 12 mo | ✅ | ❌ |
| Security Analyst | Root | Reader | N/A | 12 mo | ✅ | ✅ |
| Auditor | Root | Reader | N/A | 12 mo | ❌ | ❌ |

### 5.4 PIM (Privileged Identity Management)

**Eligible vs. Active Roles:**
- All privileged roles provisioned as **Eligible** (not Active)
- Users must **activate** roles with justification
- Activation duration limited to 8 hours
- Activations logged and reviewed monthly

**PIM Configuration:**

| Setting | Value | Purpose |
|---------|-------|---------|
| Require MFA | Yes | Enhanced security for privilege elevation |
| Require justification | Yes | Audit trail and accountability |
| Activation duration | 8 hours | Least privilege principle |
| Approval required | For subscription level | Executive oversight |
| Recurring review | 90 days | Ensure access still needed |

---

## 6. Naming Conventions

### 6.1 General Naming Rules

**Requirements:**
- Lowercase letters only (except where service requires uppercase)
- Alphanumeric and hyphens only (no underscores or special chars)
- 3-24 characters for most resources
- Start with letter, end with alphanumeric
- No spaces or consecutive hyphens

### 6.2 Resource Type Naming

| Resource Type | Pattern | Example | Max Length |
|---|---|---|---|
| Resource Group | `{env}-{workload}-{component}-{region}` | `prod-erp-web-eastus` | 90 |
| Storage Account | `{env}{workload}{component}{random}` | `proderpdisk1a2b3c` | 24 |
| Virtual Network | `{env}-{workload}-vnet-{region}` | `prod-erp-vnet-eastus` | 80 |
| Subnet | `{env}-{workload}-snet-{purpose}-{region}` | `prod-erp-snet-frontend-eastus` | 80 |
| Network Security Group | `{env}-{workload}-nsg-{purpose}` | `prod-erp-nsg-web` | 80 |
| Virtual Machine | `{env}{workload}{purpose}{number}` | `proderpoapp01` | 15 |
| App Service Plan | `{env}-{workload}-asp-{region}` | `prod-erp-asp-eastus` | 40 |
| App Service/Web App | `{env}-{workload}-app-{region}` | `prod-erp-app-eastus` | 60 |
| SQL Database | `{env}-{workload}-sqldb-{region}` | `prod-erp-sqldb-eastus` | 128 |
| SQL Server | `{env}{workload}sqlsrv{region}` | `proderpsqlsrveastus` | 63 |
| Key Vault | `{env}-{workload}-kv-{region}` | `prod-erp-kv-eastus` | 24 |
| Application Insights | `{env}-{workload}-appi-{region}` | `prod-erp-appi-eastus` | 260 |
| Log Analytics Workspace | `{env}-{workload}-law-{region}` | `prod-erp-law-eastus` | 63 |
| Container Registry | `{env}{workload}acr{region}` | `proderpacreastus` | 50 |
| AKS Cluster | `{env}-{workload}-aks-{region}` | `prod-erp-aks-eastus` | 63 |

### 6.3 Environment Abbreviations

| Environment | Code | Use Case | Retention |
|---|---|---|---|
| Production | `prod` | Customer-facing workloads | Permanent |
| Staging | `stg` | Pre-production validation | 90 days |
| Development | `dev` | Active development and testing | 30 days |
| Sandbox | `sandbox` | Experimentation and PoC | 14 days |
| Backup/Archive | `bkup` | Archived resources | As needed |

### 6.4 Region Abbreviations

| Azure Region | Code |
|---|---|
| East US | eastus |
| West US | westus |
| West Europe | westeurope |
| North Europe | northeurope |
| East US 2 | eastus2 |
| South Central US | southcentralus |
| Central India | centralindia |
| Southeast Asia | southeastasia |

### 6.5 Workload Abbreviations

| Workload | Code | Examples |
|---|---|---|
| Enterprise Resource Planning | erp | prod-erp-web-eastus |
| Analytics/Data Warehouse | analytics | dev-analytics-dwh-eastus |
| CRM System | crm | prod-crm-api-westeurope |
| E-Commerce | ecom | prod-ecom-app-eastus |
| Mobile Backend | mobile | dev-mobile-api-eastus |
| Data Platform | data | prod-data-lake-eastus |

---

## 7. Tagging Standards

### 7.1 Mandatory Tags (All Resources)

| Tag Key | Value Format | Purpose | Owner Updates |
|---|---|---|---|
| `environment` | prod/stg/dev/sandbox | Environment classification | On promotion |
| `workload` | {workload-name} | Business workload identifier | Annual review |
| `cost-center` | {cc-code} | Finance cost allocation | On reassignment |
| `owner` | {name/team} | Resource accountability | Annual review |
| `created-date` | YYYY-MM-DD | Creation tracking for cleanup | System-generated |
| `created-by` | {user-id} | Change accountability | System-generated |

**Example:**
```json
{
  "environment": "prod",
  "workload": "erp",
  "cost-center": "FIN-001",
  "owner": "john.smith@company.com",
  "created-date": "2026-08-10",
  "created-by": "azure-automation@company.com"
}
```

### 7.2 Optional But Recommended Tags

| Tag Key | Value Format | Purpose | Usage |
|---|---|---|---|
| `project` | {project-code} | Project tracking | Finance/PMO |
| `compliance` | {standard-list} | Compliance framework (ISO27001, SOC2) | Legal/Compliance |
| `backup-policy` | daily/weekly/monthly | Backup frequency | Operations |
| `dr-tier` | critical/important/best-effort | Disaster recovery tier | Operations |
| `data-classification` | public/internal/confidential/restricted | Data sensitivity | Security/Data Gov |
| `tech-stack` | {technologies} | Application technology stack | Architecture |
| `version` | {version-number} | Resource/application version | DevOps |
| `scheduled-shutdown` | yes/no | Auto-shutdown eligibility | Finance |

### 7.3 Tag Compliance Policy

| Scenario | Action | Timeline | Owner |
|---|---|---|---|
| **Missing mandatory tag** | Policy audit triggered, resource flagged | Day 1 | System |
| **Tag value invalid** | Audit alert, manual remediation requested | Day 5 | App Owner |
| **No remediation** | Resource suspended/scaled down | Day 30 | Operations |
| **Persistent non-compliance** | Resource deletion (non-prod) or escalation (prod) | Day 60 | CCoE |

### 7.4 Tag Query Examples

**Azure Resource Graph queries for governance:**

```kusto
// Find resources without cost-center tag
resources
| where tags.['cost-center'] == '' or tags.['cost-center'] == null
| project name, type, resourceGroup

// Calculate spend by cost center
resources
| join kind=inner (costs | project resourceId, totalCost) on resourceId
| summarize TotalCost = sum(totalCost) by tostring(tags.['cost-center'])
| sort by TotalCost desc

// Find resources over 90 days old without review
resources
| where todatetime(tags.['created-date']) < now(-90d)
| project name, tags.['created-date'], resourceGroup
```

---

## 8. Governance KPIs & Metrics

### 8.1 Policy Compliance KPIs

| KPI | Target | Frequency | Owner | Alert Threshold |
|---|---|---|---|---|
| **Policy Compliance Rate** | ≥ 99% | Weekly | CCoE | < 95% |
| **Denied Resource Deployments** | < 5/week | Weekly | Infrastructure | > 10/week |
| **Policy Exemptions Outstanding** | ≤ 5 active | Monthly | CCoE | > 10 |
| **Exemptions Past Expiration** | 0 | Weekly | CCoE | Any overdue |
| **Non-Compliant Resources** | ≤ 1% of total | Weekly | Infrastructure | > 2% |

### 8.2 RBAC & Access Control KPIs

| KPI | Target | Frequency | Owner | Alert Threshold |
|---|---|---|---|---|
| **Privileged Role Review Completion** | 100% within 90 days | Quarterly | Security | < 90% |
| **Overdue PIM Activations** | 0 | Daily | Security | Any overdue |
| **MFA Enrollment Rate** | 100% for privileged users | Monthly | Identity | < 95% |
| **Orphaned Role Assignments** | 0 | Quarterly | Security | Any found |
| **Access Review Cycle Time** | < 30 days | Quarterly | Security | > 45 days |

### 8.3 Resource & Tagging KPIs

| KPI | Target | Frequency | Owner | Alert Threshold |
|---|---|---|---|---|
| **Resources with Complete Tags** | ≥ 98% | Weekly | Operations | < 90% |
| **Untagged Resources** | ≤ 2% of total | Weekly | Operations | > 5% |
| **Naming Convention Compliance** | ≥ 97% | Weekly | Operations | < 90% |
| **Orphaned Resources (No owner)** | ≤ 1% | Monthly | Operations | > 2% |
| **Resource Group Utilization** | 200-400 resources avg | Monthly | Operations | > 500 |

### 8.4 Cost & Billing KPIs

| KPI | Target | Frequency | Owner | Alert Threshold |
|---|---|---|---|---|
| **Budget Variance** | ± 10% of forecast | Monthly | Finance | ± 20% |
| **Cost per Workload Deviation** | < 15% MoM | Monthly | Finance | > 25% |
| **Reserved Instance Coverage** | ≥ 60% of compute | Monthly | Finance | < 50% |
| **Unallocated Cost (No Cost Center)** | < 5% | Weekly | Finance | > 10% |
| **Resource Utilization (CPU/Memory)** | 40-70% average | Weekly | Operations | < 20% or > 85% |

### 8.5 Governance Dashboard

**Location:** Azure Monitor > Dashboards > Governance-KPI-Dashboard

**Key Visualizations:**
1. Policy Compliance Trend (30-day rolling average)
2. Privileged Role Assignments by Scope
3. Resource Compliance by Resource Type
4. Tag Compliance by Workload
5. Cost by Cost Center (actual vs. budget)
6. Exemption Status and Expiration Timeline
7. Access Reviews Completion Rate
8. Resource Count Trend by Environment

**Refresh Frequency:** Real-time, updated hourly

---

## 9. Compliance & Audit Trail

### 9.1 Audit Log Requirements

All governance actions logged to:
- **Azure Activity Log:** Native Azure events (policy applications, RBAC changes)
- **Azure AD Audit Logs:** Identity and access changes
- **Microsoft Sentinel:** Centralized SIEM for correlation and analysis

**Retention:** 90 days live, 7 years archived in Cold Storage

### 9.2 Regular Audit Cycles

| Audit Type | Frequency | Scope | Owner | Deliverable |
|---|---|---|---|---|
| **Policy Compliance Audit** | Monthly | All subscriptions | CCoE | Compliance report |
| **RBAC Access Review** | Quarterly | All privileged roles | Security | Access certification |
| **Resource Tagging Audit** | Weekly | All new resources | Operations | Non-compliance report |
| **Cost Center Allocation Audit** | Monthly | All resources | Finance | Cost allocation report |
| **Naming Convention Audit** | Monthly | All new resources | Operations | Naming violations list |
| **Disaster Recovery Testing** | Quarterly | Production only | Operations | DR test results |

### 9.3 Compliance Reporting

**Monthly Governance Report includes:**
- Policy compliance status
- RBAC and access control metrics
- Resource compliance and tagging status
- Cost and budgeting performance
- Identified exceptions and remediation status
- KPI trends and forecasts
- Executive summary and recommendations

---

## 10. Governance Review & Evolution

### 10.1 Review Schedule

| Review Cycle | Frequency | Attendees | Scope |
|---|---|---|---|
| **Tactical Review** | Weekly | Operations Lead, Infrastructure Lead | Policy violations, urgent issues |
| **Operational Review** | Monthly | CCoE Team, App Owners | KPI performance, remediation status |
| **Strategic Review** | Quarterly | CCoE Lead, Security Lead, Finance Lead | Governance effectiveness, policy updates |
| **Annual Assessment** | Annually | Executive Steering Committee | Framework effectiveness, alignment with business goals |

### 10.2 Policy Update Process

1. **Identify Need:** Monthly review identifies policy gaps or required updates
2. **Design:** CCoE team drafts new/updated policy
3. **Pilot:** Deploy to non-production (Dev/Sandbox) for testing (2-4 weeks)
4. **Communicate:** Announce changes to all stakeholders 2 weeks before prod deployment
5. **Deploy:** Roll out to production with exemption allowances
6. **Monitor:** Track policy impact and exemption requests for 30 days
7. **Refine:** Adjust based on feedback and compliance data

### 10.3 Continuous Improvement

**Feedback Channels:**
- Monthly governance review meetings
- Ticketed exceptions and exemptions
- Quarterly all-hands survey (governance effectiveness)
- Annual audit findings and recommendations

**Improvement Actions:**
- Policy simplification when exemptions exceed 10%
- Enhanced automation when manual processes exceed 20% overhead
- Role refinement based on access request patterns
- Tag schema updates based on cost allocation feedback

---

## 11. Governance Tooling & Automation

### 11.1 Tools & Services

| Tool | Purpose | Integration |
|---|---|---|
| **Azure Policy** | Compliance enforcement | ARM templates, Compliance Manager |
| **Azure RBAC** | Access control | Azure AD, PIM |
| **Azure Cost Management** | Cost tracking and allocation | Finance systems |
| **Microsoft Sentinel** | Security monitoring | Defender for Cloud |
| **Azure Monitor** | Operational monitoring | Log Analytics |
| **Azure Blueprints** | Subscription templates | Policy and RBAC |
| **Azure Resource Graph** | Resource querying | Dashboards and reports |
| **Azure Automation** | Workflow automation | Tag remediation, policy deployment |

### 11.2 Automation Runbooks

| Runbook | Trigger | Action | Frequency |
|---|---|---|---|
| Tag Enforcement | New resource created | Apply missing tags | Real-time |
| Compliance Check | Weekly schedule | Audit policy compliance, generate report | Weekly |
| Access Review Reminder | PIM expiration | Notify owners of pending expiration | 30 days before |
| Cost Alert | Budget threshold | Alert on budget overrun (80%, 100%) | Real-time |
| Resource Cleanup | 30+ days idle | Mark for deletion, notify owner | Weekly |

---

## 12. Appendices

### 12.1 Policy Template

```json
{
  "name": "policy-require-tagging",
  "displayName": "Require Tag on Resources",
  "description": "Enforce mandatory tagging on all resources",
  "mode": "indexed",
  "metadata": {
    "version": "1.0.0",
    "category": "Governance"
  },
  "parameters": {
    "tagName": {
      "type": "String",
      "metadata": {
        "displayName": "Tag Name",
        "description": "Name of the tag to require"
      }
    }
  },
  "policyRule": {
    "if": {
      "field": "[concat('tags[', parameters('tagName'), ']')]",
      "exists": "false"
    },
    "then": {
      "effect": "deny"
    }
  }
}
```

### 12.2 RBAC Role Definition Template

```json
{
  "name": "Custom Application Owner",
  "description": "Application owner with operational management capabilities",
  "type": "CustomRole",
  "permissions": [
    {
      "actions": [
        "Microsoft.*/read",
        "Microsoft.Compute/*",
        "Microsoft.Storage/*",
        "Microsoft.Network/*",
        "Microsoft.Web/*"
      ],
      "notActions": [
        "Microsoft.Authorization/*/write",
        "Microsoft.Network/virtualNetworks/write",
        "Microsoft.Resources/subscriptions/delete"
      ]
    }
  ],
  "assignableScopes": [
    "/subscriptions/{subscriptionId}"
  ]
}
```

### 12.3 Tagging Checklist

**Before resource deployment:**
- ✅ Cost center assigned and validated
- ✅ Owner identified and contacted
- ✅ Environment tag set correctly
- ✅ Workload classification confirmed
- ✅ Compliance requirements documented
- ✅ Backup/DR tier assigned
- ✅ Data classification specified
- ✅ Project code linked (if applicable)

### 12.4 Access Request Template

**Template fields:**
- Requester name and department
- Role requested and justification
- Target subscription/resource group
- Required duration (default 12 months)
- Manager approval
- Security team approval (if elevated privileges)
- Business owner approval (if cross-team access)

---

## 13. Document Control

| Section | Owner | Last Updated | Next Review |
|---|---|---|---|
| Management Groups | Infrastructure Lead | 2026-08-10 | 2026-11-10 |
| Subscription Design | CCoE Lead | 2026-08-10 | 2026-11-10 |
| Resource Groups | Operations Lead | 2026-08-10 | 2026-11-10 |
| Azure Policy | Security Lead | 2026-08-10 | 2026-10-10 |
| RBAC Model | Identity Lead | 2026-08-10 | 2026-11-10 |
| Naming Conventions | Operations Lead | 2026-08-10 | 2026-11-10 |
| Tagging Standards | Finance Lead | 2026-08-10 | 2026-10-10 |
| Governance KPIs | CCoE Lead | 2026-08-10 | 2026-09-10 |

---

**End of Document**
