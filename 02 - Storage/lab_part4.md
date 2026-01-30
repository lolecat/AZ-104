# 4. Access Tiers & Lifecycle Management

**Lab instructions reminder :**  
Optimize storage costs with access tiers and automated lifecycle policies:

1. Understand Hot, Cool, Cold, and Archive tiers and their cost implications
2. Change blob access tiers manually on existing blobs in ***teamadevstorage01***
   - Move `img1.png` from Hot → Cool
   - Move `img2.webp` from Hot → Archive
3. Create lifecycle management policies for automatic tiering
   - Rule 1: Move blobs in ***logs*** container to Cool after 30 days
   - Rule 2: Move blobs with tag `confidentiality=public` to Archive after 90 days
   - Rule 3: Delete blobs in ***logs*** container after 365 days
4. Rehydrate a blob from Archive tier
   - Rehydrate `img2.webp` with Standard priority
   - Understand High priority vs Standard priority rehydration

<br>

Cmdlets used in this lab :
- [Get-AzStorageBlob](https://learn.microsoft.com/en-us/powershell/module/az.storage/get-azstorageblob)
- [Set-AzStorageAccountManagementPolicy](https://learn.microsoft.com/en-us/powershell/module/az.storage/set-azstorageaccountmanagementpolicy)
- [Get-AzStorageAccountManagementPolicy](https://learn.microsoft.com/en-us/powershell/module/az.storage/get-azstorageaccountmanagementpolicy)
- [Copy-AzStorageBlob](https://learn.microsoft.com/en-us/powershell/module/az.storage/copy-azstorageblob)

Useful documentations :
- [Access tiers overview](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview)
- [Set a blob's access tier](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-online-manage)
- [Lifecycle management policies](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview)
- [Rehydrate an archived blob](https://learn.microsoft.com/en-us/azure/storage/blobs/archive-rehydrate-overview)

<br>

## 1. Understanding access tiers

Azure Blob Storage offers four access tiers to optimize costs based on how frequently data is accessed :

| Tier | Storage Cost | Access Cost | Min Retention | Access Latency | Use Case |
|------|--------------|-------------|---------------|----------------|----------|
| **Hot** | Highest | Lowest | None | Milliseconds | Frequently accessed data |
| **Cool** | Lower | Higher | 30 days | Milliseconds | Infrequent access, short-term backup |
| **Cold** | Even lower | Even higher | 90 days | Milliseconds | Rarely accessed, long-term backup |
| **Archive** | Lowest | Highest | 180 days | Hours | Long-term retention, compliance |

The trade-off is simple : the less you pay for storage, the more you pay to access the data. 

<br>

### Early deletion charges

If you delete or move a blob before the minimum retention period, Azure charges you for the remaining days :

| Tier | Minimum Retention | Early Deletion Penalty |
|------|-------------------|------------------------|
| Cool | 30 days | Prorated charge for remaining days |
| Cold | 90 days | Prorated charge for remaining days |
| Archive | 180 days | Prorated charge for remaining days |

> **Example :** If you move a blob from Cool to Hot after only 10 days, you'll be charged for 20 additional days of Cool storage.

<br>

### Account default vs blob-level tier

There are two levels of tier configuration :

| Level | Scope | How It Works |
|-------|-------|--------------|
| **Account default** | All new blobs | Set during storage account creation or via settings |
| **Blob-level** | Individual blob | Overrides account default, set during upload or changed later |

> **Note :** In Lab 1, we set the account default tier to **Hot**. All blobs we uploaded in Lab 3 inherited this tier. Now we'll change individual blob tiers.

<br>

## 2. Changing blob access tiers manually

Let's change the tier of the blobs we uploaded in Lab 3. We'll move `img1.png` to Cool and `img2.webp` to Archive.

### 2.1 Change tier via Azure Portal

1. Go to `teamadevstorage01` storage account
2. **Data storage** → **Containers** → **images**
3. Click on `img1.png`
4. Click **"Change tier"** (in the top menu)
5. Select **Cool** from the dropdown
6. **"Save"**

Repeat for `img2.webp` but select **Archive** :

1. Click on `img2.webp`
2. Click **"Change tier"**
3. Select **Archive**
4. **"Save"**

> **Warning :** Once a blob is in Archive tier, you cannot read or download it directly. You must first rehydrate it (section 4).

<br>

### 2.2 Change tier via PowerShell

```powershell
Set-AzContext -Subscription "Team A - Dev"
$context = New-AzStorageContext -StorageAccountName "teamadevstorage01" -UseConnectedAccount

# Move img1.png to Cool tier
$blob = Get-AzStorageBlob -Container "images" -Blob "img1.png" -Context $context
$blob.BlobClient.SetAccessTier("Cool")
```

<details>
<summary>Output</summary>

```
Status          : 200
ReasonPhrase    : OK
ContentStream   : System.IO.MemoryStream
ClientRequestId : xxxx-xxxx-xxxx
Headers         : {Server:Windows-Azure-Blob/1.0,Microsoft-HTTPAPI/2.0, x-ms-request-id:83bb9341-301e-00aa-6eff-8ce3fb000000, 
                  x-ms-client-request-id:569a5481-0470-4132-915d-65190b54adea, x-ms-version:2025-07-05…}
Content         : 
IsError         : False
```
</details>

<br>

```powershell
# Move img2.webp to Archive tier
$blob = Get-AzStorageBlob -Container "images" -Blob "img2.webp" -Context $context
$blob.BlobClient.SetAccessTier("Archive")
```

<br>

### 2.3 Verify tier changes

```powershell
Get-AzStorageBlob -Container "images" -Context $context | Select-Object Name, AccessTier, Length
```

<details>
<summary>Output</summary>

```
Name          AccessTier  Length
----          ----------  ------
img1.png      Cool        963071
img2.webp     Archive     164300
```
</details>

<br>

**Note :** Changing tiers is instant for Hot ↔ Cool ↔ Cold transitions. However, moving to Archive is also instant, but accessing Archive data requires rehydration which can take hours.

<br>

## 3. Lifecycle management policies

Lifecycle management lets us automate tier transitions and deletions based on rules. Since nobody wants to manually manage thousands of blobs, this is essential for cost optimization at scale :)

### 3.1 Understanding lifecycle rules

A lifecycle policy consists of rules, each containing :

| Component | Description |
|-----------|-------------|
| **Name** | Unique identifier for the rule |
| **Enabled** | Toggle rule on/off |
| **Filter** | Which blobs the rule applies to (prefix, blob type, tags) |
| **Actions** | What to do (tier change, delete) and when (days since creation/modification) |

<br>

### 3.2 Create lifecycle policy via Azure Portal

Let's create our three rules :
- Move blobs in `logs` container to Cool after 30 days
- Move blobs with tag `confidentiality=public` to Archive after 90 days
- Delete blobs in `logs` container after 365 days

Navigate to Lifecycle Management :
1. Go to `teamadevstorage01` storage account
2. **Data management** → **Lifecycle management**
3. Click **"+ Add rule"**

<br>

#### Rule 1 : logs → Cool after 30 days

**Details tab :**

| Setting | Value |
|---------|-------|
| Rule name | `move logs to cool` |
| Rule scope | Limit blobs with filters |
| Blob type | Block blobs |
| Blob subtype | Base blobs |

  
**Base blobs tab :**

| Condition | Action |
|-----------|--------|
| Last modified | More than 30 days ago → Move to Cool storage |

Click **"Add"** to create the rule.

  
**Filter set tab :**

| Setting | Value |
|---------|-------|
| Blob prefix | `logs/` |

> **Note :** The prefix `logs/` means this rule applies to all blobs in the `logs` container.

<br>

#### Rule 2 : Archive after 90 days when confidentiality = public

1. Click **"+ Add rule"**

**Details tab :**

| Setting | Value |
|---------|-------|
| Rule name | `archive public content` |
| Rule scope | Limit blobs with filters |
| Blob type | Block blobs |
| Blob subtype | Base blobs |

**Base blobs tab :**

| Condition | Action |
|-----------|--------|
| Last modified | More than 90 days ago → Move to Archive storage |

Click **"Add"** to create the rule.

**Filter set tab :**

| Setting | Value |
|---------|-------|
| Blob index match | Key: `confidentiality`, Operator: `==`, Value: `public` |

<br>

#### Rule 3 : Delete logs after 365 days

1. Click **"+ Add rule"**

**Details tab :**

| Setting | Value |
|---------|-------|
| Rule name | `delete old logs` |
| Rule scope | Limit blobs with filters |
| Blob type | Block blobs |
| Blob subtype | Base blobs |

**Filter set tab :**

| Setting | Value |
|---------|-------|
| Blob prefix | `logs/` |

**Base blobs tab :**

| Condition | Action |
|-----------|--------|
| Last modified | More than 365 days ago → Delete the blob |

Click **"Add"** to create the rule.

<br>

### 3.3 Create lifecycle policy via PowerShell

Here's how to create the same rules via PowerShell. This is more efficient when you need to apply the same policy across multiple storage accounts.

```powershell
Set-AzContext -Subscription "Team A - Dev"

# Rule 1: Move logs to Cool after 30 days
$rule1 = @{
    Name    = "move-logs-to-cool"
    Enabled = $true
    Definition = @{
        Filters = @{
            PrefixMatch = @("logs/")
            BlobTypes   = @("blockBlob")
        }
        Actions = @{
            BaseBlob = @{
                TierToCool = @{ DaysAfterModificationGreaterThan = 30 }
            }
        }
    }
}

# Rule 2: Archive public content after 90 days
$rule2 = @{
    Name    = "archive-public-content"
    Enabled = $true
    Definition = @{
        Filters = @{
            BlobTypes       = @("blockBlob")
            BlobIndexMatch  = @(
                @{ Name = "confidentiality"; Op = "=="; Value = "public" }
            )
        }
        Actions = @{
            BaseBlob = @{
                TierToArchive = @{ DaysAfterModificationGreaterThan = 90 }
            }
        }
    }
}

# Rule 3: Delete old logs after 365 days
$rule3 = @{
    Name    = "delete-old-logs"
    Enabled = $true
    Definition = @{
        Filters = @{
            PrefixMatch = @("logs/")
            BlobTypes   = @("blockBlob")
        }
        Actions = @{
            BaseBlob = @{
                Delete = @{ DaysAfterModificationGreaterThan = 365 }
            }
        }
    }
}

# Combine rules into policy
$policy = @{
    Rules = @($rule1, $rule2, $rule3)
}

Set-AzStorageAccountManagementPolicy -ResourceGroupName "rg-storage-lab" -StorageAccountName "teamadevstorage01" -Policy $policy
```

<details>
<summary>Output</summary>

```
ResourceGroupName  : rg-storage-lab
StorageAccountName : teamadevstorage01
Id                 : /subscriptions/xxxx-xxxx-xxxx/resourceGroups/rg-storage-lab/providers/Micros
                     oft.Storage/storageAccounts/teamadevstorage01/managementPolicies/default
Type               : Microsoft.Storage/storageAccounts/managementPolicies
LastModifiedTime   : 27/01/2026 07:28:31
Rules              : [
                       {
                         "Enabled": true,
                         "Name": "move-logs-to-cool",
                         "Definition": {
                           "Actions": {
                             "BaseBlob": {
                               "TierToCool": {
                                 "DaysAfterModificationGreaterThan": 30,
                                 "DaysAfterLastAccessTimeGreaterThan": null,
                                 "DaysAfterCreationGreaterThan": null,
                                 "DaysAfterLastTierChangeGreaterThan": null
                               },
                               "TierToArchive": null,
                               "Delete": null,
                               "TierToCold": null,
                               "TierToHot": null,
                               "EnableAutoTierToHotFromCool": null
                             },
                             "Snapshot": null,
                             "Version": null
                           },
                           "Filters": {
                             "PrefixMatch": [
                               "logs/"
                             ],
                             "BlobTypes": [
                               "blockBlob"
                             ],
                             "BlobIndexMatch": null
                           }
                         }
                       },
                       {
                         "Enabled": true,
                         "Name": "archive-public-content",
                         "Definition": {
                           "Actions": {
                             "BaseBlob": {
                               "TierToCool": null,
                               "TierToArchive": {
                                 "DaysAfterModificationGreaterThan": 90,
                                 "DaysAfterLastAccessTimeGreaterThan": null,
                                 "DaysAfterCreationGreaterThan": null,
                                 "DaysAfterLastTierChangeGreaterThan": null
                               },
                               "Delete": null,
                               "TierToCold": null,
                               "TierToHot": null,
                               "EnableAutoTierToHotFromCool": null
                             },
                             "Snapshot": null,
                             "Version": null
                           },
                           "Filters": {
                             "PrefixMatch": null,
                             "BlobTypes": [
                               "blockBlob"
                             ],
                             "BlobIndexMatch": [
                               {
                                 "Name": "confidentiality",
                                 "Op": "==",
                                 "Value": "public"
                               }
                             ]
                           }
                         }
                       },
                       {
                         "Enabled": true,
                         "Name": "delete-old-logs",
                         "Definition": {
                           "Actions": {
                             "BaseBlob": {
                               "TierToCool": null,
                               "TierToArchive": null,
                               "Delete": {
                                 "DaysAfterModificationGreaterThan": 365,
                                 "DaysAfterLastAccessTimeGreaterThan": null,
                                 "DaysAfterCreationGreaterThan": null,
                                 "DaysAfterLastTierChangeGreaterThan": null
                               },
                               "TierToCold": null,
                               "TierToHot": null,
                               "EnableAutoTierToHotFromCool": null
                             },
                             "Snapshot": null,
                             "Version": null
                           },
                           "Filters": {
                             "PrefixMatch": [
                               "logs/"
                             ],
                             "BlobTypes": [
                               "blockBlob"
                             ],
                             "BlobIndexMatch": null
                           }
                         }
                       }
                     ]
```
</details>

<br>

### 3.4 Verify lifecycle policy

Via Portal :
1. Go to **Data management** → **Lifecycle management**
2. We should see all three rules listed

Via PowerShell :
```powershell
Get-AzStorageAccountManagementPolicy -ResourceGroupName "rg-storage-lab" -StorageAccountName "teamadevstorage01"
```

<details>
<summary>Output</summary>

Same as the previous one :)
</details>

<br>

> **Important :** Lifecycle policies run once per day. Don't expect immediate results - Azure evaluates rules every 24 hours and processes eligible blobs in batches.

<br>

## 4. Rehydrating Blobs from Archive Tier

Remember `img2.webp` we moved to Archive ? Let's bring it back ! Blobs in Archive tier are offline and cannot be read directly. Rehydration copies the blob to an online tier.  
Rehydration is a background process. Plan ahead when working with Archive tier, if you need data quickly, consider keeping a copy in Cool tier or use High priority rehydration for emergencies.

### 4.1 Understanding Rehydration Options

| Priority | Time to Complete | Cost | Use Case |
|----------|-----------------|------|----------|
| **Standard** | Up to 15 hours | Lower | Normal operations, non-urgent |
| **High** | Under 1 hour for blobs < 10 GB | Higher | Urgent data recovery |

> **Note :** High priority is significantly more expensive, use it only when you really need the data quickly

<br>

### 4.2 Rehydrate via Azure Portal

1. Go to `teamadevstorage01` → **Containers** → **images**
2. Click on `img2.webp`
3. Click **"Change tier"**
4. Select **Hot** (or Cool/Cold) as the target tier
5. Set **Rehydrate priority** to **Standard**
6. Click **"Save"**

You'll notice the blob's access tier now shows **Archive (rehydration pending to Hot)**. This indicates the rehydration is in progress.

<br>

### 4.3 Rehydrate via PowerShell

There are two ways to rehydrate a blob :

**Method 1 : Set tier directly (changes in-place)**
```powershell
$context = New-AzStorageContext -StorageAccountName "teamadevstorage01" -UseConnectedAccount

$blob = Get-AzStorageBlob -Container "images" -Blob "img2.webp" -Context $context
$blob.BlobClient.SetAccessTier("Hot")
```

**Method 2 : Copy the blob to a new location / new name (preserves the original)**
```powershell
# Copy archived blob to a new name in Hot tier
Copy-AzStorageBlob `
    -SrcContainer "images" `
    -SrcBlob "img2.webp" `
    -DestContainer "images" `
    -DestBlob "img2-rehydrated.webp" `
    -StandardBlobTier Hot `
    -RehydratePriority Standard `
    -Context $context
```

<br>

### 4.4 Check rehydration status

```powershell
$blob = Get-AzStorageBlob -Container "images" -Blob "img2.webp" -Context $context
$blob.BlobProperties.ArchiveStatus
```

<details>
<summary>Output (while rehydrating)</summary>

```
rehydrate-pending-to-hot
```
</details>

<br>

Once rehydration completes, `ArchiveStatus` will be empty and `AccessTier` will show the target tier.

```powershell
$blob.BlobProperties.AccessTier
```

<details>
<summary>Output (after rehydration)</summary>

```
Hot
```
</details>

<br>

## Key Takeaways

| Concept | Remember |
|---------|----------|
| **Tier selection** | Balance storage cost vs access cost based on your patterns |
| **Early deletion** | Cool (30d), Cold (90d), Archive (180d) : moving early incurs charges |
| **Lifecycle policies** | Run daily, use for automated cost optimization at scale |
| **Archive access** | Requires rehydration (hours), plan ahead ! |
| **Rehydration priority** | Standard (up to 15h, cheaper) vs High (< 1h, expensive) |