# PowerShell Security Scripts & Reports: Azure and Entra ID

Practical scripts, a catalog of reports you can pull from Azure and Entra ID, and a cheat sheet.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Report Catalog: Entra ID](#2-report-catalog-entra-id)
3. [Report Catalog: Azure](#3-report-catalog-azure)
4. [Practical Scripts: Local System](#4-practical-scripts-local-system)
5. [Practical Scripts: Azure (Az Module)](#5-practical-scripts-azure-az-module)
6. [Practical Scripts: Entra ID (Microsoft Graph)](#6-practical-scripts-entra-id-microsoft-graph)
7. [Quick Reference Cheat Sheet](#7-quick-reference-cheat-sheet)

---

## 1. Prerequisites

```powershell
Install-Module Az -Scope CurrentUser                  # Azure resources
Install-Module Microsoft.Graph -Scope CurrentUser     # Entra ID / M365
Install-Module Az.ResourceGraph -Scope CurrentUser    # Fast cross-subscription queries

Connect-AzAccount                                      # Azure
Connect-MgGraph -Scopes "Directory.Read.All"           # Entra ID (add scopes as needed)
```

---

## 2. Report Catalog: Entra ID

All via the Microsoft Graph PowerShell SDK. Run `Find-MgGraphCommand -Command <cmdlet>` to confirm the exact permission a cmdlet needs.

| Category | Report | Key cmdlet | Permission | License |
|---|---|---|---|---|
| **Users** | All users, enabled/disabled, hybrid vs. cloud-only | `Get-MgUser -All` | User.Read.All | Free |
| | Stale accounts (no sign-in in 90+ days) | `Get-MgUser` with `signInActivity` | AuditLog.Read.All | P1 |
| | Guest users and invite status | `Get-MgUser -Filter "userType eq 'Guest'"` | User.Read.All | Free |
| **Authentication** | MFA / auth method registration per user | `Get-MgReportAuthenticationMethodUserRegistrationDetail` | AuditLog.Read.All | P1 |
| | Admins without MFA registered | Same as above, filter on `IsAdmin` | AuditLog.Read.All | P1 |
| **Sign-ins & Audit** | Failed sign-ins, legacy auth, unusual locations | `Get-MgAuditLogSignIn` | AuditLog.Read.All | P1 |
| | Directory changes (role adds, consents, policy edits) | `Get-MgAuditLogDirectoryAudit` | AuditLog.Read.All | Free (7 days) / P1 (30 days) |
| **Privileged Access** | Active directory role members | `Get-MgDirectoryRoleMember` | RoleManagement.Read.Directory | Free |
| | PIM-eligible assignments | `Get-MgRoleManagementDirectoryRoleEligibilitySchedule` | RoleManagement.Read.Directory | P2 |
| | Permanent vs. time-bound active assignments | `Get-MgRoleManagementDirectoryRoleAssignmentSchedule` | RoleManagement.Read.Directory | P2 |
| **Applications** | Expiring/expired secrets and certificates | `Get-MgApplication` | Application.Read.All | Free |
| | High-risk application permissions | `Get-MgServicePrincipalAppRoleAssignedTo` | Application.Read.All | Free |
| | Delegated consent grants (user vs. admin) | `Get-MgOauth2PermissionGrant` | Directory.Read.All | Free |
| | Apps with no owner | `Get-MgApplicationOwner` | Application.Read.All | Free |
| **Conditional Access** | CA policies: state, targets, exclusions, grant controls | `Get-MgIdentityConditionalAccessPolicy` | Policy.Read.All | P1 |
| | Named / trusted locations | `Get-MgIdentityConditionalAccessNamedLocation` | Policy.Read.All | P1 |
| **Identity Protection** | Risky users and risk detections | `Get-MgRiskyUser`, `Get-MgRiskDetection` | IdentityRiskyUser.Read.All | P2 |
| **Groups** | Role-assignable, ownerless, dynamic membership rules | `Get-MgGroup`, `Get-MgGroupOwner` | Group.Read.All | Free |
| **Devices** | Stale devices, join type, compliance state | `Get-MgDevice` | Device.Read.All | Free |
| **Hybrid** | Last Entra Connect sync, synced vs. cloud users | `Get-MgOrganization` | Organization.Read.All | Free |
| **Licensing** | Purchased vs. consumed licenses | `Get-MgSubscribedSku` | Organization.Read.All | Free |
| **Governance** | Access review definitions and decisions | `Get-MgIdentityGovernanceAccessReviewDefinition` | AccessReview.Read.All | P2 / ID Governance |

---

## 3. Report Catalog: Azure

All via the Az module. **Reader** covers most of these; Defender for Cloud reports need **Security Reader**.

| Category | Report | Key cmdlet |
|---|---|---|
| **Inventory** | All resources by type, location, tag | `Get-AzResource` |
| | Cross-subscription queries in seconds (KQL) | `Search-AzGraph` |
| | Untagged / missing required tags | `Get-AzResource` + filter on `Tags` |
| | Resource locks | `Get-AzResourceLock` |
| **RBAC** | Role assignments at any scope | `Get-AzRoleAssignment` |
| | Owners / User Access Administrators per subscription | `Get-AzRoleAssignment -RoleDefinitionName Owner` |
| | Custom role definitions and their permissions | `Get-AzRoleDefinition -Custom` |
| | Assignments to deleted/unknown principals | `Get-AzRoleAssignment` (`ObjectType -eq 'Unknown'`) |
| **Identities** | User-assigned managed identities | `Get-AzUserAssignedIdentity` |
| | Resources with system-assigned identity | `Search-AzGraph` on `identity.type` |
| **Azure Policy** | Non-compliant resources by policy | `Get-AzPolicyState` |
| | Compliance summary per subscription | `Get-AzPolicyStateSummary` |
| | Policy assignments and effects | `Get-AzPolicyAssignment` |
| | Active exemptions | `Get-AzPolicyExemption` |
| **Defender for Cloud** | Secure score | `Get-AzSecuritySecureScore` |
| | Recommendations / assessments | `Get-AzSecurityAssessment` |
| | Defender plans enabled per subscription | `Get-AzSecurityPricing` |
| | Security alerts | `Get-AzSecurityAlert` |
| **Network** | NSG rules open to the internet | `Get-AzNetworkSecurityGroup` |
| | Subnets with no NSG | `Get-AzVirtualNetwork` (subnet `NetworkSecurityGroup`) |
| | Public IP addresses and what they're attached to | `Get-AzPublicIpAddress` |
| | Private endpoints and target resources | `Get-AzPrivateEndpoint` |
| **Storage** | Public blob access, TLS, shared key, network access | `Get-AzStorageAccount` |
| **Key Vault** | Purge protection, soft delete, RBAC vs. access policies | `Get-AzKeyVault` |
| | Secrets/keys/certs nearing expiry | `Get-AzKeyVaultSecret`, `Get-AzKeyVaultCertificate` |
| **Data** | SQL servers with public network access | `Get-AzSqlServer` |
| **Compute** | VMs, power state, OS, public exposure | `Get-AzVM -Status` |
| **Logging** | Diagnostic settings per resource | `Get-AzDiagnosticSetting -ResourceId <id>` |
| | Activity log (who changed what) | `Get-AzActivityLog` |
| **Cost** | Usage and spend by resource | `Get-AzConsumptionUsageDetail` |

> **Tip:** For anything spanning many subscriptions, `Search-AzGraph` (Azure Resource Graph) is far faster than looping `Set-AzContext`. See script 5.7.

---

## 4. Practical Scripts: Local System

### 4.1 Quick host triage snapshot

```powershell
[PSCustomObject]@{
    Hostname    = $env:COMPUTERNAME
    LoggedOn    = (Get-CimInstance Win32_ComputerSystem).UserName
    OS          = (Get-CimInstance Win32_OperatingSystem).Caption
    LastBoot    = (Get-CimInstance Win32_OperatingSystem).LastBootUpTime
    LocalAdmins = (Get-LocalGroupMember -Group Administrators).Name -join "; "
}

# Listening ports mapped to the owning process
Get-NetTCPConnection -State Listen |
    Select-Object LocalAddress, LocalPort,
        @{ N = 'Process'; E = { (Get-Process -Id $_.OwningProcess).ProcessName } } |
    Sort-Object LocalPort
```

### 4.2 Failed logons in the last 24 hours (requires admin)

```powershell
Get-WinEvent -FilterHashtable @{
    LogName   = 'Security'
    Id        = 4625
    StartTime = (Get-Date).AddDays(-1)
} -ErrorAction SilentlyContinue |
    Select-Object TimeCreated,
        @{ N = 'TargetUser'; E = { $_.Properties[5].Value } },
        @{ N = 'SourceIP';   E = { $_.Properties[19].Value } } |
    Group-Object SourceIP |
    Sort-Object Count -Descending
```

### 4.3 Hash files for integrity / IOC matching

```powershell
Get-ChildItem C:\Users\Public\Downloads -File |
    Get-FileHash -Algorithm SHA256 |
    Select-Object Hash, Path |
    Export-Csv .\hashes.csv -NoTypeInformation
```

### 4.4 Test connectivity (private endpoint / firewall troubleshooting)

```powershell
Test-NetConnection -ComputerName stprod01.blob.core.windows.net -Port 443
Resolve-DnsName stprod01.blob.core.windows.net    # does it resolve to a private IP?
```

---

## 5. Practical Scripts: Azure (Az Module)

### Connecting

```powershell
Connect-AzAccount                                     # interactive
Connect-AzAccount -Identity                           # managed identity (Automation, VM)
Get-AzSubscription
Set-AzContext -Subscription "Prod-Sub"
```

### 5.1 Storage account security audit across all subscriptions

```powershell
$results = foreach ($sub in Get-AzSubscription) {
    Set-AzContext -SubscriptionId $sub.Id | Out-Null

    foreach ($sa in Get-AzStorageAccount) {
        [PSCustomObject]@{
            Subscription     = $sub.Name
            StorageAccount   = $sa.StorageAccountName
            ResourceGroup    = $sa.ResourceGroupName
            PublicBlobAccess = $sa.AllowBlobPublicAccess
            HttpsOnly        = $sa.EnableHttpsTrafficOnly
            MinTls           = $sa.MinimumTlsVersion
            SharedKeyAllowed = $sa.AllowSharedKeyAccess
            PublicNetwork    = $sa.PublicNetworkAccess
        }
    }
}

$results |
    Where-Object { $_.PublicBlobAccess -or $_.MinTls -ne 'TLS1_2' -or -not $_.HttpsOnly } |
    Export-Csv .\storage-findings.csv -NoTypeInformation
```

### 5.2 NSG rules allowing risky ports from the internet

```powershell
$riskyPorts = @('22', '3389', '445', '*')
$anySources = @('*', 'Internet', '0.0.0.0/0')

$findings = foreach ($nsg in Get-AzNetworkSecurityGroup) {
    foreach ($rule in $nsg.SecurityRules) {
        $openToWorld = $rule.SourceAddressPrefix | Where-Object { $_ -in $anySources }
        $badPort     = $rule.DestinationPortRange | Where-Object { $_ -in $riskyPorts }

        if ($rule.Direction -eq 'Inbound' -and $rule.Access -eq 'Allow' -and $openToWorld -and $badPort) {
            [PSCustomObject]@{
                NSG      = $nsg.Name
                RG       = $nsg.ResourceGroupName
                Rule     = $rule.Name
                Ports    = $rule.DestinationPortRange -join ','
                Source   = $rule.SourceAddressPrefix -join ','
                Priority = $rule.Priority
            }
        }
    }
}
$findings | Format-Table -AutoSize
```

### 5.3 Who has Owner on the subscription?

```powershell
$subId = (Get-AzContext).Subscription.Id
Get-AzRoleAssignment -RoleDefinitionName Owner -Scope "/subscriptions/$subId" |
    Select-Object DisplayName, SignInName, ObjectType, Scope |
    Sort-Object ObjectType
```

Watch for `ServicePrincipal`, `Group`, and `Unknown` (deleted principal) object types.

### 5.4 Resources missing a required tag

```powershell
Get-AzResource |
    Where-Object { -not $_.Tags -or -not $_.Tags.ContainsKey('Owner') } |
    Select-Object Name, ResourceType, ResourceGroupName |
    Export-Csv .\untagged.csv -NoTypeInformation
```

### 5.5 Non-compliant Azure Policy states

```powershell
Get-AzPolicyState -Filter "ComplianceState eq 'NonCompliant'" |
    Group-Object PolicyDefinitionName |
    Sort-Object Count -Descending |
    Select-Object Count, Name -First 15
```

### 5.6 Key Vault hygiene

```powershell
Get-AzKeyVault | ForEach-Object {
    $kv = Get-AzKeyVault -VaultName $_.VaultName
    [PSCustomObject]@{
        Vault            = $kv.VaultName
        PurgeProtection  = $kv.EnablePurgeProtection
        RbacAuthorization = $kv.EnableRbacAuthorization
        PublicNetwork    = $kv.PublicNetworkAccess
    }
} | Where-Object { -not $_.PurgeProtection -or -not $_.RbacAuthorization }
```

### 5.7 Cross-subscription query with Resource Graph (fast)

```powershell
# All storage accounts allowing public blob access, across every subscription you can read
Search-AzGraph -First 1000 -Query @"
Resources
| where type =~ 'microsoft.storage/storageaccounts'
| where properties.allowBlobPublicAccess == true
| project name, resourceGroup, subscriptionId, location
"@
```

### 5.8 Defender for Cloud: plans and secure score

```powershell
Get-AzSecurityPricing | Select-Object Name, PricingTier
Get-AzSecuritySecureScore | Select-Object DisplayName, Percentage
```

### 5.9 Safe change pattern: `-WhatIf` first

```powershell
Set-AzStorageAccount -ResourceGroupName rg-app -Name stapp01 -AllowBlobPublicAccess $false -WhatIf
Set-AzStorageAccount -ResourceGroupName rg-app -Name stapp01 -AllowBlobPublicAccess $false
```

---

## 6. Practical Scripts: Entra ID (Microsoft Graph)

### Connecting with least privilege

```powershell
Connect-MgGraph -Scopes "Application.Read.All","Directory.Read.All","AuditLog.Read.All"
Get-MgContext
Find-MgGraphCommand -Command Get-MgApplication | Select-Object -ExpandProperty Permissions
```

### 6.1 App registrations with secrets/certs expiring in 30 days (or expired)

```powershell
$cutoff = (Get-Date).AddDays(30)
$apps   = Get-MgApplication -All -Property "id,appId,displayName,passwordCredentials,keyCredentials"

$report = foreach ($app in $apps) {
    foreach ($cred in @($app.PasswordCredentials) + @($app.KeyCredentials)) {
        if ($cred -and $cred.EndDateTime -lt $cutoff) {
            [PSCustomObject]@{
                App       = $app.DisplayName
                AppId     = $app.AppId
                ExpiresOn = $cred.EndDateTime
                Status    = if ($cred.EndDateTime -lt (Get-Date)) { 'EXPIRED' } else { 'Expiring' }
            }
        }
    }
}
$report | Sort-Object ExpiresOn | Format-Table -AutoSize
```

### 6.2 Members of privileged directory roles

```powershell
$privileged = @('Global Administrator','Privileged Role Administrator',
                'Application Administrator','Cloud Application Administrator')

foreach ($role in Get-MgDirectoryRole -All | Where-Object DisplayName -in $privileged) {
    Get-MgDirectoryRoleMember -DirectoryRoleId $role.Id -All | ForEach-Object {
        [PSCustomObject]@{
            Role       = $role.DisplayName
            MemberType = $_.AdditionalProperties['@odata.type'] -replace '#microsoft.graph.',''
            Name       = $_.AdditionalProperties['displayName']
            UPN        = $_.AdditionalProperties['userPrincipalName']
        }
    }
}
```

> Active assignments only. For PIM-eligible, use `Get-MgRoleManagementDirectoryRoleEligibilitySchedule`.

### 6.3 Users with no sign-in for 90+ days

```powershell
$threshold = (Get-Date).AddDays(-90)

Get-MgUser -All -Property "displayName,userPrincipalName,accountEnabled,signInActivity" |
    Where-Object {
        $_.AccountEnabled -and
        ($_.SignInActivity.LastSignInDateTime -lt $threshold -or -not $_.SignInActivity.LastSignInDateTime)
    } |
    Select-Object DisplayName, UserPrincipalName,
        @{ N = 'LastSignIn'; E = { $_.SignInActivity.LastSignInDateTime } } |
    Export-Csv .\stale-users.csv -NoTypeInformation
```

### 6.4 Service principals with high-risk Graph application permissions

```powershell
$graphSp   = Get-MgServicePrincipal -Filter "appId eq '00000003-0000-0000-c000-000000000000'"
$dangerous = @('RoleManagement.ReadWrite.Directory','AppRoleAssignment.ReadWrite.All',
               'Application.ReadWrite.All','Directory.ReadWrite.All','Mail.ReadWrite')

$roleMap = @{}
$graphSp.AppRoles | ForEach-Object { $roleMap[$_.Id] = $_.Value }

Get-MgServicePrincipalAppRoleAssignedTo -ServicePrincipalId $graphSp.Id -All |
    Where-Object { $roleMap[$_.AppRoleId] -in $dangerous } |
    Select-Object PrincipalDisplayName, @{ N = 'Permission'; E = { $roleMap[$_.AppRoleId] } }
```

### 6.5 Admins without MFA registered

```powershell
Get-MgReportAuthenticationMethodUserRegistrationDetail -All |
    Where-Object { $_.IsAdmin -and -not $_.IsMfaRegistered } |
    Select-Object UserPrincipalName, IsMfaCapable,
        @{ N = 'Methods'; E = { $_.MethodsRegistered -join ', ' } }
```

### 6.6 Conditional Access policy summary

```powershell
Get-MgIdentityConditionalAccessPolicy -All |
    Select-Object DisplayName, State,
        @{ N = 'IncludeUsers';  E = { $_.Conditions.Users.IncludeUsers -join ',' } },
        @{ N = 'ExcludeGroups'; E = { $_.Conditions.Users.ExcludeGroups -join ',' } },
        @{ N = 'Grant';         E = { $_.GrantControls.BuiltInControls -join ',' } } |
    Sort-Object State
```

Review `ExcludeGroups` closely. Exclusions are where CA coverage quietly erodes.

### 6.7 Tenant-wide delegated consent grants

```powershell
Get-MgOauth2PermissionGrant -All |
    Where-Object ConsentType -eq 'AllPrincipals' |
    ForEach-Object {
        [PSCustomObject]@{
            App    = (Get-MgServicePrincipal -ServicePrincipalId $_.ClientId).DisplayName
            Scopes = $_.Scope.Trim()
        }
    } | Where-Object Scopes -match 'Mail\.|Files\.|ReadWrite'
```

### 6.8 Hybrid sync health

```powershell
Get-MgOrganization | Select-Object DisplayName, OnPremisesSyncEnabled, OnPremisesLastSyncDateTime
```

---

## 7. Quick Reference Cheat Sheet

```powershell
# DISCOVER
Get-Command *keyword*            Get-Help <cmd> -Examples        <obj> | Get-Member

# PIPELINE
| Where-Object Prop -eq 'x'      | Select-Object A,B -First 5     | Sort-Object Prop -Descending
| Group-Object Prop              | Measure-Object -Sum -Average   | ForEach-Object { $_.Name }

# OUTPUT
| Format-Table -AutoSize         | Export-Csv f.csv -NoTypeInformation
| ConvertTo-Json -Depth 10       | Out-File f.txt                 | Out-GridView (Windows GUI)

# VARIABLES & TYPES
$s = "text"   $n = 5   $b = $true   $a = @(1,2,3)   $h = @{k='v'}   [PSCustomObject]@{A=1}

# OPERATORS
-eq -ne -gt -ge -lt -le   -like -match   -contains -in   -and -or -not

# ERRORS
try { cmd -ErrorAction Stop } catch { $_.Exception.Message } finally { }

# AZURE
Connect-AzAccount   Get-AzSubscription   Set-AzContext -Subscription x   Get-AzResource   Search-AzGraph -Query "..."

# GRAPH
Connect-MgGraph -Scopes "X.Read.All"   Get-MgContext   Find-MgGraphCommand -Command <cmd>   (always use -All)
```
