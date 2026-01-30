# 3. Blob Storage & Containers

**Lab instructions reminder :**  
Create and manage blob containers and blobs in the existing storage accounts :

1. Understand blob types (Block, Append, Page) and their access levels
2. Create 3 new containers : ***documents***, ***logs***, and ***images*** on ***teamadevstorage01***
3. Set the new containers access levels :
   - documents -> ***blob***
   - logs -> ***private***
   - images -> ***container***
4. Upload and download blobs using PowerShell
   - Upload 2 image files to the ***images*** container
   - Upload a html file to the ***documents*** container
5. List and explore blobs using Powhershell
6. Work with blob metadata, properties and index tags
   - Upload a text file to the ***documents*** container with ***4 metadatas***
   - Upload a text file to the ***documents*** container with ***5 index tags***

<br>

Cmdlets used in this lab :
- [New-AzStorageContainer](https://learn.microsoft.com/en-us/powershell/module/az.storage/new-azstoragecontainer)
- [Get-AzStorageContainer](https://learn.microsoft.com/en-us/powershell/module/az.storage/get-azstoragecontainer)
- [Set-AzStorageContainerAcl](https://learn.microsoft.com/en-us/powershell/module/az.storage/set-azstoragecontaineracl?view=azps-15.2.0)
- [Get-AzStorageBlobContent](https://learn.microsoft.com/en-us/powershell/module/az.storage/get-azstorageblobcontent)
- [Set-AzStorageBlobContent](https://learn.microsoft.com/en-us/powershell/module/az.storage/set-azstorageblobcontent)
- [Get-AzStorageBlob](https://learn.microsoft.com/en-us/powershell/module/az.storage/get-azstorageblob)
- [Get-AzStorageBlobByTag](https://learn.microsoft.com/en-us/powershell/module/az.storage/get-azstorageblobbytag?view=azps-15.2.0)
- [Remove-AzStorageBlob](https://learn.microsoft.com/en-us/powershell/module/az.storage/remove-azstorageblob)

Useful documentations :
- [Manage blob containers using PowerShell](https://learn.microsoft.com/en-us/azure/storage/blobs/blob-containers-powershell)
- [Upload, download, and list blobs - PowerShell](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-powershell)

<br>

## 1. Understanding Blob types and access levels

Azure Blob Storage supports three types of blobs, each optimized for different scenarios :

| Type | Max Size | Use Case | Key Characteristic |
|------|----------|----------|-------------------|
| **Block Blob** | 190.7 TiB | Images, documents, videos, backups | Composed of blocks, optimized for sequential read/write |
| **Append Blob** | 195 GiB | Log files, audit trails | Only append operations allowed, no modifications |
| **Page Blob** | 8 TiB | VM disks (.vhd), databases | Random read/write access, 512-byte pages |

> **Quick tip :** If you're unsure which type to use, go with Block Blob - it's the default and covers most use cases.

<br>


Containers have three access levels that control anonymous (public) access :

| Level | Anonymous Access | Use Case |
|-------|-----------------|----------|
| **Private** | None | Default, most secure - requires authentication |
| **Blob** | Read access to blobs only | Public files, but can't list container contents |
| **Container** | Read access to container + blobs | Fully public, can list and read all blobs |

> **Remember :** In Lab 1, we disabled `AllowBlobPublicAccess` at the storage account level. This means even if we set a container to "Blob" or "Container" access, anonymous access won't work. This is a security best practice !

<br>

## 2. Create 3 new containers on `teamadevstorage01`

Let's create some containers in our `teamadevstorage01` storage account. We'll use these throughout the lab.

```powershell
Set-AzContext -Subscription "Team A - Dev"

# Get storage context - since we disabled shared keys, we use Azure AD auth
$context = New-AzStorageContext -StorageAccountName "teamadevstorage01" -UseConnectedAccount

# Create containers for different purposes
New-AzStorageContainer -Name "documents" -Context $context
New-AzStorageContainer -Name "logs" -Context $context
New-AzStorageContainer -Name "images" -Context $context
```

<details>
<summary>Output</summary>

```
   Storage Account Name: teamadevstorage01

Name       PublicAccess LastModified
----       ------------ ------------
documents  Off          22/01/2026 08:29:58 +00:00
logs       Off          22/01/2026 08:29:59 +00:00
images     Off          22/01/2026 08:30:01 +00:00
```
</details>

<br>

Let's verify our containers :

```powershell
Get-AzStorageContainer -Context $context | Select-Object Name, PublicAccess, LastModified
```

<details>
<summary>Output</summary>

```
Name           PublicAccess LastModified
----           ------------ ------------
dest-container          Off 22/01/2026 03:41:02 +00:00
documents               Off 22/01/2026 08:29:58 +00:00
images                  Off 22/01/2026 08:30:01 +00:00
logs                    Off 22/01/2026 08:29:59 +00:00
```
</details>

> Note : `dest-container` comes from Lab 2's object replication setup.

<br>

## 3. Setting containers access levels

Now let's try to change the public access level of our containers. As seen in section 1, we have three levels: Private, Blob, and Container.

#### Setting different access levels
```powershell
# Set images container to "Container" level
Set-AzStorageContainerAcl -Name "images" -Permission Container -Context $context
```
And ... it won't work :) Here's the ouput :

```
Set-AzStorageContainerAcl: Public access is not permitted on this storage account.
RequestId:xxx-xxx-xxx
Time:2026-01-23T02:43:52.3401924Z
Status: 409 (Public access is not permitted on this storage account.)
ErrorCode: PublicAccessNotPermitted
```

<br>

As said in the error message, the public access is restricted at the storage account level. Indeed, that's what we did in the first lab : we disabled `AllowBlobPublicAccess` at the **storage account level**. This is a security feature that overrides container-level settings !

> **Important :** Container access level settings only work if `AllowBlobPublicAccess = $true` on the storage account. The storage account setting acts as a "master switch" for all public access.

<br>

#### Enable public access for testing

To see anonymous access in action, we'll first have to enable it at the storage account level :

```powershell
Set-AzStorageAccount -ResourceGroupName "rg-storage-lab" -Name "teamadevstorage01" -AllowBlobPublicAccess $true
```

<br>

#### Now we can set our access levels
```powershell
# Set images container to "Container" level
Set-AzStorageContainerAcl -Name "images" -Permission Container -Context $context

# Set documents container to "Blob" level
Set-AzStorageContainerAcl -Name "documents" -Permission Blob -Context $context
```

#### Verify the access levels

```powershell
Get-AzStorageContainer -Context $context | Select-Object Name, PublicAccess | Sort-Object Name
```
Output :

```
Name           PublicAccess
----           ------------
dest-container          Off
documents              Blob
images            Container
logs                    Off
```

<br>

**Key takeaway :** In production, you should keep `AllowBlobPublicAccess = $false` on your storage accounts unless you have a specific business requirement for public blob access. Even then, use SAS tokens (Lab 5) instead of anonymous access whenever possible !  

Now it's time to test our access levels, but before we need to populate our containers. That brings us to the next section 🔽

<br>

## 4. Upload and download blobs

Now let's upload some files to our containers. There are 2 image files in the *.misc/* directory, `img1.png` and `img2.webp`, and a html file `doc1.html`
- images files in the "images" container
- html file in the "documents" container


#### Upload a single blob - doc1.html

```powershell
# Reminder : set the correct subscription and storage account context
Set-AzContext -Subscription "Team A - Dev"
$context = New-AzStorageContext -StorageAccountName "teamadevstorage01" -UseConnectedAccount

# Upload the doc1.html file to documents container
Set-AzStorageBlobContent `
    -Container "documents" `
    -File ".\.misc\doc1.html" `
    -Blob "doc1.html" `
    -Context $context
```

<details>
<summary>Output</summary>

```
   AccountName: teamadevstorage01, ContainerName: documents

Name           BlobType  Length      ContentType                    LastModified         AccessTier
                                                                                                  
----           --------  ------      -----------                    ------------         ----------
doc1.html      BlockBlob 169         application/octet-stream       2026-01-23 03:30:19Z Hot
```
</details>

<br>

#### Upload multiple blobs - img1 and img2

```powershell
gci .\.misc\ -Filter "img*" -File | Set-AzStorageBlobContent -Container "images" -Context $context
```

<details>
<summary>Output</summary>

```
   AccountName: teamadevstorage01, ContainerName: images

Name          BlobType  Length      ContentType                 LastModified         AccessTier
                                                                                               
----          --------  ------      -----------                 ------------         ----------
img1.png      BlockBlob 963071      application/octet-stream    2026-01-23 03:43:50Z Hot
img2.webp     BlockBlob 164300      application/octet-stream    2026-01-23 03:43:50Z Hot
```
</details>

<br>

**Note :** By default, the Content-Type is set to "application/octet-stream". What does that mean ?  
It means that everytime we'll access a blob via a web browser, the browser will automatically downlaoad it. If we want to display the blob without the browser downloading it, we have to specify the correct Content-Type. We'll see how to do this in the 6th section of this lab 🙂

<br>

#### Download a blob

```powershell
Get-AzStorageBlobContent `
    -Container "documents" `
    -Blob "doc1.html" `
    -Destination ".\tmp-doc1.html" `
    -Context $context
```

<br>

## 5. List and explore blobs

#### List all blobs in a container

```powershell
Get-AzStorageBlob -Container "images" -Context $context | Select-Object Name, BlobType, Length, AccessTier
```
#### List all containers' blobs for a storage account
```powershell
Get-AzStorageContainer -Context $context | Get-AzStorageBlob
```

#### List blobs with a specific prefix ("virtual folder" or filtering by naming convention)

```powershell
Get-AzStorageBlob -Container "images" -Prefix "img" -Context $context
```

<br>


## 6. Working with blob metadata, properties and index tags

Azure Blob Storage provides two main ways to attach custom information to blobs:

- **Metadata :** key-value pairs that are **not indexed** by the service. Those are perfect for descriptive or application-specific data
- **Blob Index Tags :** (also called "index tags" or simply "tags") key-value pairs that are **automatically indexed** by Blob Storage, limited to 10 per blob, but enabling fast native queries across millions of objects

| Criterion                | Metadata                              | Blob Index Tags                          |
|--------------------------|---------------------------------------|------------------------------------------|
| Max count / size         | ~8 KB total per blob                  | 10 tags max per blob                     |
| Indexing & search        | No (requires listing + client filtering) | Yes – very fast `Find Blobs by Tags`     |
| Cost                     | Included in blob storage cost         | Small additional cost (~$0.06 / 10k tags/month) |
| Typical use case         | Descriptive info, app tracking        | Classification, fast filtering, lifecycle rules |
| Usable in Lifecycle Mgmt | No                                    | Yes                                      |

<br>

### 6.1 Blob metadatas

#### Upload a blob with metadata

Here we'll upload the `notes.txt` file as a new blob in the `documents` container, and define some metadatas for this blob  
```powershell
$metadata = @{
    "department"        = "learning"
    "author"            = "louis"
    "classification"    = "internal"
    "projectcode"       = "az104"
}

Set-AzStorageBlobContent `
    -Container "documents" `
    -File ".\.misc\notes.txt" `
    -Blob "notes.txt" `
    -Context $context `
    -Metadata $metadata `
    -Properties @{"ContentType" = "text/plain"}
```

#### Read metadata

```powershell
$blob = Get-AzStorageBlob -Container "documents" -Blob "notes.txt" -Context $context
$properties = $blob.BlobClient.GetProperties()
$properties.Value.Metadata
```

Output :
```
Key            Value
---            -----
classification internal
projectcode    az104
department     learning
author         louis
```

Metadatas are great for application-level data, but not suitable for large-scale filtering.

<br>

### 6.2 Blob index tags

#### Upload a blob with index tags

Here we'll upload the `more_notes.txt` file as a new blob in the `documents` container.  The blob will have the same metadatas as the previous one, but we'll add index tag to it.  
```powershell
# Keep the same metadatas as the previous blob
$metadata = @{
    "department"        = "learning"
    "author"            = "louis"
    "classification"    = "internal"
    "projectcode"       = "az104"
}

# Add some tags
$tags = @{
    "project"           = "az104"
    "confidentiality"   = "internal"
    "year"              = "2026"
    "month"             = "january"
    "type"              = "notes"
}

Set-AzStorageBlobContent `
    -Container "documents" `
    -File ".\.misc\more_notes.txt" `
    -Blob "more_notes.txt" `
    -Context $context `
    -Metadata $metadata `
    -Tag $tags `
    -Properties @{"ContentType" = "text/plain"}
```

#### Set or update tags on an existing blob

Tag mistake on the previous blob ! The confidentiality value must be set to "public" and not "internal". Here's how to change it, without having to reupload the blob
```powershell
# Get the "more-notes.txt" blob
$blob = Get-AzStorageBlob -Container "documents" -Blob "more_notes.txt" -Context $context

# Overwrite the "confidentiality" tag to "public"
$tagsUpdate = @{
    "project"           = "az104"
    "confidentiality"   = "public"
    "year"              = "2026"
    "month"             = "january"
    "type"              = "notes"
}

Set-AzStorageBlobTag `
    -Container "documents" `
    -Blob $blob.Name `
    -Tag $tagsUpdate `
    -Context $context
```

#### Search blobs by tags : the real power!

Real power, but very simple example here ... you get the idea :)
```powershell
Get-AzStorageBlobByTag -TagFilterSqlExpression """project""='az104'" -Context $context
```
<br>

**Key takeaway:** Index tags excel at:
- Quickly finding blobs without listing millions of objects
- Driving lifecycle management rules (e.g., archive if `type='log'`)
- Enabling fine-grained access control with ABAC

<br>

### 6.3 Blob properties

System properties affect behavior, especially for web access. Remember our images blobs we uploaded on the 4th section of this lab ?  
We didn't specify any property, and one of these properties is the "Content-Type". It's a good pratice to set this value based on the blob's type we're uploading.
We can do it by adding a hashtable to the `Properties` paramaeter of the `Set-AzStorageBlobContent` cmdlet, as seen in the two previous exercices where we added some metadatas and tags to new blobs.  

And that's it ! With well-managed metadata, index tags, and properties, blobs become much more easy-to-find, organized, and automation-friendly.