# 6. Azure Files & File Shares

**Lab instructions reminder :**  
Deploy and configure Azure Files for cloud file sharing:

1. Understand Azure Files tiers (Transaction Optimized, Hot, Cool, Premium) and protocols (SMB, NFS)
2. Create 2 file shares on ***teamaprodstorage01***
   - ***shared-documents*** : Transaction Optimized tier, 100 GiB quota
   - ***team-backups*** : Cool tier, 50 GiB quota
3. Upload files and manage directories in ***shared-documents*** via Azure Portal
4. Mount ***shared-documents*** on Windows using SMB and on Linux using SMB
5. Configure snapshots on ***shared-documents***
   - Create a manual snapshot
   - Restore a deleted file from the snapshot
6. Enable soft delete for file shares with a 14-day retention period

> **Note** : Unless specified, all commands are executed with an admin account (owner @ root management group level)

<br>

Cmdlets used in this lab :
- [New-AzStorageShare](https://learn.microsoft.com/en-us/powershell/module/az.storage/new-azstorageshare)
- [Update-AzRmStorageShare](https://learn.microsoft.com/en-us/powershell/module/az.storage/update-azrmstorageshare?view=azps-15.3.0)
- [Get-AzStorageShare](https://learn.microsoft.com/en-us/powershell/module/az.storage/get-azstorageshare)
- [New-AzStorageDirectory](https://learn.microsoft.com/en-us/powershell/module/az.storage/new-azstoragedirectory)
- [Set-AzStorageFileContent](https://learn.microsoft.com/en-us/powershell/module/az.storage/set-azstoragefilecontent)
- [Get-AzStorageFileContent](https://learn.microsoft.com/en-us/powershell/module/az.storage/get-azstoragefilecontent)
- [Restore-AzRmStorageShare](https://learn.microsoft.com/en-us/powershell/module/az.storage/restore-azrmstorageshare?view=azps-15.2.0)

Useful documentations :
- [Create an Azure file share](https://learn.microsoft.com/en-us/azure/storage/files/storage-how-to-create-file-share)
- [Overview of Azure Files identity-based authentication](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-active-directory-overview)
- [Enable Microsoft Entra Kerberos authentication](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-identity-auth-hybrid-identities-enable?tabs=azure-portal%2Cintune)
- [Overview of Azure Files authorization and access control](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-authorization-overview#configure-directory-and-file-level-permissions)
- [Mount SMB file share on Windows](https://learn.microsoft.com/en-us/azure/storage/files/storage-how-to-use-files-windows)
- [Mount SMB file share on Linux](https://learn.microsoft.com/en-us/azure/storage/files/storage-how-to-use-files-linux)
- [Azure file share snapshots](https://learn.microsoft.com/en-us/azure/storage/files/storage-snapshots-files)
- [Soft delete for Azure file shares](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-enable-soft-delete)

<br>

## 1. Understanding Azure Files

Azure Files provides fully managed file shares in the cloud, accessible via the SMB and NFS protocols. Think of it as a cloud-based file server.

### Tiers and performance

| Tier | Storage Type | Use Case | Latency | Billing Model |
|------|-------------|----------|---------|---------------|
| **Premium** | SSD (FileStorage account) | Databases, high IOPS workloads | Single-digit ms | Provisioned capacity |
| **Transaction Optimized** | HDD (General-purpose v2) | Team shares, applications | Standard | Pay-as-you-go |
| **Hot** | HDD (General-purpose v2) | General-purpose file sharing | Standard | Pay-as-you-go, lower storage cost than Txn Optimized |
| **Cool** | HDD (General-purpose v2) | Archival, infrequent access | Standard | Lowest storage cost, higher access cost |

**Difference with Blob tiers :** Azure Files tiers are set at the file share level, not at the individual file level. You can't have one file as Hot and another as Cool within the same share.

<br>

### Protocols

| Protocol | OS Support | Port | Authentication | Use Case |
|----------|-----------|------|----------------|----------|
| **SMB 3.x** | Windows, Linux, macOS | 445 | Storage key, AD DS, Entra ID | General file sharing, lift-and-shift |
| **NFS 4.1** | Linux only | 2049 | Network-based (no user auth) | Linux workloads, app data |

> **Note :** NFS is only supported on Premium file shares (FileStorage account). SMB works on all tiers.


<br>

### Azure Files vs Blob Storage


| Feature | Azure Files | Blob Storage |
|---------|-------------|--------------|
| **Access method** | SMB/NFS mount (drive letter) | REST API, SDKs, AzCopy |
| **Use case** | Replace/extend on-prem file servers | Unstructured data, media, backups |
| **Structure** | Hierarchical (real folders) | Flat namespace (virtual folders via `/`) |
| **Concurrent access** | Multiple clients via SMB | Single writer, multiple readers |
| **Max file share size** | 100 TiB | N/A (container has no size limit) |
| **Max file size** | 4 TiB | 190.7 TiB (block blob) |

<br>

## 2. Create 2 file shares on `teamaprodstorage01`

### 2.1 Via Azure Portal

Go to `teamaprodstorage01` → **Data storage** → **File shares** → **"+ File share"**

<br>

**First share :**

| Setting | Value |
|---------|-------|
| Name | `shared-documents` |
| Access tier | Transaction Optimized |
| Provisioned capacity | 100 GiB |
| Enable backup | ☐ (no) |

<br>

**Second share :**

| Setting | Value |
|---------|-------|
| Name | `team-backups` |
| Access tier | Cool |
| Provisioned capacity | 50 GiB |
| Enable backup | ☐ (no) |


<br>

### 2.2 Via PowerShell

```powershell
$context = New-AzStorageContext -StorageAccountName "teamaprodstorage01" -UseConnectedAccount

New-AzStorageShare -Name "shared-documents" -Context $context -Protocol Smb
Update-AzRmStorageShare -ResourceGroupName "rg-storage-lab" -StorageAccountName "teamaprodstorage01" -Name "shared-documents" -QuotaGiB 100 -AccessTier TransactionOptimized

New-AzStorageShare -Name "team-backups" -Context $context -Protocol Smb
Update-AzRmStorageShare -ResourceGroupName "rg-storage-lab" -StorageAccountName "teamaprodstorage01" -Name "team-backups" -QuotaGiB 50 -AccessTier Cool
```

<br>

### 2.3 Verify file shares

```powershell
# We need to pay a little bit with calculated properties to get the 'AccessTier' property and a correctly formated quota

Get-AzStorageShare -Context $context | Select-Object Name, @{Label="Quota";Expression={($_.ShareProperties.QuotaInGB).ToString()+" GiB"}}, @{Label="AccessTier";Expression={$_.ShareProperties.AccessTier}}
```
Output :

```
Name             Quota   AccessTier
----             -----   ----------
shared-documents 100 GiB TransactionOptimized
team-backups     50 GiB  Cool
```

<br>

## 3. Upload files and manage directories

Let's populate our `shared-documents` share with some structure. Here's the file structure we'll create :  

```
shared-documents/
├── projects/
│   └── notes.txt
└── templates/
    ├── more_notes.txt
    └── img1.png
```

<br>

### 3.1 Create directories via PowerShell

First, we'll create the "projects" and "templates" directories, then we'll upload the associated files into those new directories

```powershell
$share = Get-AzStorageShare -Name "shared-documents" -Context $context

$share | New-AzStorageDirectory -Path "projects" -Context $context
$share | New-AzStorageDirectory -Path "templates" -Context $context
```

<details>
<summary>But wait, you really thought it would be that simple ? :) It won't work. We have the following error :  </summary>

```
New-AzStorageDirectory: An HTTP header that's mandatory for this request is not specified.
RequestId: xxx-xxxx-xxx
Time:2026-02-21T10:27:58.2109402Z
Status: 400 (An HTTP header that's mandatory for this request is not specified.)
ErrorCode: MissingRequiredHeader

Additional Information:
HeaderName: x-ms-file-request-intent

Content:
 <?xml version="1.0" encoding="utf-8"?><Error><Code>MissingRequiredHeader</Code><Message>An HTTP header that's mandatory for this request is not specified.
RequestId: xxx-xxxx-xxx
Time:2026-02-21T10:27:58.2109402Z</Message><HeaderName>x-ms-file-request-intent</HeaderName></Error>

Headers:
Server: Windows-Azure-File/1.0,Microsoft-HTTPAPI/2.0
x-ms-request-id: xxx-xxxx-xxx
x-ms-client-request-id: xxx-xxxx-xxx
x-ms-version: 2025-07-05
x-ms-error-code: MissingRequiredHeader
Date: Sat, 21 Feb 2026 10:27:57 GMT
Content-Length: 305
Content-Type: application/xml
```
</details>
<br>

Why ? For two reasons :
- Remember when we created the `teamaprodstorage01` storage account in lab2 ? We did it with a Powershell cmdlet, and disabled the shared key access, but we did not specify if and how identities can access SMB shares.
- We did not configure any RBAC for those file shares. Remember : being a storage account Owner or Contributor only grants *control plane* access (manage the resource itself : create shares, configure settings, etc.). It does not grant *data plane* access (read/write files inside the share). For that, an explicit RBAC role scoped to the data plane is needed, such as Storage File Data SMB Share Contributor (read/write/delete files) or Storage File Data SMB Share Reader.

So the next steps are to configure the correct identity-based access for our scenario, and then to set up the right RBAC authorizations.  

#### 3.1.1 Identity-based access methods for Azure File Shares :

| Method | Identity source | Requires domain join | Best for |
|--------|-----------------|----------------------|----------|
| **Entra ID Kerberos** | Entra ID (cloud-only / hybrid users) | Cloud only users : no | Cloud-native environments, hybrid users synced to Entra ID |
| **Entra Domain Services** | Entra ID managed domain | Yes (to AADDS) | Lift-and-shift without on-prem AD |
| **On-premises AD DS** | Active Directory Domain Services | Yes (to on-prem AD) | Traditional enterprise with on-prem AD |
| **None (storage key)** | N/A | No | Dev/test, automation — not recommended for production |

> **For this scenario :** We'll use Entra ID Kerberos. Our storage account is cloud-native, no on-prem AD, and users/groups (Super Admins, Team A Prod) are Entra ID objects. This is the simplest and most modern approach.

Enabling the "Entra ID Kerberos" access method for `teamaprodstorage01` :
```powershell
Set-AzStorageAccount `
    -ResourceGroupName "rg-storage-lab" `
    -Name "teamaprodstorage01" `
    -EnableAzureActiveDirectoryKerberosForFile $true `
    -DefaultSharePermission "StorageFileDataSmbShareReader"
```
<br>

#### 3.1.2 Built-in data plane roles for Azure File Shares

| Role | Scope | Read | Write | Delete | Modify ACLs | Take Ownership | Notes |
|------|-------|------|-------|--------|-------------|----------------|-------|
| **Storage File Data SMB Share Reader** | Share | 🗹 | 𐄂 | 𐄂 | 𐄂 | 𐄂 | Read-only access |
| **Storage File Data SMB Share Contributor** | Share | 🗹 | 🗹 | 🗹 | 𐄂 | 𐄂 | Standard user access |
| **Storage File Data SMB Share Elevated Contributor** | Share | 🗹 | 🗹 | 🗹 | 🗹 | 𐄂 | Can modify NTFS ACLs |
| **Storage File Data SMB Share Take Ownership** | Share | 🗹 | 🗹 | 🗹 | 🗹 | 🗹 | Can take ownership of any file/folder |
| **Storage File Data SMB Admin** | Storage Account | 🗹 | 🗹 | 🗹 | 🗹 | 🗹 | Full admin across all shares |
| **Storage File Data SMB MI Admin** | Storage Account | 🗹 | 🗹 | 🗹 | 🗹 | 🗹 | Same as Admin, for Managed Identities |

Scope matters ! Share-level roles are assigned per file share (most restrictive, recommended for least-privilege), while storage account-level roles apply to all shares within the account.  

**When to use each role:**
- **Reader** → Slaves and interns
- **Contributor** → Standard team members
- **Elevated Contributor** → Less standard team members
- **Take Ownership** → IT admins for recovery scenarios 
- **Admin / MI Admin** → Backup services, automation scripts, emergency access  

<br>

Setting up the correct RBAC for "Super Admins" : 

```powershell
# Super Admins group: Full admin rights on all shares
$superAdmins = Get-AzADGroup -DisplayName "Super Admins"
New-AzRoleAssignment `
    -ObjectId $superAdmins.Id `
    -RoleDefinitionName "Storage File Data SMB Admin" `
    -Scope "/subscriptions/xxx-xxx-xxx/resourceGroups/rg-storage-lab/providers/Microsoft.Storage/storageAccounts/teamaprodstorage01"
```

#### 3.1.3 Custom role creation
What about classical users ? For example, we'd like to give the Contributor role on all shares to the `Team A` group members. In our Entra Kerberos based scenario, it won't work with the built-in RBAC (here, "Storage File Data SMB Share Contributor"), because we need to add the following permissions to the role :
- *Microsoft.Storage/storageAccounts/fileServices/readFileBackupSemantics/action*
- *Microsoft.Storage/storageAccounts/fileServices/writeFileBackupSemantics/action*

Despite being the officially recommended method by Microsoft for granting access to Azure File Shares via Entra ID Kerberos, none of the built-in roles include the required backup semantic permissions. We therefore need to create a custom role for our scenario ...

So let's create our custom role, "k8s style". First, store the JSON definition file of the SMB Share Contributor Role :  

```powershell
Get-AzRoleDefinition -Name "Storage File Data SMB Share Contributor" | ConvertTo-Json > .misc/custom_smb_role.json
```

Then, edit the *custom_smb_role.json* file :
```json
{
  "Name": "Custom - SMB Share Contributor", # Modified field  
  "Id": null,                               # Modified field                              
  "IsCustom": true,                         # Modified field                      
  "Description": "Allows for read, write, and delete access in Azure Storage file shares over SMB", 
  "assignableScopes": [
    "/providers/Microsoft.Management/managementGroups/xxx-xxx-xxx" # Replace with the valid MG's ID
  ],
  "Actions": [],
  "NotActions": [],
  "DataActions": [
    "Microsoft.Storage/storageAccounts/fileServices/fileshares/files/read",
    "Microsoft.Storage/storageAccounts/fileServices/fileshares/files/write",
    "Microsoft.Storage/storageAccounts/fileServices/fileshares/files/delete",
    "Microsoft.Storage/storageAccounts/fileServices/readFileBackupSemantics/action",    # Added field
    "Microsoft.Storage/storageAccounts/fileServices/writeFileBackupSemantics/action"    # Added field
  ],
  "NotDataActions": [],
  "Condition": null,
  "ConditionVersion": null
}
```

And add it to Azure :
```powershell
New-AzRoleDefinition -InputFile ".misc/custom_smb_role.json"
```

<details>
<summary>Output :</summary>

```
Name             : Custom - SMB Share Contributor
Id               : xxxx-xxxx-xxxx
IsCustom         : True
Description      : Allows for read, write, and delete access in Azure Storage file shares over SMB
Actions          : {}
NotActions       : {}
DataActions      : {Microsoft.Storage/storageAccounts/fileServices/fileshares/files/read,
                   Microsoft.Storage/storageAccounts/fileServices/fileshares/files/write,
                   Microsoft.Storage/storageAccounts/fileServices/fileshares/files/delete,
                   Microsoft.Storage/storageAccounts/fileServices/readFileBackupSemantics/action…}
NotDataActions   : {}
AssignableScopes : {/providers/Microsoft.Management/managementGroups/xxxx-xxxx-xxxx}
```
</details>

<br>

Now, we can assign the new role to the `Team A` group :

```powershell
# Team A Prod: read + write + delete (no ACL management)
$teamAProd = Get-AzADGroup -DisplayName "Team A"
New-AzRoleAssignment `
    -ObjectId $teamaProd.Id `
    -RoleDefinitionName "Custom - SMB Share Contributor" `
    -Scope "/subscriptions/xxx-xxx-xxx/resourceGroups/rg-storage-lab/providers/Microsoft.Storage/storageAccounts/teamaprodstorage01"
```



<br>

Now that everything is set up, we can write in our shares, so let's try to execute the folder creation command again :

```powershell
$context = New-AzStorageContext -StorageAccountName "teamaprodstorage01" -UseConnectedAccount
$share = Get-AzStorageShare -Name "shared-documents" -Context $context

$share | New-AzStorageDirectory -Path "projects" -Context $context
$share | New-AzStorageDirectory -Path "templates" -Context $context
```

Aaaand ... same error. Why ?  
Because we need to specify the "*x-ms-file-request-intent*" HTTP header. However, the Powershell cmdlet doesn't offer this option, even with the most recent module's version (9.6.0 at the time I write those lines). So, like in the [lab 3 of section 1](../01%20-%20Identity/lab_part3.md), we'll use Azure CLI instead.

```powershell
az storage directory create `
    --account-name "teamaprodstorage01" `
    --share-name "shared-documents" `
    --name "projects" `
    --auth-mode login `
    --enable-file-backup-request-intent

az storage directory create `
    --account-name "teamaprodstorage01" `
    --share-name "shared-documents" `
    --name "templates" `
    --auth-mode login `
    --enable-file-backup-request-intent
```

Why is this header giving us so much trouble ?  

The `x-ms-file-request-intent: backup` header was introduced by Microsoft to distinguish between two access patterns:

1. **Standard SMB access**: Requires the client to be domain-joined (AD DS, Entra DS, or Entra ID) and uses Kerberos authentication with proper ticket exchange
2. **Backup/Admin access**: Allows OAuth-based access without domain join, intended for backup services, automation scripts, and administrative tools

When you enable Entra ID Kerberos authentication on a storage account, Azure expects clients to authenticate via Kerberos. However, PowerShell and Azure CLI use OAuth tokens (from `Connect-AzAccount` or `az login`), not Kerberos tickets. The `backup-request-intent` header tells Azure: "I'm accessing this share programmatically for administrative purposes, accept my OAuth token instead of requiring Kerberos."

The name is misleading, even if we're not necessarily performing a backup, it's simply the mechanism Microsoft created to allow REST API and CLI access to identity-based Azure File Shares from non-domain-joined machines.


### 3.2 Files upload and download

-> Too messy with Powershell / CLI for now, using a cloud-only identity. 
I did my files download / upload operations for the rest of this lab with the Azure Storage Explorer. I'll write the rest of those steps later.

#### 3.2.1 Upload a file

**[ In progress]**

#### 3.2.2 Download a file

**[ In progress]**


### 3.3 List share contents

**[ In progress]**

<br>

## 4. Mount file shares


### 4.1 Mount on Windows using SMB

**[ In progress]**

### 4.2 Mount on Linux using SMB

**[ In progress]**

<br>

## 5. Snapshots

File share snapshots capture the state of the share at a point in time. They're incremental (only changes are stored) and useful for :
- Protecting against accidental deletions
- Quick rollback without full backup restore
- Development/testing with real data

> **Important :** Snapshots are taken at the share level, not at the individual file level. However, it's possible to restore individual files from a snapshot, similar to VSS on a on-premise SMB File Share.

<br>

### 5.1 Create a snapshot via Azure Portal

1. Go to `teamafilestore01` → **File shares** → **shared-documents**
2. Click **"Snapshots"** (in the left menu, under **Operations**)
3. Click **"+ Add snapshot"**
4. Add a comment : `Pre-deletion snapshot`
5. Click **"OK"**

Now we should see the snapshot listed with its timestamp :

Name | Date created | Initator | Comment |
-----|--------------|----------|---------|
2026-02-28T04:26:51.0000000Z | 2/28/2026, 12:26:51 PM | Manual | Pre-deletion snapshot |
<br>

### 5.2 Create a snapshot via PowerShell

**[ In progress]**

<br>

### 5.3 Delete a file

Let's delete a file so we can restore it from the snapshot :

Via Azure Portal :
1. Go to **shared-documents** → **projects**
2. Select `notes.txt`
3. Click **"Delete"** → Confirm

Via PowerShell / Azure CLI :  

**[ In progress]**

<br>

### 5.4 Restore a file from snapshot

Via Azure Portal :
1. Go to **shared-documents** → **Snapshots**
2. Click on the snapshot we created
3. Navigate to **projects/**
4. Click on `notes.txt`
5. Click **"Restore"**
6. Choose **"Overwrite original file"** (or "Rename" to keep both)
7. Click **"Restore"**

✅ The file is back in its original location.

<br>

Via PowerShell / Azure CLI :  

**[ In progress]**

<br>

## 6. Soft delete for file shares

Soft delete provides a safety net against accidental share deletion. When enabled, deleted shares are retained for a configurable period before permanent removal.

> **Note :** Soft delete protects against **share-level** deletion (deleting the entire share), not individual file deletion. For individual file protection, use **snapshots**.

<br>

### 6.1 Enable soft delete via Azure Portal

1. Go to `teamafilestore01`
2. Select **File shares** under **Data storage**
3. In the **File share settings** view, click on the **Soft delete** field value
4. Enable **Soft delete for all file shares** and set retention to **14 days**
5. Click **"Save"**

<br>

### 6.2 Enable soft delete via PowerShell

```powershell
Update-AzStorageFileServiceProperty `
    -ResourceGroupName "rg-storage-lab" `
    -StorageAccountName "teamaprodstorage01" `
    -EnableShareDeleteRetentionPolicy $true `
    -ShareRetentionDays 14
```
<details>
<summary>Output (part of) :</summary>

```powershell
StorageAccountName                                : teamaprodstorage01
ResourceGroupName                                 : rg-storage-lab
ShareDeleteRetentionPolicy.Enabled                : True
ShareDeleteRetentionPolicy.Days                   : 14
```
</details>

<br>

### 6.3 Verify soft delete is enabled

```powershell
Get-AzStorageFileServiceProperty -ResourceGroupName "rg-storage-lab" -StorageAccountName "teamaprodstorage01"
```

<details>
<summary>Output (part of) :</summary>

```powershell
# Same as the previous command
StorageAccountName                                : teamaprodstorage01
ResourceGroupName                                 : rg-storage-lab
ShareDeleteRetentionPolicy.Enabled                : True
ShareDeleteRetentionPolicy.Days                   : 14
```
</details>

<br>

### 6.4 Test soft delete

Let's delete the `team-backups` share and then recover it :

```powershell
# Delete the share
Remove-AzStorageShare -Name "team-backups" -Context $context -Force

# List shares/ Verify deletion
Get-AzStorageShare -Context $context
```
Output :
```
   File End Point: https://teamaprodstorage01.file.core.windows.net/

Name             QuotaGiB LastModified                 IsSnapshot SnapshotTime                 
----             -------- ------------                 ---------- ------------                 
shared-documents 100      2026-02-28T04:26:51.0000000Z True       2026-02-28T04:26:51.0000000Z
shared-documents 100      2026-02-21T03:42:56.0000000Z False
```

The share is now soft-deleted. We can see it in the Portal :
1. Go to **File shares**
2. Toggle **"Show deleted shares"** at the top
3. `team-backups` appears with a status of **Deleted** and the expiry date

Or we can see it with Powershell :
```powershell
Get-AzStorageShare -Context $context -IncludeDeleted
```

Output :
```
Name             QuotaGiB LastModified                 IsSnapshot SnapshotTime                 Protocols IsDeleted
----             -------- ------------                 ---------- ------------                 --------- ---------
shared-documents 100      2026-02-28T04:26:51.0000000Z True       2026-02-28T04:26:51.0000000Z
shared-documents 100      2026-02-21T03:42:56.0000000Z False
team-backups     50       2026-02-20T14:03:15.0000000Z False                                             True
```

<br>

**Recover the deleted share :**

Via Portal :
1. Click on the "**...**" field of the deleted `team-backups` share
2. Click **"Undelete"**

Via PowerShell :
```powershell
# Get the "team-backups" delete share object
$deletedShare = Get-AzStorageShare -Context $context -IncludeDeleted | Where-Object { $_.IsDeleted -and $_.Name -eq "team-backups"}

# Restore the share
Restore-AzRmStorageShare -ResourceGroupName "rg-storage-lab" -AccountName "teamaprodstorage01" -Name $deletedShare.Name -DeletedShareVersion $deletedShare.VersionID
```
<details>
<summary>Output :</summary>

```
   ResourceGroupName: rg-storage-lab, StorageAccountName: teamaprodstorage01

Name         QuotaGiB EnabledProtocols AccessTier Deleted Version ShareUsageBytes SnapshotTime
----         -------- ---------------- ---------- ------- ------- --------------- ------------
team-backups 50                        Cool
```
</details>  

<br>

The share is now restored with all its contents intact.

<br>

> **Note :** Soft delete is a safety net, not a backup strategy. It protects against accidental share deletion for a limited time. For comprehensive data protection, combine soft delete with snapshots and Azure Backup.

<br>

## Key Takeaways

| Concept | Remember |
|---------|----------|
| **Tiers** | Premium (SSD), Transaction Optimized, Hot, Cool - set at share level |
| **Protocols** | SMB (all tiers, all OS) vs NFS (Premium only, Linux only) |
| **Port 445** | Required for SMB, often blocked by ISPs |
| **Quotas** | Soft limit on share size, can be changed anytime |
| **Snapshots** | Share-level, incremental, restore individual files |
| **Soft delete** | Protects against share deletion, not file deletion |
| **Large file shares** | Enable at account creation for shares > 5 TiB, limits redundancy to LRS/ZRS |

<br>

> **Exam tip :** Know the differences between Azure Files and Blob Storage (access method, use case, structure). Also remember that NFS requires Premium tier (FileStorage account) and is Linux-only.

<br>

## Personal thoughts: What happened to Microsoft's engineering rigor ?

Working through this lab has been a frustrating reminder that Microsoft's tooling quality has noticeably declined. A few observations :  



**Fragmented tooling with inconsistent capabilities :**  
- PowerShell cmdlets, Azure CLI, REST API, and Azure Portal each have different feature coverage
- `New-AzStorageDirectory` doesn't support the `-FileRequestIntent` parameter despite it being documented in some Microsoft articles
- Azure CLI works where PowerShell fails, but you have to discover this through trial and error

**Documentation-reality gaps :**  
- Built-in RBAC roles are documented as the solution for Entra ID Kerberos scenarios, yet they lack the required `readFileBackupSemantics` and `writeFileBackupSemantics` permissions
- You must create custom roles to make the "recommended" approach actually work
- Error messages reference headers that the tooling doesn't expose

**The "Portal works, CLI doesn't" phenomenon :**   
- The Azure Portal's "Browse" feature with Entra authentication works fine once RBAC is set
- Achieving the same via PowerShell requires workarounds that aren't documented  

**Inconsistent cmdlet naming conventions :**  

Even within a single resource type (Azure File Shares), PowerShell cmdlet naming is inconsistent:

| Operation | Cmdlet | Prefix |
|-----------|--------|--------|
| Create share | `New-AzStorageShare` | Az |
| Get share | `Get-AzStorageShare` | Az |
| Update share | `Update-AzRmStorageShare` | AzRm |
| Remove share | `Remove-AzStorageShare` | Az |
| Restore share | `Restore-AzRmStorageShare` | AzRm |

The `Az` cmdlets operate on the data plane (via storage context), while `AzRm` cmdlets operate on the ARM control plane (via resource group and account name). But why would *updating* a share's tier require ARM while *creating* it doesn't? Why is *restore* an ARM operation but *delete* isn't?

This forces you to constantly switch mental models:
```powershell
# Create with Az (needs $context)
New-AzStorageShare -Name "myshare" -Context $context

# Update with AzRm (needs resource group, no context)
Update-AzRmStorageShare -ResourceGroupName "rg" -StorageAccountName "sa" -Name "myshare" -QuotaGiB 100

# Delete with Az (back to $context)
Remove-AzStorageShare -Name "myshare" -Context $context
```

There's no discernible logic to which operations use which prefix. You just have to memorize it, or more realistically, fail, Google it, and try the other prefix.

**Contrast with the past :**  
Those of us who remember Microsoft's developer tooling from the 2000s-2015s era (Visual Studio, MSDN documentation, .NET Framework) recall a company obsessive about consistency, backward compatibility, and comprehensive documentation. Every API was meticulously documented with examples, edge cases, and error codes.

Today's Azure feels like it's developed by dozens of independent teams shipping features at different velocities, with no central authority ensuring coherence. The rapid release cadence prioritizes "new features" over "features that work reliably together."
