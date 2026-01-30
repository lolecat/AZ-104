# 2. Storage Redundancy & Object Replication

**Lab instructions reminder :**  
Configure storage redundancy and set up cross-region object replication :

1. Understand all redundancy options (LRS, ZRS, GRS, RA-GRS, GZRS, RA-GZRS)
2. Change redundancy on the ***teambdevstorage01*** account : switch to ***RA-GRS***
3. Create a new storage account in ***Team A - Prod*** subscription
4. Configure Object Replication from ***Team A Prod*** to ***Team A - Dev*** for backup purposes
5. Understand failover scenarios and RPO/RTO

<br>

Cmdlets used in this lab :  
- [Update-AzStorageBlobServiceProperty](https://learn.microsoft.com/en-us/powershell/module/az.storage/update-azstorageblobserviceproperty?view=azps-15.2.0)
- [New-AzStorageContext](https://learn.microsoft.com/en-us/powershell/module/az.storage/new-azstoragecontext?view=azps-15.2.0)
- [New-AzStorageContainer](https://learn.microsoft.com/en-us/powershell/module/az.storage/new-azstoragecontainer?view=azps-15.2.0)
- [New-AzStorageObjectReplicationPolicyRule](https://learn.microsoft.com/en-us/powershell/module/az.storage/new-azstorageobjectreplicationpolicyrule)
- [Set-AzStorageObjectReplicationPolicy](https://learn.microsoft.com/en-us/powershell/module/az.storage/set-azstorageobjectreplicationpolicy)
- [Get-AzStorageObjectReplicationPolicy](https://learn.microsoft.com/en-us/powershell/module/az.storage/get-azstorageobjectreplicationpolicy)



Useful documentations :
- [Azure Storage redundancy](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)
- [Object replication overview](https://learn.microsoft.com/en-us/azure/storage/blobs/object-replication-overview)
- [Enable and manage blob versioning](https://learn.microsoft.com/en-us/azure/storage/blobs/versioning-enable?tabs=powershell)
- [Initiate storage account failover](https://learn.microsoft.com/en-us/azure/storage/common/storage-initiate-account-failover)
- [Azure region pairs](https://learn.microsoft.com/en-us/azure/reliability/cross-region-replication-azure#azure-paired-regions)
- [Azure regions list](https://learn.microsoft.com/en-us/azure/reliability/regions-list)

<br>

## 1. Understand redundancy options



| Option | Full Name | Description | Durability | Use Case |
|--------|-----------|-------------|------------|----------|
| **LRS** | Locally Redundant Storage | 3 copies in single datacenter | 11 nines | Dev/test, non-critical data |
| **ZRS** | Zone-Redundant Storage | 3 copies across 3 availability zones | 12 nines | Production, zone failure protection |
| **GRS** | Geo-Redundant Storage | LRS primary + LRS secondary region | 16 nines | DR, cross-region protection |
| **RA-GRS** | Read-Access Geo-Redundant Storage | GRS + read access to secondary | 16 nines | DR with read failover |
| **GZRS** | Geo-Zone-Redundant Storage | ZRS primary + LRS secondary region | 16 nines | Maximum availability + DR |
| **RA-GZRS** | Read-Access Geo-Zone-Redundant Storage | GZRS + read access to secondary | 16 nines | Critical production workloads |

<br>

**For this lab, here are the Azure paired regions we'll use :**

| Primary Region | Paired Region |
|---------------|---------------|
| France Central | France South |
| Switzerland North | Switzerland West |

<br>

## 2. Change redundancy on the teambdevstorage01 account

### Swith to RA-GRS via Azure Portal : 

Navigate to Redundancy Settings :
1. Go to `teambdevstorage01` storage account
2. Click **"Data management"** → **"Redundancy"**


We should see :  
- Current redundancy : Locally-redundant storage (LRS)
- Location : Switzerland North
- Datacenter type : Primary

Change to RA-GRS (add copies to a region pair)  :
1. In the **Redundancy** dropdown, select **"Geo-redundant storage with read access (RA-GRS)"**
2. **"Save"**  

<br>

> Now, a read-only endpoint for the secondary region becomes available

View Secondary Endpoint :
1. After saving, go to **"Settings"** → **"Endpoints"**
2. Notice the new **Secondary** endpoints :
- Secondary endpoint - Blob: `https://teambdevstorage01-secondary.blob.core.windows.net/`
- Secondary endpoint - File: `https://teambdevstorage01.file.core.windows.net/`

<br>

### Swith back to LRS via Powershell : 

```powershell
Set-AzContext -Subscription "Team B - Dev"

$saTeamB = Get-AzStorageAccount -ResourceGroupName "rg-storage-lab" -Name "teambdevstorage01"

$saTeamB | Set-AzStorgeAccount -SkuName "Standard_LRS"
```

<details>
<summary>Output</summary>

```
StorageAccountName ResourceGroupName PrimaryLocation  SkuName      Kind      AccessTier  ProvisioningState
------------------ ----------------- ---------------  -------      ----      ----------  -----------
teambdevstorage01  rg-storage-lab    switzerlandnorth Standard_LRS StorageV2 Hot         Succeeded
```
</details>

<br>

## 3. Create a new storage account in Team A - Prod subscription

For object replication, we need:
- **Source**: A new storage account in Team A - Prod : `teamaprodstorage01`
- **Destination**: `teamadevstorage01` (Team A - Dev)

In this case, the object replication will be in a same region, and in same subscription.
Note that it's possible to setup an object replication between different regions, subscriptions, or Entra ID tenants.

Let's create this quick via Powershell, as seen in the previous lab.  
```powershell
Set-AzContext -Subscription "Team A - Prod"

New-AzResourceGroup -Name "rg-storage-lab" -Location "francecentral"

$storageParams = @{
    ResourceGroupName           = "rg-storage-lab"
    Name                        = "teamaprodstorage01"
    Location                    = "francecentral"
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

<details>
<summary>Output</summary>

```
StorageAccountName ResourceGroupName PrimaryLocation SkuName      Kind      AccessTier ProvisioningState
------------------ ----------------- --------------- -------      ----      ---------- ------------
teamaprodstorage01 rg-storage-lab    francecentral   Standard_LRS StorageV2 Hot        Succeeded
```
</details>

<br>

## 4. Configure Object Replication

Object replication asynchronously copies block blobs between storage accounts. Use cases:
- Disaster recovery
- Latency reduction (replicate to region closer to users)
- Data distribution

**Prerequisites for Object Replication :**

| Requirement | Source | Destination |
|-------------|--------|-------------|
| Blob versioning | Enabled | Enabled |
| Change feed | Enabled | Not required |
| Storage account type | General purpose v2 or Premium | General purpose v2 or Premium |

### Step 1: Settings on source and destination accounts

Enable versioning on the Dev storage account (destination) - Azure Portal method :

1. Switch to **Team A - Dev** subscription
2. Go to `teamadevstorage01`
3. **Data management** → **Data protection**
4. Check **"Enable versioning for blobs"**
5. **Save**  


Enable versioning and change feed on the Prod storage account (source) - Powershell method :
```powershell
# Switch to Team A - Prod
Set-AzContext -Subscription "Team A - Prod"

# Enable versioning and change feed
Update-AzStorageBlobServiceProperty `
    -ResourceGroupName "rg-storage-lab" `
    -StorageAccountName "teamaprodstorage01" `
    -IsVersioningEnabled $true `
    -EnableChangeFeed $true `
    -ChangeFeedRetentionInDays 7
```

<details>
<summary>Output (part of) :</summary>

```
StorageAccountName                                     : teamaprodstorage01
ResourceGroupName                                      : rg-storage-lab
DeleteRetentionPolicy.Enabled                          : False
DeleteRetentionPolicy.AllowPermanentDelete             : False
ChangeFeed.Enabled                                     : True
ChangeFeed.RetentionInDays                             : 7
IsVersioningEnabled                                    : True
```

</details>

### Step 2: Create containers on both accounts

> Object replication works at the container level.

Container creation on Dev storage account (destination) - Azure Portal method :
1. Switch to **Team A - Dev** subscription
2. Go to `teamadevstorage01`
3. **Data storage** → **Containers**
4. Click **"+ Container"**
5. Name: `dest-container`
6. Anonymous access level: **Private** (already by default, inherited from the storage account config)
7. **"Create"**

Container creation on Dev storage account (destination) - Powershell method :
```powershell
# Switch to Team A - Prod
Set-AzContext -Subscription "Team A - Prod"

# Get the context for the "teamaprodstorage01" storage account. It'll be needed for the container creation. 
$context = New-AzStorageContext -StorageAccountName "teamaprodstorage01" -UseConnectedAccount

# Create the container
New-AzStorageContainer -Name "src-container" -Context $context
```

### Step 3: Configure Object Replication Policy

Via Portal (start from source account):

1. Go to `teamaprodstorage01` (source)
2. **Data management** → **Object replication**
3. Click **"Set up replication rules"**
4. Configure:

| Setting | Value |
|---------|-------|
| Destination subscription | `Team A - Dev` |
| Destination storage account | `teamadevstorage01` |
| Source storage account | `teamadevstorage01` |
| Source container | `src-container` |
| Destination container | `dest-container` |

5. **"Create"**

> **Important:** Setting up replication from the destination account automatically configures both source and destination policies.

Via PowerShell:
```powershell
# Get storage account contexts
Set-AzContext -Subscription "Team A - Prod"
$sourceAccount = Get-AzStorageAccount -ResourceGroupName "rg-storage-lab" -Name "teamaprodstorage01"

Set-AzContext -Subscription "Team A - Dev"
$destAccount = Get-AzStorageAccount -ResourceGroupName "rg-storage-lab" -Name "teamadevstorage01"

######

# Create replication rule
$rule = New-AzStorageObjectReplicationPolicyRule `
    -SourceContainer "src-container" `
    -DestinationContainer "dest-container"


# Set policy on destination account
Set-AzContext -Subscription "Team A - Dev"

$destPolicy = Set-AzStorageObjectReplicationPolicy `
    -ResourceGroupName "rg-storage-lab" `
    -StorageAccountName "teamadevstorage01" `
    -SourceAccount $sourceAccount.Id `
    -Rule $rule


# Set policy on source account (using the policy ID from destination)
Set-AzContext -Subscription "Team A - Prod"

Set-AzStorageObjectReplicationPolicy `
    -ResourceGroupName "rg-storage-lab" `
    -StorageAccountName "teamaprodstorage01" `
    -InputObject $destPolicy
```

### Step 4: Test Object Replication

1. Upload a test file to `src-container` on `teamaprodstorage01`
2. Wait a few minutes, as the replication is asynchronous
3. Check `dest-container` on `teamadevstorage01`
4. The file should appear (may take 5-15 minutes)

```powershell
# Uploading a test.txt file to the src-container
Write-Output "Y R U GEH ?" > "C:\Users\Louis\Desktop\test.txt"

azcopy cp "C:\Users\Louis\Desktop\test.txt" "https://teamaprodstorage01.blob.core.windows.net/src-container/test.txt"

######

# After a few minutes, get the content of the dest-container
Set-AzContext -Subscription "Team A - Dev"
$destAccount = Get-AzStorageAccount -ResourceGroupName "rg-storage-lab" -Name "teamadevstorage01"

Get-AzStorageContainer -Name "dest-container" -Context $destAccount.Context | Get-AzStorageBlob | fl Name, IsLatestVersion, VersionId, LastModified

    Name            : test.txt
    IsLatestVersion : True
    VersionId       : 2026-01-22T03:07:28.5473368Z
    LastModified    : 22/01/2026 03:07:28 +00:00
```


### Step 5: View Replication Status

Via Portal :
1. Navigate to the source account in the Azure portal.
2. Locate the container that includes the source blob.
3. Select the blob to display its properties.

Via Powershell :
```powershell
$srcContext = (Get-AzStorageAccount -ResourceGroupName "rg-storage-lab" -StorageAccountName "teamaprodstorage01").Context

$blobSrc = Get-AzStorageBlob -Container "src-container" -Context $srcContext -Blob "test.txt"

$blobSrc.BlobProperties.ObjectReplicationSourceProperties[0].Rules[0].ReplicationStatus

    Complete
```

<br>

## 5. Storage Account Failover (GRS/GZRS)

For accounts with geo-redundancy, you can initiate failover to the secondary region.

> **Warning:** Failover is a significant operation that should only be used in disaster scenarios.

### Failover Concepts

| Term | Definition |
|------|------------|
| **RPO** | Recovery Point Objective - How much data loss is acceptable (GRS: up to 15 min) |
| **RTO** | Recovery Time Objective - How long to recover (Failover: ~1 hour) |
| **Last Sync Time** | When data was last synchronized to secondary |

### View Last Sync Time

Via Portal:
1. Go to storage account with GRS/GZRS
2. **Data management** → **Redundancy**
3. Note the **"Last sync time"** field

Via PowerShell:
```powershell
$sa = Get-AzStorageAccount -ResourceGroupName "rg-storage-lab" -Name "teamadevstorage01"
$sa.GeoReplicationStats
```

### Initiate Failover

Via Portal:
1. **Data management** → **Redundancy**
2. Click **"Prepare for failover"**
3. Type the storage account name to confirm
4. Click **"Failover"**

> **Warning:**
> - Failover converts GRS to LRS (you lose geo-redundancy)
> - Data written after Last Sync Time may be lost
> - Original primary becomes inaccessible