# PowerShell Fundamentals for Cloud Security Engineers

A practical guide to PowerShell: the core concepts, the syntax you'll use every day, and working scripts for local, Azure, and Entra ID security tasks.

---

## Table of Contents

1. [What PowerShell Is (and Why It's Different)](#1-what-powershell-is-and-why-its-different)
2. [Setup](#2-setup)
3. [The Three Commands That Teach You Everything Else](#3-the-three-commands-that-teach-you-everything-else)
4. [Cmdlets and the Verb-Noun Pattern](#4-cmdlets-and-the-verb-noun-pattern)
5. [Objects and the Pipeline (The Most Important Concept)](#5-objects-and-the-pipeline-the-most-important-concept)
6. [Variables and Data Types](#6-variables-and-data-types)
7. [Arrays and Hashtables](#7-arrays-and-hashtables)
8. [Operators](#8-operators)
9. [Conditionals](#9-conditionals)
10. [Loops](#10-loops)
11. [Functions and Parameters](#11-functions-and-parameters)
12. [Error Handling](#12-error-handling)
13. [Working with Files, CSV, and JSON](#13-working-with-files-csv-and-json)
14. [Writing and Running Scripts](#14-writing-and-running-scripts)
15. [Practical Scripts: Local System](#15-practical-scripts-local-system)
16. [Practical Scripts: Azure (Az Module)](#16-practical-scripts-azure-az-module)
17. [Practical Scripts: Entra ID (Microsoft Graph)](#17-practical-scripts-entra-id-microsoft-graph)
18. [Common Gotchas](#18-common-gotchas)
19. [Security Best Practices for Scripts](#19-security-best-practices-for-scripts)
20. [Practice Exercises](#20-practice-exercises)
21. [Quick Reference Cheat Sheet](#21-quick-reference-cheat-sheet)

---

## 1. What PowerShell Is (and Why It's Different)

PowerShell is a shell **and** a scripting language built on .NET. The single biggest difference from Bash or CMD:

| Bash / CMD | PowerShell |
|---|---|
| Commands pass **text** to each other | Commands pass **objects** to each other |
| You parse output with `grep`, `awk`, `cut` | You access **properties** directly (`.Name`, `.Status`) |
| Output format = data | Output format is just a display of the data |

**Example that drives it home:**

```bash
# Bash: find a process's memory — you parse columns of text
ps aux | grep chrome | awk '{print $6}'
```

```powershell
# PowerShell: the output IS a structured object — just ask for the property
Get-Process chrome | Select-Object Name, WorkingSet
```

Think of it like this: Bash hands you a printed report. PowerShell hands you the spreadsheet behind the report.

---

## 2. Setup

### Versions

| Version | Name | Notes |
|---|---|---|
| 5.1 | Windows PowerShell | Built into Windows. Legacy, no new features. |
| 7.x | PowerShell (Core) | Cross-platform (Windows/macOS/Linux). **Use this.** |

```powershell
# Check your version
$PSVersionTable.PSVersion
```

Install PowerShell 7: `winget install Microsoft.PowerShell` (Windows) or `brew install powershell` (macOS).

### Editor

Use **VS Code** with the **PowerShell extension**. You get IntelliSense, debugging, and linting (PSScriptAnalyzer).

### Modules you'll need

```powershell
# Azure resources
Install-Module Az -Scope CurrentUser

# Entra ID / M365 (replaces the deprecated AzureAD and MSOnline modules)
Install-Module Microsoft.Graph -Scope CurrentUser
```

---

## 3. The Three Commands That Teach You Everything Else

If you only memorize three commands, make it these. They make PowerShell self-documenting.

| Command | Question it answers | Example |
|---|---|---|
| `Get-Command` | "What command does X?" | `Get-Command *firewall*` |
| `Get-Help` | "How do I use this command?" | `Get-Help Get-Process -Examples` |
| `Get-Member` | "What's inside this object?" | `Get-Process \| Get-Member` |

```powershell
# Find every command related to network security groups
Get-Command -Noun *NetworkSecurityGroup*

# See real usage examples (run Update-Help once first)
Get-Help Get-AzRoleAssignment -Examples

# Discover the properties and methods of what a command returns
Get-Service | Get-Member
```

> **Tip:** `Get-Member` is how you figure out what properties you can filter on. Whenever you think "what fields does this have?", pipe it to `Get-Member`.

---

## 4. Cmdlets and the Verb-Noun Pattern

Every native command follows `Verb-Noun`. Once you know the pattern, you can guess commands.

| Verb | Meaning | Example |
|---|---|---|
| `Get` | Read/retrieve | `Get-AzVM` |
| `Set` | Modify existing | `Set-AzStorageAccount` |
| `New` | Create | `New-AzResourceGroup` |
| `Remove` | Delete | `Remove-AzRoleAssignment` |
| `Start` / `Stop` | Change running state | `Stop-Service` |
| `Test` | Check a condition, returns true/false | `Test-Path`, `Test-NetConnection` |
| `Invoke` | Run an action | `Invoke-RestMethod` |

```powershell
# See all approved verbs
Get-Verb
```

### Parameters

```powershell
Get-ChildItem -Path C:\Logs -Filter *.log -Recurse
#  cmdlet      param  value  param value  switch (no value)
```

---

## 5. Objects and the Pipeline (The Most Important Concept)

The pipe `|` passes **objects** from one command to the next. Most real work is a chain of:

**Get → Filter → Select → Sort → Output**

```powershell
Get-Service |                              # Get: all services
    Where-Object Status -eq 'Running' |    # Filter: only running ones
    Select-Object Name, StartType |        # Select: only the fields I care about
    Sort-Object Name |                     # Sort
    Export-Csv running.csv -NoTypeInformation   # Output
```

### The core pipeline cmdlets

| Cmdlet | Alias | SQL equivalent | Purpose |
|---|---|---|---|
| `Where-Object` | `?` | `WHERE` | Filter rows |
| `Select-Object` | `select` | `SELECT` | Pick columns / first N |
| `Sort-Object` | `sort` | `ORDER BY` | Sort |
| `Group-Object` | `group` | `GROUP BY` | Group and count |
| `Measure-Object` | `measure` | `COUNT/SUM/AVG` | Aggregate |
| `ForEach-Object` | `%` | — | Run code per item |

### `$_` (or `$PSItem`) = "the current object in the pipeline"

```powershell
# Processes using more than 500 MB
Get-Process | Where-Object { $_.WorkingSet -gt 500MB }

# Simplified syntax (works for single conditions)
Get-Process | Where-Object WorkingSet -gt 500MB
```

### Calculated properties (rename or transform a column)

```powershell
Get-Process | Select-Object Name,
    @{ Name = 'MemoryMB'; Expression = { [math]::Round($_.WorkingSet / 1MB, 2) } }
```

### Group-Object example (great for triage)

```powershell
# Count failed logons by account — "who's getting hammered?"
Get-WinEvent -FilterHashtable @{ LogName='Security'; Id=4625 } -MaxEvents 1000 |
    Group-Object { $_.Properties[5].Value } |
    Sort-Object Count -Descending |
    Select-Object Count, Name -First 10
```

---

## 6. Variables and Data Types

Variables start with `$`. Types are inferred but you can enforce them.

```powershell
$name       = "Storage-Prod-01"        # string
$count      = 42                       # int
$enabled    = $true                    # boolean ($true / $false)
$nothing    = $null                    # null
$today      = Get-Date                 # DateTime object
[int]$port  = "443"                    # cast string -> int

$name.GetType().Name                   # -> String
```

### Strings: single vs double quotes

```powershell
$env = "Prod"
"Environment: $env"          # Double quotes EXPAND variables -> Environment: Prod
'Environment: $env'          # Single quotes are LITERAL     -> Environment: $env

# Expanding a property or expression needs $( )
$vm = Get-AzVM -Name "web01" -ResourceGroupName "rg-web"
"VM location: $($vm.Location)"
```

### Useful string methods

```powershell
$s = "  Contoso-Prod-Storage  "
$s.Trim()                  # remove whitespace
$s.ToLower()
$s.Contains("Prod")        # True (case-sensitive)
$s.Split("-")              # array of parts
$s.Replace("Prod","Dev")
"{0} has {1} findings" -f "sub-01", 7   # format operator
```

### Environment variables

```powershell
$env:USERNAME
$env:PATH
$env:MY_TENANT_ID = "xxxx"   # set for current session only
```

### Dates (constantly useful for security work)

```powershell
$now       = Get-Date
$thirtyAgo = (Get-Date).AddDays(-30)
$inNinety  = (Get-Date).AddDays(90)
(Get-Date).ToString("yyyy-MM-dd")       # 2026-10-04
```

---

## 7. Arrays and Hashtables

### Arrays (ordered list)

```powershell
$ports = @(22, 3389, 445)
$ports[0]             # 22
$ports.Count          # 3
$ports -contains 3389 # True

# Efficient way to build up a list in a loop
$results = [System.Collections.Generic.List[object]]::new()
$results.Add("item1")
```

### Hashtables (key/value pairs — like a dictionary)

```powershell
$riskyPorts = @{
    22   = "SSH"
    3389 = "RDP"
    445  = "SMB"
}
$riskyPorts[3389]          # RDP
$riskyPorts.ContainsKey(22)
$riskyPorts.Keys
```

### Custom objects (how you build report rows)

```powershell
$finding = [PSCustomObject]@{
    Resource = "stprod01"
    Issue    = "Public blob access enabled"
    Severity = "High"
}
$finding.Severity   # High
```

> **Pattern you'll use constantly:** loop over resources → build a `[PSCustomObject]` per finding → collect them → `Export-Csv`.

---

## 8. Operators

PowerShell uses **word operators**, not symbols. `>` is redirection, not "greater than."

### Comparison

| Operator | Meaning | Example |
|---|---|---|
| `-eq` / `-ne` | equal / not equal | `$a -eq 5` |
| `-gt` / `-ge` | greater than / or equal | `$days -gt 90` |
| `-lt` / `-le` | less than / or equal | |
| `-like` | wildcard match | `$name -like "*prod*"` |
| `-match` | regex match | `$ip -match '^10\.'` |
| `-contains` | collection contains value | `$list -contains "Owner"` |
| `-in` | value is in collection | `"Owner" -in $list` |

> Comparisons are **case-insensitive** by default. Use `-ceq`, `-clike`, `-cmatch` for case-sensitive.

### Logical

```powershell
($a -gt 5) -and ($b -eq "x")
($a -gt 5) -or  ($b -eq "x")
-not $enabled      # or !$enabled
```

### Regex capture example

```powershell
$upn = "jdoe@contoso.com"
if ($upn -match '^(.+)@(.+)$') {
    $Matches[1]   # jdoe
    $Matches[2]   # contoso.com
}
```

---

## 9. Conditionals

```powershell
$severity = "High"

if ($severity -eq "Critical") {
    "Page on-call"
}
elseif ($severity -eq "High") {
    "Open ticket within 4 hours"
}
else {
    "Add to backlog"
}
```

### Switch (cleaner than many elseif's)

```powershell
switch ($severity) {
    "Critical" { "Page on-call" }
    "High"     { "Ticket within 4 hours" }
    "Medium"   { "Ticket within 3 days" }
    default    { "Backlog" }
}
```

---

## 10. Loops

```powershell
# foreach statement — use for collections in variables (fastest)
$rgs = Get-AzResourceGroup
foreach ($rg in $rgs) {
    "Checking $($rg.ResourceGroupName)"
}

# ForEach-Object — use inside a pipeline
Get-AzResourceGroup | ForEach-Object { $_.ResourceGroupName }

# for — when you need an index
for ($i = 0; $i -lt 5; $i++) { "Attempt $i" }

# while — retry/poll pattern
$tries = 0
while (-not (Test-Path "C:\ready.flag") -and $tries -lt 10) {
    Start-Sleep -Seconds 5
    $tries++
}
```

| Use | When |
|---|---|
| `foreach ($x in $list)` | You already have the data in a variable |
| `ForEach-Object` | You're mid-pipeline / streaming large data |
| `ForEach-Object -Parallel` (PS7) | Many slow independent calls (e.g., per subscription) |

`break` exits a loop, `continue` skips to the next item.

---

## 11. Functions and Parameters

Functions turn one-off commands into reusable tools.

### Basic

```powershell
function Get-Greeting {
    param([string]$Name)
    "Hello, $Name"
}
Get-Greeting -Name "Analyst"
```

### Advanced function (production quality)

```powershell
function Get-ExpiringSecret {
    [CmdletBinding()]                       # adds -Verbose, -ErrorAction, etc.
    param(
        [Parameter(Mandatory)]
        [object[]]$Applications,

        [ValidateRange(1, 365)]
        [int]$DaysAhead = 30                # default value
    )

    $cutoff = (Get-Date).AddDays($DaysAhead)
    foreach ($app in $Applications) {
        foreach ($cred in $app.PasswordCredentials) {
            if ($cred.EndDateTime -lt $cutoff) {
                Write-Verbose "Match: $($app.DisplayName)"
                [PSCustomObject]@{
                    App       = $app.DisplayName
                    AppId     = $app.AppId
                    ExpiresOn = $cred.EndDateTime
                }
            }
        }
    }
}
```

Key ideas:
- **`[CmdletBinding()]`** makes your function behave like a real cmdlet.
- **`Mandatory`** forces the caller to provide a value.
- **`Validate*`** attributes (`ValidateSet`, `ValidateRange`, `ValidatePattern`) reject bad input before your code runs.
- **Return objects, not text.** Let the caller decide whether to display, filter, or export.

### Supporting `-WhatIf` (critical for anything that changes things)

```powershell
function Disable-StaleUser {
    [CmdletBinding(SupportsShouldProcess)]
    param([string]$UserId)

    if ($PSCmdlet.ShouldProcess($UserId, "Disable account")) {
        Update-MgUser -UserId $UserId -AccountEnabled:$false
    }
}

Disable-StaleUser -UserId "jdoe@contoso.com" -WhatIf   # shows what WOULD happen
```

### Splatting (pass parameters as a hashtable)

```powershell
$params = @{
    ResourceGroupName = "rg-sec"
    Name              = "stsecaudit01"
    Location          = "eastus"
    SkuName           = "Standard_LRS"
    MinimumTlsVersion = "TLS1_2"
}
New-AzStorageAccount @params     # note @ not $
```

Much easier to read and edit than one giant line.

---

## 12. Error Handling

PowerShell has two kinds of errors:

| Type | Behavior | Example |
|---|---|---|
| **Non-terminating** | Prints red text, **keeps going** | `Get-Item` on one missing file in a list |
| **Terminating** | Stops execution | Syntax error, `throw` |

`try/catch` only catches **terminating** errors, so add `-ErrorAction Stop` to make a cmdlet's error catchable.

```powershell
try {
    $kv = Get-AzKeyVault -VaultName "kv-prod-01" -ErrorAction Stop
    "Found vault in $($kv.Location)"
}
catch {
    Write-Warning "Failed: $($_.Exception.Message)"
}
finally {
    # always runs — cleanup, disconnects, etc.
}
```

Script-wide strictness (put at the top of scripts):

```powershell
$ErrorActionPreference = 'Stop'   # treat all errors as terminating
Set-StrictMode -Version Latest    # error on undefined variables / typos
```

### Output streams — use the right one

| Cmdlet | Use for |
|---|---|
| `Write-Output` (or just output the object) | Data your script returns |
| `Write-Verbose` | Debug detail (shows with `-Verbose`) |
| `Write-Warning` | Something's off but continuing |
| `Write-Error` | A failure |
| `Write-Host` | Console-only display (can't be captured — use sparingly) |

---

## 13. Working with Files, CSV, and JSON

```powershell
# Files and folders
Test-Path "C:\Reports"
New-Item -ItemType Directory -Path "C:\Reports" -Force
Get-ChildItem -Path C:\Logs -Filter *.log -Recurse
Get-Content .\app.log -Tail 50            # last 50 lines
Get-Content .\app.log -Wait               # live tail, like tail -f
Select-String -Path .\*.log -Pattern "denied|failed"   # grep equivalent
"line of text" | Out-File .\notes.txt -Append

# CSV — the bread and butter of reporting
$users = Import-Csv .\users.csv           # each row becomes an object
$users | Where-Object Department -eq "IT"
$findings | Export-Csv .\findings.csv -NoTypeInformation

# JSON — ARM templates, API responses, policy definitions
$policy = Get-Content .\policy.json -Raw | ConvertFrom-Json
$policy.properties.policyRule.then.effect
$obj | ConvertTo-Json -Depth 10 | Out-File .\out.json
```

> **Gotcha:** `ConvertTo-Json` defaults to depth 2 and silently flattens deeper objects. Always set `-Depth`.

### Calling REST APIs

```powershell
$response = Invoke-RestMethod -Uri "https://api.github.com/repos/PowerShell/PowerShell/releases/latest"
$response.tag_name     # Already parsed into an object, no JSON parsing needed
```

---

## 14. Writing and Running Scripts

A script is just a `.ps1` file.

### Script template

```powershell
<#
.SYNOPSIS
    Short description of what the script does.
.DESCRIPTION
    Longer description.
.PARAMETER OutputPath
    Where to write the report.
.EXAMPLE
    .\Get-StorageAudit.ps1 -OutputPath .\report.csv
#>
[CmdletBinding()]
param(
    [string]$OutputPath = ".\report.csv"
)

$ErrorActionPreference = 'Stop'
Set-StrictMode -Version Latest

# --- main logic here ---
```

The comment block at the top means `Get-Help .\Get-StorageAudit.ps1` works on your own script.

### Running scripts

```powershell
.\MyScript.ps1                      # must use .\ for the current folder
.\MyScript.ps1 -OutputPath C:\out.csv
```

### Execution policy

```powershell
Get-ExecutionPolicy -List
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

| Policy | Meaning |
|---|---|
| `Restricted` | No scripts run |
| `RemoteSigned` | Local scripts run; downloaded scripts must be signed |
| `AllSigned` | Every script must be signed |
| `Bypass` | Nothing blocked |

> **Security note:** Execution policy is a **safety guardrail, not a security boundary.** An attacker can bypass it trivially (`powershell -ExecutionPolicy Bypass`, or piping code in). Real controls are AppLocker/WDAC, Constrained Language Mode, Script Block Logging, and AMSI.

---

## 15. Practical Scripts: Local System

### 15.1 Quick host triage snapshot

```powershell
# Useful first look during an incident on a Windows host
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

### 15.2 Failed logons in the last 24 hours (requires admin)

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

### 15.3 Hash files for integrity / IOC matching

```powershell
Get-ChildItem C:\Users\Public\Downloads -File |
    Get-FileHash -Algorithm SHA256 |
    Select-Object Hash, Path |
    Export-Csv .\hashes.csv -NoTypeInformation
```

### 15.4 Test connectivity (great for private endpoint / firewall troubleshooting)

```powershell
Test-NetConnection -ComputerName stprod01.blob.core.windows.net -Port 443
Resolve-DnsName stprod01.blob.core.windows.net    # does it resolve to a private IP?
```

---

## 16. Practical Scripts: Azure (Az Module)

### Connecting

```powershell
Connect-AzAccount                                     # interactive
Connect-AzAccount -Identity                           # from a managed identity (Automation, VM)
Get-AzSubscription
Set-AzContext -Subscription "Prod-Sub"
```

### 16.1 Storage account security audit across all subscriptions

```powershell
$results = foreach ($sub in Get-AzSubscription) {
    Set-AzContext -SubscriptionId $sub.Id | Out-Null

    foreach ($sa in Get-AzStorageAccount) {
        [PSCustomObject]@{
            Subscription      = $sub.Name
            StorageAccount    = $sa.StorageAccountName
            ResourceGroup     = $sa.ResourceGroupName
            PublicBlobAccess  = $sa.AllowBlobPublicAccess
            HttpsOnly         = $sa.EnableHttpsTrafficOnly
            MinTls            = $sa.MinimumTlsVersion
            SharedKeyAllowed  = $sa.AllowSharedKeyAccess
            PublicNetwork     = $sa.PublicNetworkAccess
        }
    }
}

$results |
    Where-Object { $_.PublicBlobAccess -or $_.MinTls -ne 'TLS1_2' -or -not $_.HttpsOnly } |
    Export-Csv .\storage-findings.csv -NoTypeInformation
```

Note the pattern: **assign the output of a `foreach` directly to a variable**. Every object emitted inside the loop gets collected. No `+=` needed.

### 16.2 NSG rules allowing risky ports from the internet

```powershell
$riskyPorts  = @('22', '3389', '445', '*')
$anySources  = @('*', 'Internet', '0.0.0.0/0')

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

### 16.3 Who has Owner on the subscription?

```powershell
$subId = (Get-AzContext).Subscription.Id
Get-AzRoleAssignment -RoleDefinitionName Owner -Scope "/subscriptions/$subId" |
    Select-Object DisplayName, SignInName, ObjectType, Scope |
    Sort-Object ObjectType
```

Look for `ServicePrincipal` and `Group` object types. Those are often the overlooked standing access.

### 16.4 Find resources missing a required tag

```powershell
Get-AzResource |
    Where-Object { -not $_.Tags -or -not $_.Tags.ContainsKey('Owner') } |
    Select-Object Name, ResourceType, ResourceGroupName |
    Export-Csv .\untagged.csv -NoTypeInformation
```

### 16.5 Non-compliant Azure Policy states

```powershell
Get-AzPolicyState -Filter "ComplianceState eq 'NonCompliant'" |
    Group-Object PolicyDefinitionName |
    Sort-Object Count -Descending |
    Select-Object Count, Name -First 15
```

### 16.6 Safe change pattern: `-WhatIf` first

```powershell
# Preview first
Set-AzStorageAccount -ResourceGroupName rg-app -Name stapp01 -AllowBlobPublicAccess $false -WhatIf

# Then apply
Set-AzStorageAccount -ResourceGroupName rg-app -Name stapp01 -AllowBlobPublicAccess $false
```

---

## 17. Practical Scripts: Entra ID (Microsoft Graph)

> The `AzureAD` and `MSOnline` modules are retired. Use `Microsoft.Graph`.

### Connecting with least privilege

```powershell
# Request only the scopes you need
Connect-MgGraph -Scopes "Application.Read.All","Directory.Read.All","AuditLog.Read.All"
Get-MgContext      # confirm tenant, account, and scopes
```

**Finding which permission a cmdlet needs:**

```powershell
Find-MgGraphCommand -Command Get-MgApplication | Select-Object -ExpandProperty Permissions
```

### 17.1 App registrations with secrets/certs expiring in 30 days (or already expired)

```powershell
$cutoff = (Get-Date).AddDays(30)

$apps = Get-MgApplication -All -Property "id,appId,displayName,passwordCredentials,keyCredentials"

$report = foreach ($app in $apps) {
    foreach ($cred in @($app.PasswordCredentials) + @($app.KeyCredentials)) {
        if ($cred -and $cred.EndDateTime -lt $cutoff) {
            [PSCustomObject]@{
                App       = $app.DisplayName
                AppId     = $app.AppId
                CredType  = if ($cred.PSObject.Properties['Type']) { 'Certificate' } else { 'Secret' }
                ExpiresOn = $cred.EndDateTime
                Status    = if ($cred.EndDateTime -lt (Get-Date)) { 'EXPIRED' } else { 'Expiring' }
            }
        }
    }
}
$report | Sort-Object ExpiresOn | Format-Table -AutoSize
```

### 17.2 Members of privileged directory roles

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

> `Get-MgDirectoryRole` shows **active** assignments only. PIM-eligible assignments live under `Get-MgRoleManagementDirectoryRoleEligibilitySchedule`.

### 17.3 Users who haven't signed in for 90+ days

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

(Requires `AuditLog.Read.All` and an Entra ID P1/P2 license for `signInActivity`.)

### 17.4 Service principals with high-risk Graph application permissions

```powershell
$graphSp   = Get-MgServicePrincipal -Filter "appId eq '00000003-0000-0000-c000-000000000000'"
$dangerous = @('RoleManagement.ReadWrite.Directory','AppRoleAssignment.ReadWrite.All',
               'Application.ReadWrite.All','Directory.ReadWrite.All','Mail.ReadWrite')

$roleMap = @{}
$graphSp.AppRoles | ForEach-Object { $roleMap[$_.Id] = $_.Value }

Get-MgServicePrincipalAppRoleAssignedTo -ServicePrincipalId $graphSp.Id -All |
    Where-Object { $roleMap[$_.AppRoleId] -in $dangerous } |
    Select-Object PrincipalDisplayName,
        @{ N = 'Permission'; E = { $roleMap[$_.AppRoleId] } }
```

This is a classic attack-path check: any app holding these permissions can escalate to Global Admin or read every mailbox.

---

## 18. Common Gotchas

| Gotcha | What happens | Fix |
|---|---|---|
| `if ($x > 5)` | `>` is **redirection**, creates a file named `5` | Use `-gt` |
| `$array += $item` in big loops | Copies the whole array every time, gets very slow | Assign loop output to a variable, or use `List[object]` |
| `$var -eq $null` when `$var` is an array | Filters the array instead of checking null | Put `$null` on the **left**: `$null -eq $var` |
| `ConvertTo-Json` without `-Depth` | Nested data silently becomes strings | `-Depth 10` |
| `Format-Table` then `Export-Csv` | CSV full of formatting garbage | `Format-*` is **last** in the pipeline, display only |
| One result vs many | One result isn't an array, so `.Count` behaves oddly in 5.1 | Wrap: `@(Get-Something).Count` |
| `Write-Host` for data | Output can't be piped or captured | Output objects or use `Write-Output` |
| Forgetting `-All` on Graph cmdlets | Only the first page (often 100) comes back | Add `-All` |
| Aliases in scripts (`%`, `?`, `gci`) | Harder to read and maintain | Use full names in scripts; aliases are fine interactively |

---

## 19. Security Best Practices for Scripts

| Practice | Why |
|---|---|
| **Never hardcode secrets** | Scripts end up in repos, tickets, and screenshots |
| Use **managed identities** (`Connect-AzAccount -Identity`) for automation | No credential to steal or rotate |
| Pull secrets from **Key Vault** at runtime | `Get-AzKeyVaultSecret -VaultName kv -Name x -AsPlainText` |
| Request **least-privilege Graph scopes** | Limit blast radius if a token leaks |
| Support **`-WhatIf`** on anything destructive | Preview before you break production |
| **Sign scripts** in production | Enables `AllSigned` and detects tampering |
| Run **PSScriptAnalyzer** | `Invoke-ScriptAnalyzer .\script.ps1` catches bad patterns |
| Avoid `Invoke-Expression` on input | It's PowerShell's equivalent of `eval()` and an injection risk |

### For the defender side, know these

| Control | What it gives you |
|---|---|
| **Script Block Logging** (Event ID 4104) | Logs the de-obfuscated code that actually ran |
| **Module Logging** (Event ID 4103) | Logs pipeline execution details |
| **Transcription** | Full session transcripts to a file share |
| **AMSI** | Lets Defender/EDR scan script content at runtime |
| **Constrained Language Mode** | Blocks .NET/COM abuse for untrusted users |

Red flags in PowerShell telemetry: `-EncodedCommand`, `-WindowStyle Hidden`, `-NoProfile -ExecutionPolicy Bypass`, `IEX (New-Object Net.WebClient).DownloadString(...)`, `FromBase64String`.

---

## 20. Practice Exercises

Work through these in order. Each builds on the last.

| # | Exercise | Skills practiced |
|---|---|---|
| 1 | List the 10 processes using the most memory, showing MB rounded to 2 decimals | Pipeline, calculated properties |
| 2 | Find all `.log` files over 10 MB under a folder and export name/size/last-modified to CSV | Files, filtering, export |
| 3 | Write a function `Test-Port` that takes `-ComputerName` and `-Port` and returns `$true/$false` | Functions, parameters |
| 4 | Import a CSV of UPNs and report which ones don't exist in Entra ID | Import-Csv, try/catch, Graph |
| 5 | Count resources per resource type in a subscription and show the top 10 | Group-Object, Az |
| 6 | Report every Key Vault where purge protection is off | Az, custom objects |
| 7 | Extend script 16.1 to run across subscriptions with `ForEach-Object -Parallel` | Parallelism (PS7) |
| 8 | Build a script that disables a list of stale guest users, with full `-WhatIf` support and a CSV audit log of actions taken | SupportsShouldProcess, error handling, logging |

---

## 21. Quick Reference Cheat Sheet

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
Connect-AzAccount   Get-AzSubscription   Set-AzContext -Subscription x   Get-AzResource

# GRAPH
Connect-MgGraph -Scopes "X.Read.All"   Get-MgContext   Find-MgGraphCommand -Command <cmd>
```

---

### Recommended next steps

1. **Microsoft Learn:** "Introduction to PowerShell" and "Automate administrative tasks by using PowerShell" learning paths.
2. **Rewrite one manual task per week** as a script (e.g., a CSPM finding you triage by hand).
3. **Learn Pester** (PowerShell's testing framework) once your scripts become tools other people rely on.
4. **Move scripts into Azure Automation or GitHub Actions** with managed identity / OIDC federation, so nothing runs on your laptop with your credentials.
