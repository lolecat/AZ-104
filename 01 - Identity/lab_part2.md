# 2. Management Groups and RBAC

**Lab instructions reminder :**  
Create 2 Management Groups, directly under the Root Management Group:

* Team A
* Team B

#### RBAC assignments:

* "Super Admins" users have Owner role on the Root Management Group
* "Subscriptions Managers" users have Management Group Contributor role on the Root Management Group
* "Team A" users have the Contributor role on the Team A Management Group
* "Team B" users have the Contributor role on the Team B Management Group  

<br>

Cmdlets used in this lab:
- [Get-AzManagementGroup](https://learn.microsoft.com/en-us/powershell/module/az.resources/get-azmanagementgroup?view=azps-15.1.0)
- [New-AzManagementGroup](https://learn.microsoft.com/en-us/powershell/module/az.resources/new-azmanagementgroup?view=azps-15.1.0)
- [Get-AzADGroup](https://learn.microsoft.com/en-us/powershell/module/az.resources/get-azadgroup?view=azps-15.1.0)
- [New-AzRoleAssignment](https://learn.microsoft.com/en-us/powershell/module/az.resources/new-azroleassignment?view=azps-15.1.0)
- [Get-AzRoleAssignment](https://learn.microsoft.com/en-us/powershell/module/az.resources/get-azroleassignment?view=azps-15.1.0)

<br>

## 1. Management Groups creation

We have two management groups to create : "Team A" and "Team B".  

```powershell
# Get the root MG : since it's the only MG for now, just use the Get-AzManagementGroup cmdlet wihtout ay parameter
$rootMG = Get-AzManagementGroup

@("Team_A","Team_B") | %{New-AzManagementGroup -GroupName $_ -DisplayName $_.Replace('_',' ') -ParentId $rootMG.Id}
```  
<details>
<summary>Output</summary>

```
Id                : /providers/Microsoft.Management/managementGroups/Team_A
Type              : Microsoft.Management/managementGroups
Name              : Team_A
TenantId          : xxx-xxx-xxx
DisplayName       : Team A
UpdatedTime       : 30/12/2025 07:30:50
UpdatedBy         : xxx-xxx-xxx
ParentId          : /providers/Microsoft.Management/managementGroups/xxx-xxx-xxx
ParentName        : xxx-xxx-xxx
ParentDisplayName : Tenant Root Group

Id                : /providers/Microsoft.Management/managementGroups/Team_B
Type              : Microsoft.Management/managementGroups
Name              : Team_B
TenantId          : xxx-xxx-xxx
DisplayName       : Team B
UpdatedTime       : 30/12/2025 07:31:17
UpdatedBy         : c2a57f65-c907-4150-84f4-cc95d18c024f
ParentId          : /providers/Microsoft.Management/managementGroups/xxx-xxx-xxx
ParentName        : xxx-xxx-xxx
ParentDisplayName : Tenant Root Group
```
</details>

<br>

## 2. RBAC assignments

### Super Admin and Subscriptions Manager

> -> ***Super Admins users have Owner role on the Root Management Group***

**Objective :** The "Super Admins" group has full persmissions and control over all the managements groups, subscriptions and resources in the tenant (this is acceptable for learning purposes but should be adapted in production environments).

> Microsoft Entra ID and Azure resources are secured independently from one another. That is, Microsoft Entra role assignments do not grant access to Azure resources, and Azure role assignments do not grant access to Microsoft Entra ID. However, if you are a Global Administrator in Microsoft Entra ID, you can assign yourself access to all Azure subscriptions and management groups in your tenant.  
*Cf.*  [this doc]([https://](https://learn.microsoft.com/en-us/azure/role-based-access-control/elevate-access-global-admin?tabs=powershell%2Centra-audit-logs))

In order to achieve this, we'll have to :
1. With a Entra Global Administrator account, elevate its access to manage Azure resources
2. With this elevated access, assign the Owner role to the Super Admins group on the root management group
3. Following the best practice, remove the elevated access for the Global Admin user  


#### Step 1: Elevate Global Administrator Access
Via the Azure Portal:
1. Sign in as a Global Administrator
2. **Microsoft Entra ID** → **Properties**
3. Under **Access management for Azure resources**, toggle **"Yes"**
4. Save

This grants the Global Admin the `User Access Administrator` role at the root scope (`/`), allowing it to manage all Azure subscriptions and Management Groups.

#### Step 2: Assign Owner Role to Super Admins Group

Once elevated access is active :
```powershell

$rootMG = Get-AzManagementGroup | Where-Object { $_.DisplayName -eq "Tenant Root Group" }

$superAdminsGroup = Get-AzADGroup -DisplayName "Super Admins"

# Assign Owner role
New-AzRoleAssignment `
    -ObjectId $superAdminsGroup.Id `
    -RoleDefinitionName "Owner" `
    -Scope $rootMG.Id
```

Checking :
```powershell
Get-AzRoleAssignment -Scope $rootMG.Id | Where-Object { $_.DisplayName -eq "Super Admins" } | Select-Object DisplayName, RoleDefinitionName, Scope


DisplayName  RoleDefinitionName Scope
-----------  ------------------ -----
Super Admins Owner              /providers/Microsoft.Management/managementGroups/xxx-xxx-xxx
```
#### Step 3 : Remove Elevated Access
Via the Azure Portal:
1. Sign in as a Global Administrator
2. **Microsoft Entra ID** → **Properties**
3. Under **Access management for Azure resources**, toggle **"No"**
4. Save

Or delete the "User Access Administrator" role assignment in the Tenant Root Group's IAM management blade.

<details>
<summary><b>⚠️ Security Consideration: Permanent Owner Assignment</b></summary>
<br>

**Lab approach:** The "Super Admins" group now has **permanent** Owner access on the Root Management Group. This simplifies the lab and demonstrates RBAC concepts clearly.

**Production reality:** This approach is **NOT recommended** for production environments. Here's why:

| Risk | Impact |
|------|--------|
| **Compromised account** | Full tenant takeover possible |
| **No time limitation** | Malicious actor has unlimited access window |
| **Standing privileges** | Violates least-privilege principle |
| **Limited audit trail** | Hard to distinguish legitimate from suspicious activity |

**Production alternative: Azure PIM (Privileged Identity Management)**

Instead of permanent assignments, use **just-in-time access**:
- Owner role is **eligible**, not **active**
- User must **request activation** (e.g., for 8 hours)
- Requires **justification** and potentially **approval**
- All actions are **fully audited**
- Role **automatically expires**
</details>

<br>

---

> -> ***Subscriptions Manager users have Management Group Contributor role on the Root Management Group***

**Objective :** The "Subscriptions Manager" group can create and move subscriptions between Management Groups, but cannot manage resources within them.

**Challenge :** This requires permissions from TWO separate systems in Azure :
1. Billing permissions 
2. RBAC permissions

#### Step 1. Billing Permissions (for subscription creation)

Via the Azure Portal :
1. Sign in as a Global Administrator or Billing Account Owner
2. Navigate to **Cost Management + Billing** → **Billing Account** → **Billing Profiles** → select the Billing Profile → **Invoice Sections**  
3. From there, select an Invoice Section → **Access control (IAM)**  
4. Then, assign the ***Azure subscription creator*** role to the group.


#### Step 2. RBAC Permissions (for subscription management)
```powershell
$rootMG = Get-AzManagementGroup | ?{$_.DisplayName -eq "Tenant Root Group"}
$subsManagersGroup = Get-AzADGroup -DisplayName "Subscriptions Managers"

New-AzRoleAssignment `
    -ObjectId $subsManagersGroup.Id `
    -RoleDefinitionName "Management Group Contributor" `
    -Scope $rootMG.Id
```

<details>
<summary>Output </summary> 

```
RoleAssignmentName : xxx-xxx-xxx
RoleAssignmentId   : /providers/Microsoft.Management/managementGroups/xxx-xxx-xxx/providers/Mi
                     crosoft.Authorization/roleAssignments/xxx-xxx-xxx
Scope              : /providers/Microsoft.Management/managementGroups/xxx-xxx-xxx
DisplayName        : Subscriptions Managers
SignInName         :
RoleDefinitionName : Management Group Contributor
RoleDefinitionId   : xxx-xxx-xxx
ObjectId           : xxx-xxx-xxx
ObjectType         : Group
CanDelegate        : False
Description        :
ConditionVersion   :
Condition          :
```  
</details>
<br>

**Result:** Members can move subscriptions between Management Groups and manage the MG hierarchy.

<details>
<summary><b>🔑 Key Takeaway: Billing ≠ RBAC</b></summary>
<br>

**Billing** controls WHO can create subscriptions (and where costs are billed)  
**RBAC** controls WHAT can be done with subscriptions once they exist

Both are required for complete subscription lifecycle management. This separation is a fundamental Azure concept often overlooked but critical for proper governance.

| Permission Type | Role | Purpose | Scope |
|----------------|------|---------|-------|
| **Billing** | Azure subscription creator | Create new subscriptions | Invoice Section |
| **RBAC** | Management Group Contributor | Move & manage subscriptions | Management Group |

</details>

---

### Team A and Team B


> -> ***Team A users have Contributor role on Team A Management Group***  
> -> ***Team B users have Contributor role on Team B Management Group***

**Objective:** Members of each team can create and manage resources within their respective Management Group scope, but cannot:
- Manage RBAC permissions (requires Owner)
- Move subscriptions between Management Groups (requires Management Group Contributor)
- Access resources in the other team's Management Group  

```powershell
$teamAMG = Get-AzManagementGroup -GroupName "Team_A"
$teamBMG = Get-AzManagementGroup -GroupName "Team_B"

$teamAGroup = Get-AzADGroup -DisplayName "Team A"
$teamBGroup = Get-AzADGroup -DisplayName "Team B"

# Assign Contributor role to Team A
New-AzRoleAssignment `
    -ObjectId $teamAGroup.Id `
    -RoleDefinitionName "Contributor" `
    -Scope $teamAMG.Id

# Assign Contributor role to Team B
New-AzRoleAssignment `
    -ObjectId $teamBGroup.Id `
    -RoleDefinitionName "Contributor" `
    -Scope $teamBMG.Id
```

<details>
<summary>Output </summary>

```
RoleAssignmentName : xxx-xxx-xxx
RoleAssignmentId   : /providers/Microsoft.Management/managementGroups/Team_A/providers/Microsoft.Authorization/roleAssignments/xxx-xxx-xxx
Scope              : /providers/Microsoft.Management/managementGroups/Team_A
DisplayName        : Team A
SignInName         :
RoleDefinitionName : Contributor
RoleDefinitionId   : xxx-xxx-xxx
ObjectId           : xxx-xxx-xxx
ObjectType         : Group
CanDelegate        : False
Description        :
ConditionVersion   :
Condition          :


RoleAssignmentName : xxx-xxx-xxx
RoleAssignmentId   : /providers/Microsoft.Management/managementGroups/Team_B/providers/Microsoft.Authorization/roleAssignments/xxx-xxx-xxx
Scope              : /providers/Microsoft.Management/managementGroups/Team_B
DisplayName        : Team B
SignInName         :
RoleDefinitionName : Contributor
RoleDefinitionId   : xxx-xxx-xxx
ObjectId           : xxx-xxx-xxx
ObjectType         : Group
CanDelegate        : False
Description        :
ConditionVersion   :
Condition          :
```
</details>

<br>

**Result:** Each team can now create and manage resources (VMs, storage accounts, networks, etc.) within their Management Group, but permissions stop at their boundary.



| Group | Scope | Role | Can Create Subs | Can Move Subs | Can Manage Resources | Can Assign RBAC |
|-------|-------|------|-----------------|---------------|----------------------|-----------------|
| **Super Admins** | Root MG | Owner | ❌ | ✅ | ✅ | ✅ |
| **Sub Managers** | Root MG | MG Contributor | ✅ | ✅ | ❌ | ❌ |
| **Team A** | Team A MG | Contributor | ❌ | ❌ | ✅ | ❌ |
| **Team B** | Team B MG | Contributor | ❌ | ❌ | ✅ | ❌ |
