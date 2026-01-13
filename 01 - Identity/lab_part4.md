# 4. Azure Policies

**Lab instructions reminder:**

#### Tag inheritance policies
Use Azure Policies to enforce tag inheritance:
- `team = A` applied at the Team A Management Group
- `team = B` applied at the Team B Management Group
- `env = dev` applied at Dev subscriptions
- `env = prod` applied at Prod subscriptions


#### Location restriction policies
Use Azure Policies to restrict resource deployment locations:
- Team A can deploy resources only in France Central
- Team B can deploy resources only in Switzerland North

<br>

Cmdlets used in this lab :
- [Get-AzPolicyDefinition](https://learn.microsoft.com/en-us/powershell/module/az.resources/get-azpolicydefinition)
- [New-AzPolicyAssignment](https://learn.microsoft.com/en-us/powershell/module/az.resources/new-azpolicyassignment)
- [Start-AzPolicyRemediation](https://learn.microsoft.com/en-us/powershell/module/az.policyinsights/start-azpolicyremediation)
- [New-AzTag](https://learn.microsoft.com/en-us/powershell/module/az.resources/new-aztag?view=azps-15.1.0)

Useful documentations :
- [Assign policies - Powershell]([https://](https://learn.microsoft.com/en-us/azure/governance/policy/assign-policy-powershell))

<br>

## 1. Tag Inheritance Policies


### Tag Inheritance Strategy : Management Groups scope

**Policy 1: Add team tag to subscriptions**
- Policy definition: **"Add a tag to subscriptions"**
- Scope: Management Group (Team A / Team B)
- Parameters:
  - Tag Name: `team`
  - Tag Value: `a` or `b`
- Effect: Adds the team tag to all subscriptions under the MG
- Remediation: Create remediation task for existing subscriptions

**Policy 2: Inherit tags from subscription to resources**
- Policy definition: **"Inherit a tag from the subscription if missing"**
- Scope: Management Group (Team A / Team B)
- Parameters:
  - Tag Name: `team`
- Effect: Resources inherit the team tag from their subscription  

> Note : I done those steps via the Azure Portal

<br>

### Tag Inheritance Strategy : Subscriptions scope
Here, we'll manually apply the `env` tag at the subscription level, and let the previously created policies do the job :)  
So th only thing we have to do is to create a tag for the subscriptions :

```powershell
$subs = @(
    @{Name="Team A - Dev"; Tags=@{env="dev"}},
    @{Name="Team A - Prod"; Tags=@{env="prod"}},
    @{Name="Team B - Dev"; Tags=@{env="dev"}},
    @{Name="Team B - Prod"; Tags=@{env="prod"}}
)

$subs | %{New-AzTag -ResourceId "subscriptions/$((Get-AzSubscription -SubscriptionName $_.Name).SubscriptionId)"  -Tag $_.Tags}
```

<details>
<summary>Output  (part of) :</summary>

```
Id         : /subscriptions/xxxx-xxxx-xxxx/providers/Microsoft.Resources/tags/default
Name       : default
Type       : Microsoft.Resources/tags
Properties :
             Name  Value
             ====  =====
             env   dev


Id         : /subscriptions/xxxx-xxxx-xxxx/providers/Microsoft.Resources/tags/default
Name       : default
Type       : Microsoft.Resources/tags
Properties :
             Name  Value
             ====  =====
             env   prod
```
</details>

<br>

## 2. Location Restriction Policies


**Policy: Allowed locations**
- Policy definition: **"Allowed locations"**
- Scope: Management Group (Team A / Team B)
- Parameters:
  - Allowed locations: `France Central` (Team A) / `Switzerland North` (Team B)
- Effect: Deny resource creation outside allowed location  


```powershell
$teamAMG = Get-AzManagementGroup -GroupName "Team_A"
$teamBMG = Get-AzManagementGroup -GroupName "Team_B"

# Get the policy definition
$policyDef = Get-AzPolicyDefinition | Where-Object { $_.DisplayName -eq "Allowed locations" }

# Assign to Team A - France Central only
New-AzPolicyAssignment `
    -Name "allowed-locations-team-a" `
    -DisplayName "Allowed locations - France Central" `
    -Scope $teamAMG.Id `
    -PolicyDefinition $policyDef `
    -PolicyParameterObject @{listOfAllowedLocations=@("francecentral")} `
    -Location "francecentral"

# Assign to Team B - Switzerland North only
New-AzPolicyAssignment `
    -Name "allowed-locations-team-b" `
    -DisplayName "Allowed locations - Switzerland North" `
    -Scope $teamBMG.Id `
    -PolicyDefinition $policyDef `
    -PolicyParameterObject @{listOfAllowedLocations=@("switzerlandnorth")} `
    -Location "switzerlandnorth"
```

<details>
<summary>Output (part of):</summary>

```
Metadata                     : @{createdBy=xxxx-xxxx-xxxx; createdOn=13/01/2026 04:05:31}
NonComplianceMessage         :
NotScope                     :
Parameter                    : @{listOfAllowedLocations=}
DefinitionVersion            : 1.*.*
Description                  :
DisplayName                  : Allowed locations - France Central
EnforcementMode              : Default
Id                           : /providers/Microsoft.Management/managementGroups/Team_A/providers/Microsoft.Authorization/policyAssignmen
                               ts/allowed-locations-team-a
IdentityPrincipalId          :
IdentityTenantId             :
IdentityType                 :
IdentityUserAssignedIdentity : Microsoft.Azure.PowerShell.Cmdlets.Policy.Models.IdentityUserAssignedIdentities
Location                     : francecentral
Name                         : allowed-locations-team-a
Override                     :
PolicyDefinitionId           : /providers/Microsoft.Authorization/policyDefinitions/e56962a6-4747-49cd-b67b-bf8b01975c4c
ResourceSelector             :
Scope                        : /providers/Microsoft.Management/managementGroups/Team_A
Type                         : Microsoft.Authorization/policyAssignments



Metadata                     : @{createdBy=xxxx-xxxx-xxxx; createdOn=13/01/2026 04:07:26}
NonComplianceMessage         :
NotScope                     :
Parameter                    : @{listOfAllowedLocations=}
DefinitionVersion            : 1.*.*
Description                  :
DisplayName                  : Allowed locations - Switzerland North
EnforcementMode              : Default
Id                           : /providers/Microsoft.Management/managementGroups/Team_B/providers/Microsoft.Authorization/policyAssignmen
                               ts/allowed-locations-team-b
IdentityPrincipalId          :
IdentityTenantId             :
IdentityType                 :
IdentityUserAssignedIdentity : Microsoft.Azure.PowerShell.Cmdlets.Policy.Models.IdentityUserAssignedIdentities
Location                     : switzerlandnorth
Name                         : allowed-locations-team-b
Override                     :
PolicyDefinitionId           : /providers/Microsoft.Authorization/policyDefinitions/e56962a6-4747-49cd-b67b-bf8b01975c4c
ResourceSelector             :
Scope                        : /providers/Microsoft.Management/managementGroups/Team_B
Type                         : Microsoft.Authorization/policyAssignments


```
</details>

<br>

---

### Final governance architecture :

```
🗂️  Team A (MG)
    ├─ 📜 Policy: "Add a tag to subscriptions" (team=a)
    ├─ 📜 Policy: "Inherit a tag from the subscription if missing" (team)
    ├─ 📜 Policy: "Allowed locations" (France Central)
    ├─ 🔑 Team A - Dev
    │   └─ 🏷️  Manual tag: "env=dev"
    └─ 🔑 Team A - Prod
        └─ 🏷️  Manual tag: "env=prod"

🗂️  Team B (MG)
    ├─ 📜 Policy: "Add a tag to subscriptions" (team=b)
    ├─ 📜 Policy: "Inherit a tag from the subscription if missing" (team)
    ├─ 📜 Policy: "Allowed locations" (Switzerland North)
    ├─ 🔑 Team B - Dev
    │   └─ 🏷️  Manual tag: "env=dev"
    └─ 🔑 Team B - Prod
        └─ 🏷️  Maual tag: "env=prod"
```