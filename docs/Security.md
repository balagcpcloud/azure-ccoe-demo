# Azure Security Baseline – Zero Trust Architecture

**Version:** 1.0  
**Last Updated:** 2026-08-10  
**Owner:** Cloud Security Team  
**Status:** Active  
**Framework Alignment:** Zero Trust, NIST Cybersecurity Framework, Microsoft Security Best Practices

---

## Executive Summary

This security baseline establishes enterprise-grade security controls for Azure environments based on Zero Trust principles. Zero Trust assumes that no user, device, or network is inherently trusted. Every access request must be authenticated, authorized, and continuously validated regardless of origin or network location.

**Core Tenets:**
1. **Verify explicitly** - Always authenticate and authorize based on available data
2. **Secure by default** - Apply least privilege and deny by default
3. **Assume breach** - Plan for compromise and minimize blast radius
4. **Microsegmentation** - Isolate workloads and limit lateral movement

---

## 1. Zero Trust Principles in Azure

### 1.1 Zero Trust Model Alignment

```
Traditional Perimeter Model:          Zero Trust Model:
┌─────────────────────────┐          ┌──────────────────────────┐
│   Trusted Internal      │          │  Continuous Verification │
│  ┌────────────────┐     │          │  ┌────────────────┐      │
│  │ Any User/Device       │          │  │ Identity Verified      │
│  │ Full Access           │          │  │ Device Compliant       │
│  └────────────────┘     │          │  │ Network Monitored      │
│   Untrusted External    │          │  │ Risk Assessment        │
└─────────────────────────┘          │  └────────────────┘      │
   Firewall              │          │  Micro-Segmentation   │
                                    │  Continuous Monitoring │
                                    └──────────────────────────┘
```

### 1.2 Zero Trust Across the Stack

| Layer | Zero Trust Control | Implementation |
|-------|-------------------|-----------------|
| **Identity** | Every user verified with MFA | Azure AD + Conditional Access + PIM |
| **Device** | Device compliance enforced | Intune mobile device management |
| **Application** | API authentication required | OAuth 2.0, service principals, managed identities |
| **Network** | Microsegmentation enforced | Azure Firewall, NSGs, Private Endpoints |
| **Data** | Encryption and access control | TLS in transit, encryption at rest, RBAC |

### 1.3 Trust Verification Signals

**Primary Verification:**
- Multi-factor authentication (MFA) - mandatory for all users
- Device compliance status - Intune compliance policies
- Risk assessment - Azure AD Identity Protection
- Network location - IP ranges and geolocation analysis
- Anomaly detection - Behavioral analytics via Azure AD

**Continuous Validation:**
- Real-time risk re-evaluation during sessions
- Conditional Access policy enforcement
- Network traffic inspection
- Resource access logging and monitoring

---

## 2. Identity Security

### 2.1 Azure AD Configuration

#### Global Settings
- **Password Policy:** 14+ characters, complexity required, no dictionary words
- **Session Timeout:** 1 hour for privileged users, 8 hours for standard users
- **Password Expiration:** 90 days for service accounts, 180 days for users
- **Azure AD Connect:** Hybrid identity with password hash sync + SSO
- **Azure AD MFA:** Enforced for all users globally (no exceptions)

#### Self-Service Password Reset (SSPR)
- **Registration Rate Target:** 95%+ of users enrolled
- **Reset Methods:** Email + security questions (minimum 2)
- **Verification:** User must answer 2 questions before reset
- **Notification:** Admin notified of SSPR events
- **Lockout Policy:** Account locked after 5 failed attempts (30-min cooldown)

#### Azure AD Groups Management
| Group Type | Naming Pattern | Membership | Use Case |
|---|---|---|---|
| **Security** | `sec-{workload}-{role}` | Manual/Dynamic | RBAC, Conditional Access |
| **Office 365** | `team-{department}-{function}` | Dynamic rules | Email distribution |
| **Dynamic** | `dyn-{criteria}` | Automatic based on attributes | Automatic policy assignment |

**Example Dynamic Rules:**
```
// Assign MFA requirement to all privileged users
(user.jobTitle -eq "Administrator") OR (user.department -eq "Security")

// Include all users in specific department
(user.department -eq "Engineering") AND (user.country -eq "US")
```

### 2.2 User and Admin Lifecycle

#### User Onboarding (Day 0-1)
1. Create Azure AD user account with email
2. Assign temporary password (force change on first login)
3. Enroll in MFA (Microsoft Authenticator + FIDO2 optional)
4. Assign security groups based on role
5. Grant initial least-privilege RBAC roles
6. Send welcome email with security awareness content

**Assigned Roles:**
- Standard Users: Reader on own subscription
- Developers: Contributor (Resource Group-level only)
- Administrators: Eligible for Owner role (via PIM)

#### User Offboarding (Day Last)
1. **Day 0:** Manager submits offboarding request
2. **Day 1:** Disable account, revoke all MFA registrations
3. **Day 2:** Reset all passwords and access keys
4. **Day 7:** Remove from all security groups and distribution lists
5. **Day 30:** Delete account and archive logs
6. **Day 90:** Purge from Azure AD recycle bin

**Offboarding Verification:**
- Confirm all resources reassigned or deleted
- Verify no active sessions remaining
- Audit all access logs for suspicious activity in past 30 days

#### Contractor/Vendor Access
- Limited to specific resource groups (no subscription-level access)
- Time-bound access (maximum 12 months, auto-expire)
- Require FIDO2 hardware keys (no phone-based MFA)
- Monthly access reviews by resource owner
- Immediate disable upon contract termination

### 2.3 Identity Protection & Risk Management

#### Azure AD Identity Protection

| Risk Level | Detection | Action |
|---|---|---|
| **Low Risk** | Unusual sign-in properties | Audit logged |
| **Medium Risk** | Anonymous IP, leaked credentials | Require MFA + password change |
| **High Risk** | Impossible travel, multiple failures | Block + admin review required |

**Risky User Policies:**
- Reset password immediately upon high-risk detection
- Require sign-in from known device
- Block legacy authentication (NTLM, Pop3, IMAP)
- Force re-authentication after 24 hours of high-risk

**Risky Sign-in Policies:**
- Require MFA for medium-risk sign-ins
- Block high-risk sign-ins automatically
- All risky sign-ins logged to Sentinel
- Executive review of blocked attempts weekly

#### Risk Scoring Model

```
Base Risk = 0%

If (anonymous IP)              +30%
If (unusual location)          +20%
If (unfamiliar sign-in)        +15%
If (leaked credentials)        +40%
If (impossible travel)         +50%
If (atypical travel)           +25%
If (malware detected)          +60%
If (multiple failed attempts)  +35%

Action:
  0-20%   → Log only
  20-40%  → Require MFA
  40-70%  → Require MFA + password change
  70-100% → Block + require admin review
```

#### User Risk Policies
- High-risk users cannot access sensitive applications
- Force password reset every 30 days if flagged
- Re-registration in MFA required quarterly
- Risk remediation tracked and verified
- Users can appeal high-risk blocks via help desk

### 2.4 B2B & B2C Scenarios

#### B2B Partner Access
- **Identity Source:** Partner's Azure AD tenant (federated)
- **MFA:** Enforced by their home tenant
- **Access Duration:** 12 months maximum, annual renewal
- **Scoping:** Specific resource groups only
- **Monitoring:** Sentinel alerts for all partner access
- **Revocation:** 24-hour notice to remove access

**Partner Onboarding Process:**
1. Generate B2B invitation link
2. Partner accepts invitation and registers
3. Assign to security group with limited scope
4. Partner grants consent to sign-in application
5. Document MOU and data handling agreement

#### B2C Customer Access (if applicable)
- **Authentication:** Phone/email verification
- **MFA:** Optional based on application risk
- **Session Duration:** 30 days with no re-auth
- **Token Lifetime:** 1 hour access token, 14 days refresh
- **Social Identity:** Facebook/Google allowed with MFA

---

## 3. Conditional Access

### 3.1 Conditional Access Architecture

Conditional Access evaluates **every login attempt** based on signals and applies **adaptive controls**.

```
User Login Request
    ↓
[Signal Collection]
  • Identity (username, risk)
  • Device (compliant, location)
  • Application (sensitivity)
  • Location (IP, geography)
  • Session (ongoing access)
    ↓
[Policy Evaluation Engine]
  Policy 1: MFA required if...
  Policy 2: Block if...
  Policy 3: Session control if...
    ↓
[Decision]
  → Allow (continue)
  → Allow with condition (MFA, sign-in frequency)
  → Block (deny access)
```

### 3.2 Conditional Access Policies

#### Policy 1: Baseline MFA for All Users
**Name:** CA-001-Require-MFA-All-Users

| Setting | Value | Purpose |
|---|---|---|
| Assignments | All users | Universal coverage |
| Cloud apps | All cloud apps | Full protection |
| Conditions | - | No exceptions |
| Grant | Require MFA | Mandatory authentication |
| Session | Sign-in frequency: 30 days | Re-authentication requirement |

**Exclusions:** Service accounts, break-glass emergency accounts (documented)

#### Policy 2: Require MFA for Admin Roles
**Name:** CA-002-Require-MFA-Admins

| Setting | Value | Purpose |
|---|---|---|
| Assignments | Admin roles (Global Admin, Security Admin) | Privilege escalation prevention |
| Cloud apps | All cloud apps | No admin exceptions |
| Conditions | - | Mandatory |
| Grant | Require MFA + password change | Enhanced security |

**Enforcement:** Immediate, no exceptions or exemptions

#### Policy 3: Block Legacy Authentication
**Name:** CA-003-Block-Legacy-Auth

| Setting | Value | Purpose |
|---|---|---|
| Assignments | All users | Disable weak protocols |
| Cloud apps | All cloud apps | Comprehensive |
| Conditions | Client app = Exchange ActiveSync, IMAP, POP3, SMTP | Block protocols |
| Grant | Block | Deny legacy clients |

**Impact:** Requires modern authentication (OAuth 2.0, SAML 2.0)

#### Policy 4: High-Risk Sign-in Handling
**Name:** CA-004-Block-High-Risk-SignIn

| Setting | Value | Purpose |
|---|---|---|
| Assignments | All users | Identity protection |
| Cloud apps | All cloud apps | Blanket coverage |
| Conditions | Sign-in risk: High | Zero Trust validation |
| Grant | Block access | Deny compromised accounts |
| Session | - | No partial allow |

**Alternative:** Require password change + re-verify MFA (less restrictive)

#### Policy 5: Non-Compliant Device Block
**Name:** CA-005-Require-Device-Compliance

| Setting | Value | Purpose |
|---|---|---|
| Assignments | All users | Device security enforcement |
| Cloud apps | Sensitive apps (Azure Portal, Exchange Online) | Risk-based |
| Conditions | Device state: Non-compliant | Only trusted devices |
| Grant | Block OR Require remediation | Prevent unmanaged device access |

**Compliant Devices:** Intune-enrolled, updated OS, AV enabled

#### Policy 6: Location-Based Access
**Name:** CA-006-Restrict-Risky-Locations

| Setting | Value | Purpose |
|---|---|---|
| Assignments | All users | Geo-restriction |
| Cloud apps | All cloud apps | Universal protection |
| Conditions | Location: Untrusted countries (blocklist) | Compliance/risk mitigation |
| Grant | Block | Deny access from high-risk countries |

**Blocklist:** Countries with high cybercrime activity, sanctioned nations

#### Policy 7: Impossible Travel Prevention
**Name:** CA-007-Impossible-Travel

| Setting | Value | Purpose |
|---|---|---|
| Assignments | All users | Detect credential compromise |
| Cloud apps | All cloud apps | Full coverage |
| Conditions | Risk: Impossible travel detected | Time/distance analysis |
| Grant | Require MFA + password change | Verify legitimate user |

**Logic:** If user signs in from Location A, then Location B within 1 hour (impossible given travel time), trigger policy.

#### Policy 8: Session Control - Persistent Browser
**Name:** CA-008-Persistent-Browser-Session

| Setting | Value | Purpose |
|---|---|---|
| Assignments | Privileged users | Admin account protection |
| Cloud apps | Azure Portal, Azure Management | Control plane |
| Conditions | - | Always apply |
| Grant | Require device to be marked as compliant | Device binding |
| Session | Use Conditional Access app control | Monitor admin activity |

### 3.3 Conditional Access Testing & Deployment

**Testing Phases:**
1. **Report-Only Mode:** Monitor matches without enforcing (30 days)
2. **Pilot Group:** Deploy to test users with exemption override (14 days)
3. **Staged Rollout:** Deploy to departments progressively (2-4 weeks)
4. **General Availability:** Enforce globally with documented exemptions

**Exemption Criteria:**
- Business process requires legacy auth (approval required)
- Device unable to support MFA (documentation required)
- Service account with specific needs (CCoE approval + monitoring)

**Exemption Lifecycle:**
- **Created:** Requires CCoE approval
- **Active:** 90 days maximum
- **Review:** 30-day warning before expiration
- **Expired:** Automatic removal, access blocked
- **Appeal:** Submit new request with updated business justification

---

## 4. Privileged Identity Management (PIM)

### 4.1 PIM Principles

**Core Concept:** Roles are **eligible** (not active) by default. Users must **request activation** with justification, undergoing approval and time-limited elevation.

```
User Role Assignment Lifecycle:

Day 1: Assign Eligible Role
  ↓
User needs role (e.g., day 30)
  ↓
User requests activation with justification
  ↓
Manager approval (if configured)
  ↓
MFA challenge required
  ↓
Role activated for 8 hours (time-limited)
  ↓
Activity logged and monitored
  ↓
Auto-deactivation after 8 hours
  ↓
Access review reminder at 60 days
  ↓
Role assignment renewed or revoked
```

### 4.2 Role Configuration

#### Tier 1: Non-Privileged Users
**Roles:** Reader, Virtual Machine User Contributor

| Setting | Value | Purpose |
|---|---|---|
| Eligible Duration | 12 months | Annual review cycle |
| Active Duration | Permanent | Daily operational use |
| MFA Required | No | Low risk roles |
| Approval Required | No | User self-activation |
| Audit | Logged | Non-sensitive |

#### Tier 2: Privileged Users (Admins, Leads)
**Roles:** Contributor, Virtual Machine Contributor, Network Contributor

| Setting | Value | Purpose |
|---|---|---|
| Eligible Duration | 6 months | Frequent reviews |
| Active Duration | 4 hours | Limited elevation window |
| MFA Required | Yes | Enhanced security |
| Approval Required | Manager + CCoE | Dual approval |
| Audit | Enhanced logging | Track privilege use |

#### Tier 3: Super-Privileged (Subscription Owner, Global Admin)
**Roles:** Owner, Global Administrator, Security Administrator

| Setting | Value | Purpose |
|---|---|---|
| Eligible Duration | 3 months | Quarterly reviews |
| Active Duration | 1 hour | Minimal window |
| MFA Required | Yes + Physical approval | Maximum security |
| Approval Required | Executive + Security | Multi-level approval |
| Audit | Real-time alerts | Immediate notification |
| Session Recording | Enabled | Full audit trail |

**Approval Chain Example:**
- User requests Owner role activation for "Deploy backup system"
- Manager reviews (within 2 hours)
- CCoE Lead reviews (within 2 hours)
- Both approve → MFA challenge → 4-hour activation

### 4.3 PIM Activation Rules

| Scenario | Rule | Enforcement |
|---|---|---|
| **User activates eligible role** | Require MFA | Before activation allowed |
| **Activation without justification** | Block until provided | User must supply business reason |
| **Multiple rejections in week** | Escalate to manager | Alert manager of suspicious pattern |
| **Activation during odd hours** | Require extra approval | After-hours activations reviewed |
| **Same user, multiple roles** | Cross-approval required | Manager reviews all elevation |
| **Activation approaching expiry** | Re-enrollment reminder | 30-day warning email |

### 4.4 Access Reviews

**Review Schedule:**
- **Quarterly:** All privileged roles (Owner, Admin roles)
- **Semi-annually:** Contributor roles
- **Annually:** Reader and non-privileged roles

**Review Process:**
1. **Automated Email:** Owner notified of assigned users
2. **Review Period:** 30 days to review access
3. **Action Items:**
   - ✅ Approve - keep role assigned
   - ❌ Deny - remove role permanently
   - ❓ Unsure - escalate to manager
4. **Follow-up:** Denied users notified, access removed in 7 days
5. **Audit:** Results logged, trends reviewed monthly

**Escalation:** Users with 3+ denied reviews within 12 months → IT security investigation

### 4.5 Break-Glass Emergency Accounts

**Purpose:** Ensure access during Azure AD outages

**Configuration:**
- 2 accounts per organization (redundancy)
- Stored offline (encrypted password in safe)
- Permanent assignment as Global Admin (no PIM)
- MFA disabled (MFA service may be unavailable)
- Cloud-native accounts (no on-premises sync)
- Never used except during emergency
- Access logged and reviewed immediately after use
- Used emergency → password changed within 24 hours

**Activation Protocol:**
1. Confirm Azure AD service unavailable (multiple teams verify)
2. Open safe and retrieve break-glass credential
3. Sign in and restore service
4. Document incident and timeline
5. Change break-glass password immediately
6. Report to CTO and Security Lead
7. Conduct incident review within 48 hours

---

## 5. Defender for Cloud

### 5.1 Defender for Cloud Architecture

```
Azure Subscriptions
    ↓
[Vulnerability Scanning]
  • VMs and containers
  • Application vulnerabilities
  • Dependency scanning
    ↓
[Threat Detection]
  • Anomalous behavior
  • Malware signatures
  • Attack patterns
    ↓
[Posture Management]
  • Configuration compliance
  • Security standards alignment
  • Benchmark scoring
    ↓
[Recommendations Engine]
  • Security hygiene issues
  • Compliance gaps
  • Risk prioritization
    ↓
[Central Dashboard]
  • Secure Score
  • Recommendations
  • Compliance status
```

### 5.2 Defender Plans

| Plan | Coverage | Cost | Use Case | Enablement |
|---|---|---|---|---|
| **Defender for Servers** | VMs and servers | $16.99/month/VM | Linux/Windows protection | Required for production |
| **Defender for Storage** | Blob storage, Data Lake | $0.50/GB/month | Malware scanning | Required for prod data |
| **Defender for SQL** | SQL Database, SQL MI | $14.99/month | Database threat detection | Required for prod DBs |
| **Defender for Key Vault** | Vault access patterns | $0.02/10K operations | Unauthorized access detection | Required for secrets |
| **Defender for Containers** | AKS, Container registries | $6/month/core | Container image scanning | Required for AKS |
| **Defender for App Service** | Web apps, Function apps | $0.58/instance/day | Web threat protection | Required for prod apps |

**Deployment Status:**
- ✅ Mandatory for all Production resources
- ✅ Recommended for Staging resources
- ❌ Optional for Development resources
- ❌ Not required for Sandbox resources

### 5.3 Defender Configuration

#### Threat Detection Settings
```
Auto Provisioning: Enabled
  ├─ Log Analytics agent
  ├─ Vulnerability assessment extension
  ├─ Adaptive application control
  └─ Defender sensors

Email Notifications: Enabled
  ├─ High-severity alerts → Security team
  ├─ Medium alerts → Resource owner
  └─ Low alerts → Logged only

Response Automation: Enabled
  ├─ Trigger Security Orchestration
  ├─ Create incidents in ServiceNow
  └─ Call custom webhooks
```

#### Vulnerability Management
- **Scan Frequency:** Daily for VMs, continuous for container images
- **Baseline:** Compare against NIST guidelines
- **CVSS Scoring:** Critical (9.0+), High (7.0-8.9), Medium (4.0-6.9), Low (<4.0)
- **Remediation SLA:** Critical within 24 hours, High within 7 days
- **Tracking:** All CVEs logged to Azure DevOps for remediation tickets

#### Secure Score
**Scoring Model:**
- Max score: 100 points
- Recommendations weighted by impact and effort
- Updated daily as controls are implemented
- Historical tracking shows progress

**Target Scores by Environment:**
- Production: ≥ 85 (critical controls only)
- Staging: ≥ 75 (important controls)
- Development: ≥ 60 (optional controls)

**Sample Recommendations:**
- Enable MFA on all users (+5 points)
- Enable Defender on all servers (+10 points)
- Restrict network access (+8 points)
- Encrypt data at rest (+6 points)
- Enable logging (+7 points)

#### Custom Policies
**Example:** Require Microsoft Antimalware on all VMs

```
Recommendation Name: "Install Microsoft Antimalware"
Severity: High
Affected Resources: Servers without antimalware
Remediation:
  1. Deploy Microsoft Antimalware extension
  2. Configure to scan daily
  3. Enable real-time protection
  4. Verify detection engine updates enabled
```

### 5.4 Defender Alerts & Response

| Alert Type | Example | Severity | Response Time |
|---|---|---|---|
| **Suspicious Process** | "Mimikatz detected on production VM" | Critical | 15 min |
| **Malware Detection** | "Trojan.Win32 quarantined" | Critical | 15 min |
| **Brute Force Attack** | "50 failed RDP attempts from China" | High | 1 hour |
| **Unusual Activity** | "Database queried at 3 AM from unusual IP" | High | 2 hours |
| **Compliance Issue** | "Unencrypted database discovered" | Medium | 24 hours |
| **Configuration Drift** | "NSG rule deleted unexpectedly" | Medium | 24 hours |

**Alert Actions:**
1. **Auto Remediation:** Security controls automatically applied (with approval for sensitive changes)
2. **Alert Forwarding:** Critical alerts pushed to Sentinel, ServiceNow, and security team
3. **Investigation:** Defender provides attack timeline and impacted resources
4. **Containment:** Run playbooks to isolate compromised resources
5. **Eradication:** Remove malware and patch vulnerabilities
6. **Verification:** Rescan to confirm threat eliminated

---

## 6. Microsoft Sentinel

### 6.1 Sentinel Architecture

```
Data Sources
  ├─ Azure Activity Logs
  ├─ Azure AD Audit Logs
  ├─ Defender for Cloud alerts
  ├─ Firewall logs (Azure Firewall)
  ├─ Network logs (NSG flow logs)
  ├─ Application logs (App Insights)
  ├─ Third-party SIEM (Syslog)
  └─ Custom applications
    ↓
[Data Ingestion & Storage]
  Log Analytics Workspace
  Retention: 90 days hot, 365 days cool
    ↓
[Analysis & Detection]
  Analytics Rules (custom queries)
  Microsoft threat intelligence
  UEBA (User and Entity Behavior Analytics)
    ↓
[Response & Investigation]
  Automated playbooks
  Incident creation
  Investigation workflow
    ↓
[Notifications]
  Teams alerts
  Email notifications
  SOAR integration (if configured)
```

### 6.2 Data Sources & Connectors

| Data Source | Connector | Logs/Events | Retention |
|---|---|---|---|
| **Azure Activity** | Azure Activity | Subscription operations | 90 days |
| **Azure AD** | Azure AD audit, sign-in logs | User authentication, admin changes | 90 days |
| **Defender for Cloud** | Cloud security connector | Security alerts, vulnerabilities | 90 days |
| **Azure Firewall** | Azure Firewall | Network flows, blocked traffic | 30 days |
| **NSG Flow Logs** | NSG Flow Logs | Network traffic patterns | 30 days |
| **Application Insights** | App Insights | Application errors, performance | 90 days |
| **Microsoft 365** | Office 365 Management API | Email, SharePoint, Teams logs | 90 days |
| **Third-Party SIEM** | Syslog/CEF | On-premises systems | 365 days (archive) |

**Storage Strategy:**
- **Hot Storage:** 90 days (searchable, full cost)
- **Cool Storage:** 91-365 days (less frequently accessed)
- **Archive:** 365+ days (compliance archival, minimal cost)

### 6.3 Analytics Rules (Detection)

#### Rule 1: Brute Force Attack Detection
**Name:** Multiple Failed Login Attempts

```kusto
SigninLogs
| where ResultType == "50126"  // Invalid password
| summarize FailureCount = count() by UserPrincipalName, IPAddress
| where FailureCount > 5 within 1h
| join (SigninLogs | where ResultType == "0") on UserPrincipalName, IPAddress
| project UserPrincipalName, IPAddress, FailureCount, SuccessfulSignin=TimeGenerated
```

**Trigger:** 5+ failed logins followed by success within 1 hour  
**Severity:** High  
**Response:** Block account, trigger MFA re-enrollment

#### Rule 2: Privilege Escalation Detection
**Name:** Unexpected Role Assignment

```kusto
AuditLogs
| where OperationName == "Add role assignment"
| where Properties.targetResources[0].displayName != "built-in"
| join (AuditLogs | where OperationName == "Activate role") on InitiatedBy
| project TimeGenerated, InitiatedBy, Role=Properties.targetResources[0].displayName
```

**Trigger:** Unexpected role assignment to non-admin user  
**Severity:** Critical  
**Response:** Immediate audit, block if unauthorized

#### Rule 3: Data Exfiltration Detection
**Name:** Large Data Transfer to External Location

```kusto
AzureDiagnostics
| where ResourceType == "STORAGEACCOUNTS"
| where OperationName == "GetBlob" or OperationName == "PutBlob"
| summarize DataTransferred = sum(responseSize_d) by ClientIP
| where DataTransferred > 10GB within 1h
| where ClientIP !in (whitelist_ips)
```

**Trigger:** >10GB data transfer from storage in 1 hour  
**Severity:** Critical  
**Response:** Isolate account, block storage access, alert SOC

#### Rule 4: Anomalous Azure Portal Activity
**Name:** Administrative Actions During Off-Hours

```kusto
AuditLogs
| where TimeGenerated between (datetime(2024-08-10T20:00:00Z) and datetime(2024-08-11T06:00:00Z))
| where OperationName in ("Create role assignment", "Delete resource", "Modify policy")
| where Properties.initiatedBy[0].userPrincipalName !in (on_call_admins)
```

**Trigger:** Admin operations between 8 PM - 6 AM (outside business hours)  
**Severity:** Medium  
**Response:** Alert on-call manager, verify authorization

#### Rule 5: Impossible Travel Detection
**Name:** Sign-in from Geographically Impossible Locations

```kusto
SigninLogs
| where ResultType == "0"  // Successful
| project UserPrincipalName, IPAddress, Country=tostring(LocationDetails.countryOrRegion), TimeGenerated
| partition by UserPrincipalName
(
  window w1 = range TimeFrom = ago(1h) to TimeNow by 1h
  (
    join kind=inner SigninLogs on UserPrincipalName
    | where TimeGenerated > TimeFrom
    | summarize MaxDistance = max_distance(Country)
    | where MaxDistance > 1000  // km between countries
  )
)
```

**Trigger:** Sign-in from 2 countries >1000 km apart in < 1 hour  
**Severity:** High  
**Response:** Block user, force MFA re-authentication

### 6.4 Sentinel Playbooks (Automated Response)

#### Playbook 1: Respond to High-Risk Sign-In

```
Trigger: Alert - High-risk sign-in detected
  ↓
Action 1: Block user account
  ↓
Action 2: Force password reset
  ↓
Action 3: Send notification to manager
  ↓
Action 4: Create incident in Sentinel
  ↓
Action 5: Add to watch list for 7 days
  ↓
Outcome: User blocked until security team verifies
```

#### Playbook 2: Quarantine Compromised VM

```
Trigger: Alert - Malware detected on VM
  ↓
Action 1: Isolate VM network (remove from app subnet)
  ↓
Action 2: Move to isolation resource group
  ↓
Action 3: Enable network monitoring
  ↓
Action 4: Disable RDP access
  ↓
Action 5: Create ticket for incident response team
  ↓
Action 6: Notify resource owner
  ↓
Outcome: VM isolated, contained for investigation
```

#### Playbook 3: Disable Compromised Storage Account

```
Trigger: Alert - Suspicious data access pattern
  ↓
Action 1: Rotate storage account keys
  ↓
Action 2: Enable SAS token signing exclusively
  ↓
Action 3: Restrict IP access to known locations
  ↓
Action 4: Enable additional audit logging
  ↓
Action 5: Create incident for investigation
  ↓
Outcome: Storage account hardened against exploitation
```

### 6.5 Incident Management

**Incident Severity Levels:**
- **Critical:** Breach confirmed, data compromised, multi-systems affected → Response within 15 min
- **High:** Active attack, unauthorized access, critical asset threatened → Response within 1 hour
- **Medium:** Security issue detected, control weakness, suspicious activity → Response within 4 hours
- **Low:** Minor anomaly, security hygiene issue, process deviation → Response within 1 business day

**Investigation Workflow:**
1. **Triage:** Validate alert, check false positive rate, determine scope
2. **Containment:** Isolate affected systems, prevent spread
3. **Investigation:** Collect logs, timeline analysis, root cause identification
4. **Eradication:** Remove threat, patch vulnerabilities, restore systems
5. **Recovery:** Validate system integrity, restore from backups if needed
6. **Communication:** Notify stakeholders, document findings
7. **Post-Incident:** Lessons learned, control improvements, training

---

## 7. Azure Key Vault

### 7.1 Key Vault Principles

**Core Concept:** Centralized, encrypted storage for secrets, keys, and certificates with audit logging.

```
Application or User
    ↓
[Access Request with Identity]
  • Verify Azure AD identity
  • Check RBAC permissions
  • Evaluate network location
    ↓
[Audit Decision]
  • Grant access if authorized
  • Deny and log if unauthorized
    ↓
[Retrieve Secret/Key]
  • Return decrypted value
  • Log retrieval event
    ↓
[Return to Caller]
  • Application uses value
  • Value never stored locally
```

### 7.2 Vault Configuration

#### Authentication & Network

| Setting | Configuration | Purpose |
|---|---|---|
| **Authentication** | Azure AD mandatory | Only Azure identities allowed |
| **Network** | Private endpoint only | No public internet access |
| **Firewall** | Enabled, whitelist trusted IPs | Defense in depth |
| **Public Access** | Disabled | Deny all by default |

**Network Diagram:**
```
Azure Virtual Network
    ↓
Private Endpoint (subnet isolated)
    ↓
Key Vault (behind private link)
    ↓
Access Log Analytics + Sentinel
```

#### Access Policies

| Identity | Permissions | Keys | Secrets | Certificates |
|---|---|---|---|---|
| **Managed Identity** (App Service) | Get | - | ✅ | ✅ |
| **Managed Identity** (Function) | Get | ✅ | ✅ | ✅ |
| **Managed Identity** (AKS) | Get | ✅ | ✅ | - |
| **Service Principal** (CI/CD) | Get, List | - | ✅ | ✅ |
| **User** (Administrator) | All | ✅ | ✅ | ✅ |
| **User** (Developer) | Get, List | - | ✅ | - |

**Least Privilege Rule:**
- Service accounts: GET only (minimal permissions)
- Admins: All operations (full access for emergencies)
- Users: Read-only by default (request escalation for changes)

### 7.3 Secret Management

#### Secret Types & Lifecycle

| Secret Type | Example | Rotation | Storage | Rotation Trigger |
|---|---|---|---|---|
| **Database Passwords** | SQL DB connection string | 90 days | Key Vault | Automated via Logic App |
| **API Keys** | Third-party API credentials | 120 days | Key Vault | Quarterly renewal |
| **Storage Keys** | Storage account access keys | 180 days | Key Vault | Annual rotation |
| **Service Principal Secrets** | Azure AD app password | 12 months | Key Vault | Expiration alert |
| **SSH Keys** | VM authentication keys | Never (or with new VM) | Key Vault | On infrastructure update |

**Automated Rotation Process:**
```
Day 1 (Rotation Day):
  1. Generate new secret version
  2. Test with new value
  3. Update dependent services
  4. Mark old version as deprecated
  
Day 2-7:
  5. Monitor for failures
  6. Rollback if issues detected
  
Day 8:
  7. Delete old secret version
  8. Verify no services using old version
```

#### Secret Naming Convention
**Format:** `{app}-{secret-type}-{environment}`

**Examples:**
- `erp-db-password-prod` (production database)
- `crm-api-key-staging` (staging API key)
- `analytics-storage-key-dev` (development storage)

#### Retrieval Methods

**Option 1: Managed Identity (Recommended)**
```csharp
// Application code automatically authenticated via managed identity
var credential = new DefaultAzureCredential();
var client = new SecretClient(vaultUri, credential);
KeyVaultSecret secret = client.GetSecret("app-db-password-prod");
```

**Option 2: Service Principal (Service-to-Service)**
```csharp
var credential = new ClientSecretCredential(tenantId, clientId, clientSecret);
var client = new SecretClient(vaultUri, credential);
KeyVaultSecret secret = client.GetSecret("app-api-key-prod");
```

**Option 3: User Access (Troubleshooting)**
```powershell
// Azure CLI for authorized users
az keyvault secret show --vault-name prod-app-kv --name app-db-password-prod
```

### 7.4 Certificate Management

#### SSL/TLS Certificate Lifecycle

| Stage | Duration | Action | Owner |
|---|---|---|---|
| **Created** | Day 1 | Import self-signed or CA certificate | Security team |
| **Active** | 360 days | Use certificate for HTTPS/TLS | Application |
| **Warning (30 days)** | Day 331 | Auto-notification to owner | System alert |
| **Renewal (7 days)** | Day 354 | Request new certificate | Security team |
| **Deployment** | Day 361 | Deploy new certificate to resources | DevOps |
| **Archived** | Day 365 | Move expired cert to archive | System |

**Certificate Auto-Renewal:**
```
If (certificate.ExpiryDate - Today) < 30 days:
  1. Create renewal request
  2. Request signature from CA
  3. Update certificate in Key Vault
  4. Trigger webhook to update App Gateway / Front Door
  5. Log renewal event
  6. Alert certificate owner
```

#### Certificate Monitoring

```kusto
// KQL query to monitor certificate expiration
AzureDiagnostics
| where ResourceType == "VAULTS"
| where OperationName == "CertificateExpiring"
| project TimeGenerated, Certificate=tostring(properties_name_s), DaysToExpiry=todouble(properties_days_to_expiry_d)
| where DaysToExpiry < 30
| summarize by Certificate, DaysToExpiry
| sort by DaysToExpiry asc
```

### 7.5 Audit & Logging

**Audit Trail Captures:**
- Who accessed the secret (user/app identity)
- When access occurred (timestamp)
- What operation performed (Get, List, Create, Delete)
- From which IP address
- Success or failure status
- Approval/authorization details

**Audit Storage:**
- Real-time: Azure Monitor (90 days)
- Long-term: Log Analytics (365 days)
- Archive: Storage account (immutable 7 years)

**Alert Rules:**
- ⚠️ Unauthorized access attempt → Alert security team immediately
- ⚠️ Secret delete operation → Require manager approval
- ⚠️ Mass secret retrieval → Possible exfiltration, block access
- ⚠️ After-hours access → Flag for review

---

## 8. Network Security

### 8.1 Network Segmentation (Microsegmentation)

**Zero Trust Network Principle:** No implicit trust; every resource isolated until proven authorized.

```
                    Azure Firewall (Central Inspection Point)
                              ↓
    ┌─────────────────────────┼──────────────────────────┐
    ↓                         ↓                          ↓
Front-End Subnet      API/Middleware Subnet       Database Subnet
(App Gateway,         (Functions, APIs)           (SQL DB, Redis)
Web Servers)
    ├─ NSG: Allow 443 in      ├─ NSG: Allow 80/443    ├─ NSG: Allow 1433
    ├─ NSG: Deny all else     ├─ NSG: Deny direct DB  ├─ NSG: Deny public
    └─ Private Subnet         └─ Private subnet       └─ Private subnet
```

### 8.2 Network Security Groups (NSG)

#### Frontend Subnet NSG Rules

| Priority | Name | Source | Destination | Port | Action | Purpose |
|---|---|---|---|---|---|---|
| 100 | AllowHttpsIn | Internet | VNet | 443 | Allow | Public HTTPS access |
| 110 | AllowHttpRedirect | Internet | VNet | 80 | Allow | HTTP → HTTPS redirect |
| 200 | AllowFromFWtoFE | AzureFirewall | Subnet | Any | Allow | Traffic from firewall |
| 300 | DenyAll | * | * | * | Deny | Default deny rule |

#### API Subnet NSG Rules

| Priority | Name | Source | Destination | Port | Action | Purpose |
|---|---|---|---|---|---|---|
| 100 | AllowFromFEtoAPI | FrontEnd-Subnet | Subnet | 80,443 | Allow | Frontend to API |
| 110 | AllowFromFWtoAPI | AzureFirewall | Subnet | Any | Allow | Firewall traffic |
| 200 | AllowToDBSubnet | Subnet | Database-Subnet | 1433 | Allow | API to database |
| 300 | DenyDirect | Internet | Subnet | * | Deny | No direct internet |
| 400 | DenyAll | * | * | * | Deny | Default deny rule |

#### Database Subnet NSG Rules

| Priority | Name | Source | Destination | Port | Action | Purpose |
|---|---|---|---|---|---|---|
| 100 | AllowFromAPItoSQL | API-Subnet | Subnet | 1433 | Allow | API tier access |
| 200 | AllowMonitoringIn | AzureMonitor | Subnet | 443 | Allow | Metrics/logs |
| 300 | DenyAll | * | * | * | Deny | Deny all else |

### 8.3 Azure Firewall

**Role:** Centralized inspection point for all east-west and north-south traffic.

#### Firewall Rules - Application Layer

| Priority | Name | Source | Destination | Protocol | Action | Purpose |
|---|---|---|---|---|---|---|
| 100 | AllowOutbound-MSUpdates | VNet | *.update.microsoft.com | HTTPS | Allow | Windows update |
| 110 | AllowOutbound-NTP | VNet | time.nist.gov | UDP/123 | Allow | Time sync |
| 120 | AllowOutbound-DNS | VNet | 8.8.8.8 | UDP/53 | Allow | External DNS |
| 200 | DenyOutbound-High-Risk | VNet | High-risk domains | * | Deny | Malware prevention |
| 300 | DenyAll | * | * | * | Deny | Default deny |

#### Firewall Network Rules

| Priority | Name | Source | Dest Port | Protocol | Action | Purpose |
|---|---|---|---|---|---|---|
| 100 | AllowVNettoVNet | VNet | 443 | TCP | Allow | Spoke-to-spoke |
| 110 | AllowOutbound-Azure | VNet | Azure public | 443 | TCP | Azure services |
| 200 | AllowMonitoring | Monitoring | VNet | 443 | TCP | Log forwarding |
| 300 | DenyAll | * | * | * | Deny | Default deny |

#### Firewall Threat Intelligence
- **Mode:** Alert + Deny (not just alert)
- **Data Sources:** Microsoft threat intel, OSINT feeds
- **Categories Blocked:**
  - Malware hosting sites
  - Command & control servers
  - Botnet nodes
  - Ransomware distribution

### 8.4 SASE (Secure Access Service Edge)

**Definition:** SASE converges network and security services into a cloud-native platform, replacing traditional enterprise perimeters with identity-based, zero-trust access.

**SASE Architecture:**
```
                         ☁️ SASE Cloud Platform
        ┌──────────────────────────────────────────────────┐
        │                                                    │
        │  ┌─────────────┐  ┌─────────────┐  ┌──────────┐  │
        │  │ Secure Web  │  │    ZTNA     │  │   FWaaS  │  │
        │  │  Gateway    │  │  (SDP)      │  │ (Next-gen)   │
        │  │  (Cloud DLP)│  │  (Access)   │  │          │  │
        │  └─────────────┘  └─────────────┘  └──────────┘  │
        │                                                    │
        │  ┌─────────────┐  ┌─────────────┐  ┌──────────┐  │
        │  │   Threat    │  │ Cloud Email │  │  CASB    │  │
        │  │ Prevention  │  │ Security    │  │ (SaaS    │  │
        │  │ (Sandbox)   │  │ (Anti-spam) │  │  Mgmt)   │  │
        │  └─────────────┘  └─────────────┘  └──────────┘  │
        └──────────────────────────────────────────────────┘
                                ↑
        ┌───────────────────────┼───────────────────────┐
        ↓                       ↓                       ↓
    Remote Users         Office Locations         Data Centers
    (SASE Client)        (SASE Gateway)           (Private LAN)
        
        All Traffic Flow → SASE → Azure Resources (Zero Trust)
```

#### SASE Components in Azure

| Component | Purpose | Azure Equivalent/Integration |
|---|---|---|
| **Firewall as a Service (FWaaS)** | Network traffic inspection, policy enforcement | Azure Firewall + Defender for Cloud |
| **Secure Web Gateway (SWG)** | HTTP/HTTPS traffic filtering, DLP | Proxy to Azure Firewall + Azure DLP |
| **Zero Trust Network Access (ZTNA/SDP)** | Identity-based access control | Conditional Access + Private Endpoints |
| **Cloud Access Security Broker (CASB)** | SaaS application visibility and control | Defender for Cloud Apps |
| **Threat Prevention** | Antimalware, sandboxing, detection | Defender for Endpoint + Sentinel |
| **Cloud Email Security** | Email filtering, phishing protection | Defender for Office 365 |

#### SASE Implementation Strategy

**Phase 1: Identity & Access (Weeks 1-4)**
- Deploy Conditional Access policies (MFA, device compliance)
- Implement Private Endpoints for all Azure services
- Configure Azure AD authentication for all resources
- **Result:** Users authenticate with identity, not network location

**Phase 2: Network Inspection (Weeks 4-8)**
- Deploy SASE client on all endpoints (laptops, mobiles, IoT)
- Route all traffic through SASE cloud platform
- Establish Azure Firewall filtering rules
- **Result:** All traffic inspected, no blind spots

**Phase 3: App-Level Control (Weeks 8-12)**
- Deploy Defender for Cloud Apps for SaaS monitoring
- Implement cloud app policies (data exfiltration prevention)
- Enable session controls for sensitive operations
- **Result:** Visibility and control over all SaaS usage

**Phase 4: Continuous Verification (Weeks 12+)**
- Monitor device posture (antivirus, patch status)
- Revoke access for non-compliant devices
- Adjust policies based on risk signals
- **Result:** Adaptive, continuous trust verification

#### SASE Benefits vs. Traditional Perimeter

| Aspect | Traditional Perimeter | SASE |
|---|---|---|
| **Trust Model** | Trust network (dangerous) | Zero Trust (identity-based) |
| **Remote Access** | VPN (slow, inflexible) | Direct cloud access (fast) |
| **Policy Scope** | Network-based | Identity + device + behavior + location |
| **Latency** | High (backhaul through HQ) | Low (anycast, local exit) |
| **Scalability** | Limited (VPN concentrators) | Unlimited (cloud-native) |
| **Setup Time** | Weeks (hardware config) | Days (cloud provisioning) |
| **Cost Model** | CapEx (hardware) | OpEx (cloud subscription) |
| **Geographic Reach** | Limited to office locations | Global cloud edge locations |

#### SASE Traffic Flow Example

```
Remote User Accessing Azure SQL Database:

1. User initiates connection
   └─ SASE client intercepts (endpoint security)

2. Authentication
   └─ Azure AD validates identity + MFA

3. Device Compliance Check
   └─ Is device patched?
   └─ Is antivirus enabled?
   └─ Is user on approved network?

4. Risk Assessment
   └─ Impossible travel detected? Block
   └─ Anomalous location? Step-up auth
   └─ High-risk user profile? Conditional access

5. Traffic Routing
   └─ Route through SASE firewall (closest egress point)
   └─ Apply DLP policies (prevent data exfiltration)
   └─ Inspect for threats (malware, exploits)

6. Private Endpoint Access
   └─ Direct connection to SQL via private endpoint
   └─ No exposure through public internet

7. Session Logging
   └─ All actions logged to Sentinel
   └─ User activity tracked for compliance
   └─ Anomalies trigger alerts
```

#### SASE Client Deployment

**Configuration on Endpoints:**
```
SASE Client Settings:

├─ Traffic Forwarding
│  ├─ Mode: Split tunneling (corporate traffic to SASE, personal to internet)
│  ├─ Exceptions: Allowed domains for direct access
│  └─ Fallback: Local proxy if SASE unavailable

├─ Device Posture Checks
│  ├─ Antivirus: Must be enabled and current
│  ├─ Firewall: Windows Defender Firewall required
│  ├─ Encryption: BitLocker enabled on Windows
│  ├─ Patch Level: KB updates within 30 days
│  └─ Disk Encryption: FileVault on macOS

├─ Data Loss Prevention (DLP)
│  ├─ Prevent copy/paste of sensitive data
│  ├─ Block file uploads to unauthorized apps
│  ├─ Monitor clipboard for PII patterns
│  └─ Log all DLP violations

├─ Authentication
│  ├─ Continuous re-authentication (every 8 hours)
│  ├─ MFA required for sensitive resources
│  ├─ Device certificate installed
│  └─ Single sign-on (SSO) integration
```

---

### 8.5 Private Endpoints

**Purpose:** Enable access to Azure PaaS services without public IP exposure.

```
Application in VNet
    ↓
Private Endpoint (IP address in subnet)
    ↓
Private Link (AWS PrivateLink equivalent)
    ↓
Azure Service (SQL DB, Storage, Key Vault)
    ↓
No traversal through public internet
```

**Services with Private Endpoints:**
- ✅ Azure Storage (Blob, Queue, File, Table)
- ✅ Azure SQL Database
- ✅ Azure Database for PostgreSQL/MySQL
- ✅ Azure Cosmos DB
- ✅ Azure Key Vault
- ✅ Azure App Service
- ✅ Azure Container Registry
- ✅ Azure Service Bus
- ✅ Azure Event Hubs

**Configuration:**
```
Resource Group: prod-app-network
  └─ Virtual Network: prod-app-vnet
      └─ Subnet: prod-app-private-snet
          └─ Private Endpoint: prod-app-sqldb-pe
              └─ Links to: prod-app-sqldb (private)
                  └─ NIC IP: 10.1.2.10 (internal only)
```

### 8.6 DDoS Protection

#### Azure DDoS Protection Standard

| Feature | Configuration | Purpose |
|---|---|---|
| **Layer 3/4 Protection** | Enabled | Network-level DDoS mitigation |
| **Layer 7 Protection** | Enabled (WAF integration) | Application-level attack mitigation |
| **Auto-mitigation** | Enabled | Automatic attack detection and response |
| **Alerts** | Configured | Alert on DDoS detection attempts |

**Traffic Flow with DDoS Protection:**
```
Inbound Traffic
    ↓
[DDoS Detection Engine]
  Volumetric attacks (floods)
  Protocol attacks (fragmentation)
  Resource attacks (DNS amplification)
    ↓
[Rate Limiting & Filtering]
  Drop malicious traffic
  Allow legitimate traffic
    ↓
[Healthy Traffic Forwarded]
```

### 8.7 WAF (Web Application Firewall)

**Deployment:** Azure WAF integrated with Application Gateway and Front Door.

#### WAF Rule Groups

| Group | Rules | Purpose | Action |
|---|---|---|---|
| **OWASP Core** | SQLi, XSS, RFI, LFI, etc. | Common web attacks | Block |
| **Known CVE** | ColdFusion, Joomla, WordPress | Known vulnerability patterns | Block |
| **Geo-filtering** | Country-based rules | Restrict by geography | Block |
| **Rate Limiting** | Requests per IP/session | Prevent brute force | Throttle |
| **Bot Protection** | Bot signatures | Prevent automated attacks | Block/Challenge |

**WAF Rules Example (Custom):**
```
Rule Name: Block SQL Injection Attempts
Condition: 
  - URL Parameter "id" contains: ' OR '1'='1
  - OR URL Parameter "id" contains: UNION SELECT
  - OR URL Parameter "id" contains: DROP TABLE
Action: Block (403 Forbidden)
Logging: Alert security team
```

---

## 9. Incident Response

### 9.1 Incident Response Framework

```
1. DETECT          2. ANALYZE        3. CONTAIN        4. ERADICATE      5. RECOVER
   ↓                  ↓                 ↓                 ↓                 ↓
Alert triggered   Severity          Isolate           Remove            Restore
Investigation     Assessment        affected          threat            services
begins                               systems                             to normal
```

### 9.2 Incident Classification

| Level | Description | Examples | Response Time | Escalation |
|---|---|---|---|---|
| **CRITICAL** | Breach confirmed, data compromised | Active exploit, ransomware | 15 minutes | CEO, Legal, PR |
| **HIGH** | Unauthorized access, integrity threat | Privilege escalation, malware | 1 hour | CISO, CTO |
| **MEDIUM** | Security control failure, anomaly | Config drift, unauthorized login attempt | 4 hours | Security Lead |
| **LOW** | Security hygiene issue, minor risk | Weak password, outdated patch | 1 business day | Resource Owner |

### 9.3 Incident Response Procedures

#### Critical Incident: Suspected Data Breach

**Timeline:**
```
T+00 min: Alert received (Sentinel, Defender)
  └─ Automatically create incident
  └─ Page on-call security team
  └─ Start incident conference call

T+05 min: Initial triage
  └─ Validate alert (false positive check)
  └─ Determine scope (systems affected)
  └─ Assess data exposure potential

T+15 min: Containment
  └─ Isolate affected systems (disconnect from network)
  └─ Block suspected attacker IPs
  └─ Revoke access credentials
  └─ Enable enhanced logging

T+1 hour: Investigation
  └─ Collect forensic evidence (memory dumps, disk images)
  └─ Analyze logs for attack timeline
  └─ Identify data accessed
  └─ Determine if data exfiltrated

T+4 hours: Notification decision
  └─ Determine if data breach notification required (legal, regulatory)
  └─ Contact affected customers if applicable
  └─ Notify regulators (GDPR, HIPAA, etc.)

T+24 hours: Eradication
  └─ Patch vulnerabilities exploited
  └─ Update firewall rules
  └─ Reset all credentials
  └─ Remove backdoors/malware

T+48 hours: Recovery
  └─ Restore systems from clean backups
  └─ Verify system integrity
  └─ Resume normal operations
  └─ Re-enable monitoring

T+1 week: Post-incident
  └─ Complete forensic analysis
  └─ Identify root cause
  └─ Implement corrective actions
  └─ Document lessons learned
  └─ Update incident response procedures
```

#### High Incident: Unauthorized Administrative Access

**Suspected Compromise of Admin Account**

**Immediate Actions (T+0-15 min):**
1. Disable compromised admin account
2. Force password reset for all admin users
3. Revoke all active sessions
4. Enable audit logging on all subscriptions
5. Review recent admin activities for unauthorized changes

**Investigation (T+15-60 min):**
1. Collect Azure AD sign-in logs
2. Review administrative actions in activity logs
3. Check for privileged role assignments added
4. Audit Key Vault accesses
5. Review API calls and resource modifications

**Containment (T+1-4 hours):**
1. Revert any unauthorized resource changes
2. Reset service account credentials
3. Review and restrict network access
4. Add additional MFA requirements
5. Enable conditional access strictness

**Recovery (T+4-24 hours):**
1. Restore modified resources from backups if needed
2. Verify data integrity
3. Re-enable normal access controls
4. Conduct access review for all privileged users
5. Monitor for any re-compromise attempts

### 9.4 Incident Response Team Roles

| Role | Responsibility | Skills | Escalation Path |
|---|---|---|---|
| **Incident Commander** | Overall coordination, decision making | Leadership, communication | Reports to CISO |
| **Forensics Analyst** | Evidence collection, timeline analysis | Logs, memory dumps, disk analysis | Reports to Incident Commander |
| **Threat Hunter** | Attack pattern analysis, adversary behavior | Threat intel, malware analysis | Reports to Incident Commander |
| **Remediation Engineer** | System restoration, patching, configuration | Azure, Windows/Linux admin | Reports to Incident Commander |
| **Communications Lead** | Stakeholder updates, notifications | Legal, PR, customer communication | Reports to CISO/Legal |

### 9.5 Incident Response Playbooks

**Playbook 1: Ransomware Attack Response**
```
Detection: Sentinel alert "mass file encryption detected"
Activation Level: CRITICAL
Timeline: Response within 15 minutes
Steps:
  1. Isolate affected server from network (kill switch)
  2. Disable backups to prevent ransom encryption
  3. Preserve evidence (don't shutdown, preserve memory)
  4. Enable full disk monitoring
  5. Identify patient zero (first compromised system)
  6. Trace infection path through network
  7. Restore from last known clean backup
  8. Verify no persistence mechanism remains
Escalation: Immediate notification to legal, PR, customers
Post-incident: Review backup strategy, segmentation, endpoint protection
```

**Playbook 2: Credential Compromise Response**
```
Detection: Multiple failed logins from impossible location
Activation Level: HIGH
Timeline: Response within 1 hour
Steps:
  1. Disable user account immediately
  2. Force password reset (multi-step verification)
  3. Revoke all active sessions and tokens
  4. Review all resources accessed in past 24 hours
  5. Check for privilege escalation attempts
  6. Audit access to sensitive data
  7. Enable enhanced logging for future access
  8. Require hardware token MFA
Post-incident: Access review, security awareness training
```

**Playbook 3: DDoS Attack Response**
```
Detection: Azure DDoS Protection detected attack
Activation Level: MEDIUM (depends on impact)
Timeline: Auto-mitigation within seconds, manual review within 1 hour
Steps:
  1. Verify attack detection is accurate (check legitimate traffic)
  2. Enable rate limiting if not automatic
  3. Route traffic through WAF if applicable
  4. Review attack patterns (source IPs, traffic type)
  5. Coordinate with ISP if external DDoS
  6. Document attack duration and impact
  7. Analyze logs for any exploitation attempts
  8. Confirm service restored
Post-incident: Review DDoS protection settings, update WAF rules
```

### 9.6 Incident Documentation

**Required Documentation:**
- **Incident Report:** Summary, timeline, impact, root cause
- **Forensic Report:** Technical analysis, evidence chain of custody
- **Lessons Learned:** What worked, what to improve
- **Action Items:** Preventive measures, training, process updates
- **Timeline Log:** Precise timestamps of all actions taken

**Distribution:**
- Executive summary to leadership/board
- Full report to security team
- Relevant portions to affected teams
- Archive for regulatory/compliance requirements

---

## 10. Security KPIs & Metrics

| KPI | Target | Frequency | Owner | Alert Threshold |
|---|---|---|---|---|
| **Secure Score** | ≥ 85 (prod), ≥ 75 (staging) | Weekly | Security Lead | < 75 |
| **Compliance Rate** | ≥ 99% | Weekly | Compliance Officer | < 95% |
| **MFA Enrollment** | 100% for privileged users | Monthly | Identity Lead | < 98% |
| **Incident Response Time** | Critical: <30 min, High: <2 hours | Per incident | Incident Commander | Any slower |
| **Vulnerabilities Remediated** | Critical: 24 hrs, High: 7 days | Weekly | Infrastructure Lead | Any overdue |
| **Access Reviews Completed** | 100% within 90 days | Quarterly | Security Lead | < 90% |
| **Unpatched Systems** | < 1% of fleet | Weekly | Patch Manager | > 2% |
| **Failed Login Attempts Blocked** | > 99.9% | Weekly | Identity Lead | < 99% |

---

## 11. Security Compliance & Audit

### 11.1 Compliance Frameworks

| Framework | Scope | Frequency | Owner | Auditor |
|---|---|---|---|---|
| **CIS Benchmark** | Azure configuration | Monthly | CCoE | Automated |
| **Azure Security Benchmark** | Security controls | Quarterly | Security Lead | Internal audit |
| **NIST Cybersecurity Framework** | Overall posture | Annually | CISO | External auditor |
| **ISO 27001** | Information security | Annually | Compliance Officer | Certification body |

### 11.2 Audit Trail Requirements

**All Security Controls Must Log:**
- ✅ Identity: Who, when, from where, what role
- ✅ Access: What resource, what action, success/failure
- ✅ Changes: What changed, who changed it, approval trail
- ✅ Anomalies: Detected threats, investigation results
- ✅ Compliance: Policy adherence, exceptions, reviews

**Retention:** 90 days hot (searchable), 7 years cold (archive)

---

## 12. Security Training & Awareness

### 12.1 Required Training

| Course | Audience | Frequency | Duration | Topics |
|---|---|---|---|---|
| **Zero Trust Fundamentals** | All employees | Annual | 30 min | Core principles, architecture |
| **Azure Security Essentials** | Developers, Ops | Annual | 60 min | MFA, Key Vault, Defender |
| **Incident Response** | IT/Security staff | Annual | 90 min | Incident classification, playbooks |
| **Phishing & Social Engineering** | All employees | Quarterly | 15 min | Threat recognition, reporting |
| **Data Protection** | All employees | Annual | 45 min | Classification, handling, encryption |

### 12.2 Security Culture

**Guiding Principles:**
- Security is everyone's responsibility
- Report suspicious activity immediately (no penalty)
- Questions are encouraged (security through education)
- Continuous learning and improvement
- Incident post-mortems are learning opportunities, not blame

---

## 13. Document Control

| Section | Owner | Last Updated | Next Review |
|---|---|---|---|
| Zero Trust Principles | CISO | 2026-08-10 | 2026-11-10 |
| Identity Security | Identity Lead | 2026-08-10 | 2026-11-10 |
| Conditional Access | Security Lead | 2026-08-10 | 2026-10-10 |
| PIM Configuration | Identity Lead | 2026-08-10 | 2026-10-10 |
| Defender for Cloud | Security Lead | 2026-08-10 | 2026-10-10 |
| Sentinel & Detection | SOC Manager | 2026-08-10 | 2026-10-10 |
| Key Vault Practices | Infrastructure Lead | 2026-08-10 | 2026-11-10 |
| Network Security | Network Lead | 2026-08-10 | 2026-11-10 |
| Incident Response | Incident Commander | 2026-08-10 | 2026-09-10 |

---

**End of Document**
