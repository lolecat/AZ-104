# 1. Storage Account Creation & Configuration

**Lab instructions reminder:**  
Create and configure a Storage Account in the ***Team A - Dev*** and ***Team B - Dev*** subscriptions following enterprise compliance requirements :

* Data must be replicated to another region for redundancy (minimize costs)
* Block public access to all blobs or containers
* Disable shared key access
* Allow HTTPS traffic only
* Minimum TLS version: 1.2
* Hot tier as the default access tier

<br>

Cmdlets used in this lab:
- [New-AzStorageAccount](https://learn.microsoft.com/en-us/powershell/module/az.storage/new-azstorageaccount)
- [Get-AzStorageAccount](https://learn.microsoft.com/en-us/powershell/module/az.storage/get-azstorageaccount)
- [Set-AzStorageAccount](https://learn.microsoft.com/en-us/powershell/module/az.storage/set-azstorageaccount)
- [Get-AzStorageAccountKey](https://learn.microsoft.com/en-us/powershell/module/az.storage/get-azstorageaccountkey)

Useful documentations :
- [Create a storage account](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create)

<br>

## Concepts: Storage Account Types

Before creating a storage account, understand the available types:

| Type | Performance | Use Cases | Supported Services |
|------|-------------|-----------|-------------------|
| **Standard general-purpose v2** | Standard | Most scenarios, recommended default | Blob, File, Queue, Table |
| **Premium block blobs** | Premium | High transaction rates, low latency | Block blobs, Append blobs |
| **Premium file shares** | Premium | Enterprise file shares, high IOPS | Azure Files only |
| **Premium page blobs** | Premium | VM disks, databases | Page blobs only |

### Redundancy Options (Quick Reference)

| Option | Copies | Regions | Read Access to Secondary |
|--------|--------|---------|-------------------------|
| **LRS** | 3 | 1 (single datacenter) | No |
| **ZRS** | 3 | 1 (across zones) | No |
| **GRS** | 6 | 2 (LRS primary + LRS secondary) | No (failover required) |
| **RA-GRS** | 6 | 2 | Yes |
| **GZRS** | 6 | 2 (ZRS primary + LRS secondary) | No (failover required) |
| **RA-GZRS** | 6 | 2 | Yes |

<br>

## 1. Team A - Dev storage account : Azure Portal

### Step 1: Navigate to Storage Accounts

1. Sign in to [Azure Portal](https://portal.azure.com)
2. In the search bar, type **"Storage accounts"** and select it
3. Click **"+ Create"**

### Step 2: Configure Basics Tab

| Setting | Value | Explanation |
|---------|-------|-------------|
| **Subscription** | Team A - Dev | Use the subscription created in Domain 1 |
| **Resource group** | Click "Create new" → `rg-storage-lab` | Organize resources |
| **Storage account name** | `teamadevstorage01` | 3-24 chars, lowercase letters and numbers only, globally unique |
| **Region** | France Central | Should match Team A's allowed region (from Policy lab) |
| **Performance** | Standard | For general workloads |
| **Redundancy** | Locally redundant storage (LRS) | 3 replicas in the same datacenter |

> **Naming Convention:**
> `teama` (team) + `dev` (environment) + `storage` (purpose) + `01` (count)  
> note that Azure storage account names must be globally unique across all Azure customers

### Step 3: Configure Advanced Tab

| Setting | Value | Explanation |
|---------|-------|-------------|
| **Require secure transfer for REST API** | Enabled (checked) | HTTPS only |
| **Allow enabling anonymous access on containers** | Disabled (unchecked) | Block public blob access |
| **Enable storage account key access** | Disabled (unchecked) | Force Azure AD authentication |
| **Default to Microsoft Entra authorization** | Enabled (checked) | Use Azure AD by default |
| **Minimum TLS version** | Version 1.2 | Security requirement |
| **Permitted scope for copy operations** | From storage accounts in the same Entra ID tenant | Restrict cross-tenant copies |
| **Access Tier** | Hot | Set the default access tier for the storage account |

> **Security Settings:**
> - Disabling shared key access means only Azure AD authentication works
> - This is more secure but requires proper RBAC setup

### Step 4: Configure Networking Tab

| Setting | Value | Explanation |
|---------|-------|-------------|
| **Network access** | Enable public access from all networks | We'll restrict this in Lab 7 |
| **Routing preference** | Microsoft network routing | Better performance |

### Step 5: Configure Data Protection Tab

| Setting | Value | Explanation |
|---------|-------|-------------|
| **Enable point-in-time restore for containers** | Unchecked | Requires versioning and soft delete |
| **Enable soft delete for blobs** | Checked, 7 days | Recovery option |
| **Enable soft delete for containers** | Checked, 7 days | Recovery option |
| **Enable soft delete for file shares** | Checked, 7 days | Recovery option |
| **Enable versioning for blobs** | Unchecked | Will enable in Lab 8 |
| **Enable blob change feed** | Unchecked | For audit scenarios |

### Step 6: Configure Encryption Tab

| Setting | Value | Explanation |
|---------|-------|-------------|
| **Encryption type** | Microsoft-managed keys (MMK) | Default, simplest option |
| **Enable support for CMK** | Blobs and Files only | We'll configure CMK in Lab 7 |
| **Enable infrastructure encryption** | Unchecked | Double encryption, for high security needs |


### Step 7: Review + Create

1. Review all settings
2. Click **"Create"**

<br>

## 2. Team B - Dev storage account : Powershell

Here we'll create the same type of storage account, but in the ***Team B - Dev*** subscription (don't forget that the only authorized location is  "*switzerlandnorth*" 😉) :

```powershell
Connect-AzAccount
Set-AzContext -Subscription "Team B - Dev"

# Create the "rg-storage-lab" resource group
New-AzResourceGroup -Name "rg-storage-lab" -Location "switzerlandnorth"

# Create Storage Account with compliance settings
$storageParams = @{
    ResourceGroupName           = "rg-storage-lab"
    Name                        = "teambdevstorage01"
    Location                    = "switzerlandnorth"
    SkuName                     = "Standard_LRS"          
    Kind                        = "StorageV2" 
    AccessTier                  = "Hot"
    MinimumTlsVersion           = "TLS1_2"
    EnableHttpsTrafficOnly      = $true   
    AllowBlobPublicAccess       = $false 
    AllowSharedKeyAccess        = $false 
}
$storageAccount = New-AzStorageAccount @storageParams
```

And ... It might not work depending on the subscription configuration :)
In my case, I got a "*New-AzStorageAccount: Subscription xxxx-xxxx-xxxx was not found.*"  
Why ? Simply because the `Microsoft.Storage` provider was not registered for my subscription.  
To register it, it's pretty straightforward :  

```powershell
# Check the provider state (set your AzContext in the correct subscription) :
Get-AzResourceProvider -ProviderNamespace Microsoft.Storage | Select-Object ProviderNamespace, RegistrationState

   ProviderNamespace : Microsoft.Storage
   RegistrationState : NotRegistered
   [...]

# Register the provider :
Register-AzResourceProvider -ProviderNamespace Microsoft.Storage

   ProviderNamespace : Microsoft.Storage
   RegistrationState : Registering
   ResourceTypes     : {locations/ActionsRPOperationStatuses,  storageAccountsreports, storageAccounts/storageTaskAssignments,storageAccounts/   storageTaskAssignments/reports…}
   Locations         : {Canada Central, France Central, West Europe, West US 2…}
```

Shoud be better now :)

```powershell
$storageAccount = New-AzStorageAccount @storageParams

$storageAccount | fl StorageAccountName, ResourceGroupName, PrimaryLocation, Kind, AccessTier, ProvisioningState, AllowBlobPublicAccess, MinimumTlsVersion, AllowedSharedKeyAccess

   StorageAccountName      : teambdevstorage01
   ResourceGroupName       : rg-storage-lab
   PrimaryLocation         : switzerlandnorth
   Kind                    : StorageV2
   AccessTier              : Hot
   ProvisioningState       : Succeeded
   AllowBlobPublicAccess   : False
   MinimumTlsVersion       : TLS1_2
   AllowSharedKeyAccess    : False
```
<br>

## 3. Changing the default access tier

We created our storage accounts with the Hot access tier as default value.  
Let's see how we can simply change the default access tier via Powershell.  

> **Note:** We'll switch from Hot → Cool → Hot without uploading any data.  
This demonstrates tier changes without incurring early deletion charges. The Cool tier has a 30-day minimum retention policy that only applies to stored blobs, and not the empty account itself.  
If you delete or move blobs from Cool storage before 30 days, Azure charges for the remaining days.

```powershell
# Switch to cool tier for teamadevstorage01 (don't forget to set your AzContext in the correct subscription)
Set-AzStorageAccount -ResourceGroupName "rg-storage-lab" `
                     -Name "teamadevstorage01" `
                     -AccessTier "Cool"

# Verify
Get-AzStorageAccount -ResourceGroupName "rg-storage-lab" -Name "teamadevstorage01" | Select-Object StorageAccountName, AccessTier

# Switch back to Hot 
Set-AzStorageAccount -ResourceGroupName "rg-storage-lab" `
                     -Name "teambdevstorage01" `
                     -AccessTier "Hot"
```

<br>

## 4. Storage Account Endpoints

Each storage account has unique endpoints for each service:

| Service | Endpoint Format |
|---------|----------------|
| **Blob** | `https://<account>.blob.core.windows.net` |
| **File** | `https://<account>.file.core.windows.net` |
| **Queue** | `https://<account>.queue.core.windows.net` |
| **Table** | `https://<account>.table.core.windows.net` |
| **Data Lake** | `https://<account>.dfs.core.windows.net` |

### View Endpoints in Portal

1. Go to your storage account
2. Click **"Settings"** → **"Endpoints"**

### View Endpoints via PowerShell

```powershell
(Get-AzStorageAccount -ResourceGroupName "rg-storage-lab" -Name "teambdevstorage01").PrimaryEndpoints
```

Output :
```
Blob               : https://teambdevstorage01.blob.core.windows.net/
Queue              : https://teambdevstorage01.queue.core.windows.net/
Table              : https://teambdevstorage01.table.core.windows.net/
File               : https://teambdevstorage01.file.core.windows.net/
Web                : https://teambdevstorage01.z1.web.core.windows.net/
Dfs                : https://teambdevstorage01.dfs.core.windows.net/
```
<br>

## 5. Access Keys and Connection Strings

> **Note:** Since we disabled shared key access, this section is for educational purposes. In production with shared key disabled, use Azure AD authentication instead.

Each storage account has **two access keys** (key1 and key2):
- Both keys provide full access to the storage account
- Having two keys allows key rotation without downtime
- Rotate key1 → update apps to use key2 → rotate key2

### View Access Keys

Via Azure Portal:  
Go to the storage account -> **"Security + networking"** → **"Access keys"**

Via PowerShell:
```powershell
Get-AzStorageAccountKey -ResourceGroupName "rg-storage-lab" -Name "teambdevstorage1"

KeyName      : key1
Value        : V2hhdCBkaWQgeW91IGV4cGVjdCA/IENsZWFyIGtleSA/IDopCg==
Permissions  : Full
CreationTime : 14/01/2026 04:07:46

KeyName      : key2
Value        : V2hhdCBkaWQgeW91IGV4cGVjdCA/IENsZWFyIGtleSA/IDopCg==
Permissions  : Full
CreationTime : 14/01/2026 04:07:46
```

### Connection String Format

```
DefaultEndpointsProtocol=https;
AccountName=<account-name>;
AccountKey=<account-key>;
EndpointSuffix=core.windows.net
```