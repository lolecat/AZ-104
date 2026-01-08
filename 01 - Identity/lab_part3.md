# 3. Subscription management

**Lab instructions reminder :**

Create 2 subscriptions per Management Group :
- Dev
- Prod

```
Azure resources structure (governance)

🗂️ Root Management Group
├─ 🗂️ Team A
│  ├─ 🔑Team A - Dev
│  ├─ 🔑Team A - Prod
├─ 🗂️ Team B
│  ├─ 🔑Team B - Dev
│  ├─ 🔑Team B - Prod
``` 

<br>

Cmdlets used in this lab :
- [Get-AzSubscription](https://learn.microsoft.com/en-us/powershell/module/az.accounts/get-azsubscription)
- [New-AzManagementGroupSubscription](https://learn.microsoft.com/en-us/powershell/module/az.resources/new-azmanagementgroupsubscription)
- [Get-AzManagementGroupSubscription](https://learn.microsoft.com/en-us/powershell/module/az.resources/get-azmanagementgroupsubscription)
- [Get-AzBillingAccount](https://learn.microsoft.com/en-us/powershell/module/az.billing/get-azbillingaccount)
- [Get-AzBillingProfile](https://learn.microsoft.com/en-us/powershell/module/az.billing/get-azbillingprofile)
- [Get-AzInvoiceSection](https://learn.microsoft.com/en-us/powershell/module/az.billing/get-azinvoicesection)
- [az account alias create](https://learn.microsoft.com/en-us/cli/azure/account/alias?view=azure-cli-latest#az-account-alias-create)

Useful documentations :  
- [Programmatically create subscriptions (MCA)](https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/programmatically-create-subscription-microsoft-customer-agreement?tabs=azure-powershell)
- [Organize costs by customizing your billing account](https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/mca-section-invoice)

<br>

## 1. ~~Subscriptions creation~~ Before we begin

Remember the Lab 2 ? We forgot something. "Subscriptions managers" members can create new subscriptions, but only in the default invoice section. They can't create new ones. So first of all, we'll play again with some RBAC on the Billing Account :)  

**Objective :** Members of "Subscriptions managers" must be able to read and get informations about the Billing Account, and create new invoice sections.  
**Solution :** Give to the "Subscriptions managers" group the "Billing profile contributor" role on the Billing Profile scope.  

So that the "Billing part" of Azure would have the following structure :

```
Azure Billing structure

📊 Billing Account (Default)
 └─ 💳 Billing Profile (Default)
     ├─ 📄 Invoice Section: Team A
     │   ├─ 🔑 Team A - Prod
     │   └─ 🔑 Team A - Dev
     └─ 📄 Invoice Section: Team B
         ├─ 🔑 Team B - Prod
         └─ 🔑 Team B - Dev
```
Way more clean than dumping all our subscriptions under the same invoice section.  

#### Step 1 : Assign the "Billing Profile Contributor" role
This kind of persmission is granted only via the Azure portal or direct API calls. So like in the [lab part 2](./lab_part2.md), we must via the Azure Portal :

1. Sign in as a Global Administrator or Billing Account Owner
2. Navigate to **Cost Management + Billing** → **Billing Account** → **Billing Profiles** → select the Billing Profile
3. From there go to the **Access control (IAM)** tab
4. Then, assign the ***Billing Profile Contributor*** role to the group.

#### Step 2 : Create the invoice sections
In the Azure Portal :

1. Sign in as a member of the Subscriptions managers group
2. Navigate to **Cost Management + Billing** → normally, the portal should directly redirect us in the Billing Profile on which we have the permissions
3. From there go to **Billing** -> **Invoice sections**
4. Add the ***Team A*** and ***Team B*** Invoice sections

<br>

## 2. Subscriptions creation
Connect with an account member of the previously created Subscritions managers group.


```powershell
$ba = get-AzBillingAccount
$baProfile = Get-AzBillingProfile -BillingAccountName $ba.name
$invoice = Get-AzInvoiceSection -BillingAccountName $ba.name -BillingProfileName $baProfile.name

# If not already installed, get the "Az.Subscription" module in order to use the "New-AzSubscriptionAlias" cmdlet
Install-Module Az.Subscription

# /!\ Wont work, cause the Powershell Az.Subscription module is bugged (preview module)
$mgGroup = Get-AzManagementGroup -GroupName "Team_A"

New-AzSubscriptionAlias `
    -AliasName "team_a_dev" `
    -SubscriptionName "Team A - Dev" `
    -BillingScope $invoice.Id `
    -ManagementGroupId $mgGroup.Id `
    -Workload Devtest
# /!\ Wont work, cause the Powershell Az.Subscription module is bugged (preview module)

# Use Azure CLI instead :
az account alias create `
    --name "team_a_dev" `
    --billing-scope $invoice['Display Name' -eq 'Team A'].Id `
    --display-name "Team A - Dev" `
    --workload "DevTest"

az account alias create `
    --name "team_a_prod" `
    --billing-scope $invoice['Display Name' -eq 'Team A'].Id `
    --display-name "Team A - Prod" `
    --workload "Production"

az account alias create `
    --name "team_b_dev" `
    --billing-scope $invoice['Display Name' -eq 'Team B'].Id `
    --display-name "Team B - Dev" `
    --workload "DevTest"

az account alias create `
    --name "team_b_prod" `
    --billing-scope $invoice['Display Name' -eq 'Team B'].Id `
    --display-name "Team B - Prod" `
    --workload "Production"
```

<details>
<summary>Ouput (part of) </summary>

```
{
  "id": "/providers/Microsoft.Subscription/aliases/team_b_prod",
  "name": "team_b_prod",
  "properties": {
    "acceptOwnershipState": null,
    "acceptOwnershipUrl": null,
    "billingScope": null,
    "createdTime": null,
    "displayName": null,
    "managementGroupId": null,
    "provisioningState": "Succeeded",
    "resellerId": null,
    "subscriptionId": "xxx-xxx-xxx",
    "subscriptionOwnerId": null,
    "tags": null,
    "workload": null
  },
  "systemData": null,
  "type": "Microsoft.Subscription/aliases"
}
```
Note :  The "null" values everywhere are normal.
</details>

<br>

## 3. Move Subscriptions to Management Groups

> Why subscriptions aren't created directly in Management Groups ?

-> Subscription creation and placement are two separate operations with different permission requirements:

| Operation | System | Permission Required |
|-----------|--------|---------------------|
| **Create subscription** | Billing | Azure subscription creator (on Invoice) / Billing profile contributor (on billing Profile)  |
| **Place in MG** | RBAC | Management Group Contributor or Owner (on MG) |

All new subscriptions are initially placed in the Tenant Root Group. They must be explicitly moved to their target Management Group.



#### Move subscriptions
```powershell
$subs = Get-AzSubscription
$subsMapping = @(
    @{Name="Team A - Dev"; MG="Team_A"},
    @{Name="Team A - Prod"; MG="Team_A"},
    @{Name="Team B - Dev"; MG="Team_B"},
    @{Name="Team B - Prod"; MG="Team_B"}
)

foreach ($i in $subsMapping) `
{
    $targetMG = Get-AZManagementGroup -GroupName $i.MG
    $subId = $subs.Where({$_.Name -eq $i.Name}).Id

    try 
    {
        New-AzManagementGroupSubscription `
            -GroupName $targetMG.Name `
            -SubscriptionId $subId `
            -ErrorAction Stop
    } 
    catch {Write-Error "Failed to move: $($_.Exception.Message)"}
} 
```

<details>
<summary>Output (part of)</summary>

```
Id          : /providers/Microsoft.Management/managementGroups/Team_B/subscriptions/xxx-xxx-xxx
              c
Type        : Microsoft.Management/managementGroups/subscriptions
Tenant      : xxx-xxx-xxx
DisplayName : Team B - Dev
Parent      : /providers/Microsoft.Management/managementGroups/Team_B
State       : Active

Id          : /providers/Microsoft.Management/managementGroups/Team_B/subscriptions/xxx-xxx-xxx
              e
Type        : Microsoft.Management/managementGroups/subscriptions
Tenant      : xxx-xxx-xxx
DisplayName : Team B - Prod
Parent      : /providers/Microsoft.Management/managementGroups/Team_B
State       : Active
```
</details>

<br>

## Summary

At this point, the following architecture is in place:

**Billing Structure:**
```
📊 Billing Account
 └─ 💳 Billing Profile
     ├─ 📄 Invoice Section: Team A
     │   ├─ 🔑 Team A - Dev
     │   └─ 🔑 Team A - Prod
     └─ 📄 Invoice Section: Team B
         ├─ 🔑 Team B - Dev
         └─ 🔑 Team B - Prod
```

**Governance Structure:**
```
🗂️  Tenant Root Group
    ├─ 👥 Super Admins (Owner)
    ├─ 👥 Subscriptions Managers (MG Contributor)
    ├─ 🗂️  Team A
    │   ├─ 👥 Team A (Contributor)
    │   ├─ 🔑 Team A - Dev
    │   └─ 🔑 Team A - Prod
    └─ 🗂️  Team B
        ├─ 👥 Team B (Contributor)
        ├─ 🔑 Team B - Dev
        └─ 🔑 Team B - Prod
```