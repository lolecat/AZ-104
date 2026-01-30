# 5. SAS Tokens & Stored Access Policies

**Lab instructions reminder :**  
Secure storage access with Shared Access Signatures:  
1. Understand the 3 SAS types (Account SAS, Service SAS, User Delegation SAS) and when to use each
2. Generate an Account SAS on ***teamadevstorage01*** via Azure Portal
   - Grant read + list permissions on Blob service only
   - Set expiry to 1 hour
   - Test access with the generated SAS URL
3. Generate a Service SAS on the ***documents*** container via PowerShell
   - Grant read permission on blobs
   - Test by accessing `doc1.html` with the SAS token
4. Generate a User Delegation SAS via PowerShell (the most secure type)
   - Grant read + list on the ***images*** container
   - Test access with the generated SAS URL
5. Create a Stored Access Policy on the ***documents*** container
   - Define read + list permissions with a 24-hour window
   - Generate a Service SAS linked to the policy
6. Revoke access by modifying / deleting the Stored Access Policy

<br>

Cmdlets used in this lab :
- [New-AzStorageAccountSASToken](https://learn.microsoft.com/en-us/powershell/module/az.storage/new-azstorageaccountsastoken)
- [New-AzStorageContainerSASToken](https://learn.microsoft.com/en-us/powershell/module/az.storage/new-azstoragecontainersastoken)
- [New-AzStorageBlobSASToken](https://learn.microsoft.com/en-us/powershell/module/az.storage/new-azstorageblobsastoken)
- [New-AzStorageContainerStoredAccessPolicy](https://learn.microsoft.com/en-us/powershell/module/az.storage/new-azstoragecontainerstoredaccesspolicy)
- [Get-AzStorageContainerStoredAccessPolicy](https://learn.microsoft.com/en-us/powershell/module/az.storage/get-azstoragecontainerstoredaccesspolicy)
- [Set-AzStorageContainerStoredAccessPolicy](https://learn.microsoft.com/en-us/powershell/module/az.storage/set-azstoragecontainerstoredaccesspolicy)
- [Remove-AzStorageContainerStoredAccessPolicy](https://learn.microsoft.com/en-us/powershell/module/az.storage/remove-azstoragecontainerstoredaccesspolicy)

Useful documentations :
- [Grant limited access with SAS](https://learn.microsoft.com/en-us/azure/storage/common/storage-sas-overview)
- [Create a User Delegation SAS with PowerShell](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-user-delegation-sas-create-powershell)
- [Define a Stored Access Policy](https://learn.microsoft.com/en-us/azure/storage/common/storage-stored-access-policy-define-dotnet)

<br>

## 1. Understanding SAS types

A Shared Access Signature (SAS) is a URI that grants restricted access to Azure Storage resources. Instead of sharing your account key (which gives full access to everything), you create a SAS token with specific permissions, time limits, and scope.

There are three types of SAS, each with different security implications :

| | Account SAS | Service SAS | User Delegation SAS |
|---|---|---|---|
| **Signed with** | Storage account key | Storage account key | Entra ID credentials |
| **Scope** | Entire account (all services) | Single service (Blob, File, Queue, or Table) | Blob & Data Lake Storage only |
| **Granularity** | Service, container, or object level | Container or blob level | Container or blob level |
| **Requires shared keys** | Yes | Yes | No |
| **Revocation** | Regenerate account key, or use Stored Access Policy (Service SAS only) | Delete/modify Stored Access Policy, or regenerate key | Revoke User Delegation Key, or wait for expiry |
| **Security level** | ⚠️ Lowest | ⚠️ Medium | ✅ Highest |
| **Microsoft recommendation** | Avoid if possible | Use with Stored Access Policy | **Preferred** - always use when possible |

> **Key takeaway :** Microsoft recommends **User Delegation SAS** whenever possible because it doesn't require storage account keys. This aligns with what we did in Lab 1 : we disabled shared key access for better security. For Account SAS and Service SAS to work, shared keys must be enabled.

<br>

### Anatomy of a SAS token

A SAS token is a query string appended to a storage resource URL. Here's what the parameters mean :

| Parameter | Name | Example | Description |
|-----------|------|---------|-------------|
| `sv` | Signed version | `2025-07-05` | Storage API version |
| `ss` | Signed services | `b` | b=Blob, f=File, q=Queue, t=Table |
| `srt` | Signed resource types | `sco` | s=Service, c=Container, o=Object |
| `sp` | Signed permissions | `rl` | r=Read, l=List, w=Write, d=Delete, etc. |
| `se` | Signed expiry | `2026-01-28T10:00:00Z` | Expiry date/time (UTC) |
| `st` | Signed start | `2026-01-27T09:00:00Z` | Start date/time (optional) |
| `spr` | Signed protocol | `https` | https or https,http |
| `sig` | Signature | `abc123...` | HMAC-SHA256 signature |
| `sdd` | Signed directory depth | `2` | User Delegation SAS only |
| `skoid` | Signed key object ID | `guid` | User Delegation SAS : Azure AD object ID |

> **Note :** Understanding `sp` (permissions), `se` (expiry), and `sig` (signature) is essential.

<br>

## 2. Generate an Account SAS via Azure Portal

An Account SAS grants access to resources in one or more storage services. Let's generate one with read + list permissions on the Blob service.

### 2.1 Re-enable shared key access

Since we disabled shared keys in Lab 1, we need to re-enable them temporarily. Account SAS and Service SAS both require shared key access.

1. Go to `teamadevstorage01` storage account
2. **Settings** → **Configuration**
3. Set **Allow storage account key access** to **Enabled**
4. **"Save"**

Or via PowerShell :
```powershell
Set-AzContext -Subscription "Team A - Dev"
Set-AzStorageAccount -ResourceGroupName "rg-storage-lab" -Name "teamadevstorage01" -AllowSharedKeyAccess $true
```

<br>

### 2.2 Generate the Account SAS

1. Go to `teamadevstorage01` storage account
2. **Security + networking** → **Shared access signature**
3. Configure the following settings :

| Setting | Value |
|---------|-------|
| Allowed services | ☑️ Blob only |
| Allowed resource types | ☑️ Service ☑️ Container ☑️ Object |
| Allowed permissions | ☑️ Read ☑️ List |
| Start and expiry date/time | Start: now, Expiry: +1 hour |
| Allowed protocols | HTTPS only |
| Signing key | key1 |

4. Click **"Generate SAS and connection string"**

This gives us the following connection strings :  
- SAS Token : `sv=2024-11-04&ss=b&srt=sco&sp=rlf&se=2026-01-28T15:05:46Z&st=2026-01-28T13:50:46Z&spr=https&sig=8%2BeVpcnQMy7baGkrG%2Fuha0jB6flb%2FBH%2B9IOARw8s5Go%3D`
- Blob service SAS URL : `https://teamadevstorage01.blob.core.windows.net/?sv=2024-11-04&ss=b&srt=sco&sp=rlf&se=2026-01-28T15:05:46Z&st=2026-01-28T13:50:46Z&spr=https&sig=8%2BeVpcnQMy7baGkrG%2Fuha0jB6flb%2FBH%2B9IOARw8s5Go%3D`

5. Copy the **Blob service SAS URL**

<br>

### 2.3 Test the Account SAS

Let's test our SAS by listing blobs in the `images` container. Open a PowerShell terminal :

```powershell
$sasUrl = "https://teamadevstorage01.blob.core.windows.net/?sv=2024-11-04&ss=b&srt=sco&sp=rlf&se=2026-01-28T15:05:46Z&st=2026-01-28T13:50:46Z&spr=https&sig=8%2BeVpcnQMy7baGkrG%2Fuha0jB6flb%2FBH%2B9IOARw8s5Go%3D"

# List blobs in the images container using the SAS
$listUrl = "https://teamadevstorage01.blob.core.windows.net/images?restype=container&comp=list&$($sasUrl.Split('?')[1])"
Invoke-RestMethod -Uri $listUrl -Method Get
```

<details>
<summary>Output</summary>

```xml
xml version="1.0" encoding="utf-8"
EnumerationResults ServiceEndpoint="https://teamadevstorage01.blob.core.windows.net/" ContainerName="images"
  Blobs
    Blob
      Name: img1.png
      Properties
        Content-Length: 963071
        Content-Type: application/octet-stream
        AccessTier: Cool
    Blob
      Name: img2.webp
      Properties
        Content-Length: 164300
        Content-Type: application/octet-stream
        AccessTier: Hot
```
</details>

<br>

**Note :** Account SAS gives broad access. If someone gets this token, they can read and list all blobs across all containers. That's why it's the least secure option.

<br>

## 3. Generate a Service SAS via PowerShell

A Service SAS is scoped to a single service (Blob, File, Queue, or Table). Let's create one that grants read access to blobs in the `documents` container. PowerShell is more convenient here because it gives us granular control over the SAS parameters in a scriptable way.

### 3.1 Generate the Service SAS token

```powershell
Set-AzContext -Subscription "Team A - Dev"

$keys = Get-AzStorageAccountKey -ResourceGroupName "rg-storage-lab" -Name "teamadevstorage01"
$keyContext = New-AzStorageContext -StorageAccountName "teamadevstorage01" -StorageAccountKey $keys[0].Value

# Generate a Service SAS for the documents container
$sasToken = New-AzStorageContainerSASToken `
    -Name "documents" `
    -Permission "rl" `
    -ExpiryTime (Get-Date).AddHours(1) `
    -Protocol HttpsOnly `
    -Context $keyContext

Write-Output "SAS Token: $sasToken"
```

<details>
<summary>Output</summary>

```
SAS Token: sv=2025-07-05&spr=https&se=2026-01-30T03%3A51%3A19Z&sr=c&sp=rl&sig=DY6dHAuZIt6YTSx69i5uUejKK%2F8Bmn8gR4eo0NtkeUM%3D
```
</details>

<br>

### 3.2 Test accessing doc1.html

```powershell
$blobUrl = "https://teamadevstorage01.blob.core.windows.net/documents/doc1.html" + "?$sasToken"
Invoke-WebRequest -Uri $blobUrl
```

<details>
<summary>Output</summary>

```
StatusCode        : 200
StatusDescription : OK
Content           :  <!DOCTYPE html>
                    <html>
                    <head>
                    <title>AZ-104 Lab 2</title>
                    </head>
                    <body>

                    <h1>It's about storage</h1>
                    <p>And playing with access levels</p>

                    </body>
                    </html> 
RawContent        : HTTP/1.1 200 OK
                    Accept-Ranges: bytes
                    ETag: "0x8DE5A357EB24DCC"
                    Server: Windows-Azure-Blob/1.0
                    Server: Microsoft-HTTPAPI/2.0
                    x-ms-request-id: 4748e689-101e-00ad-7793-918f98000000
                    x-ms-version: 20…
Headers           : {[Accept-Ranges, System.String[]], [ETag, System.String[]], [Server, System.String[]], [x-ms-request-id, System.String[]]…}
Images            : {}
InputFields       : {}
Links             : {}
RawContentLength  : 169
RelationLink      : {}
```
</details>

<br>

### 3.3 Generate a blob-level SAS (even more restrictive)

We can also create a SAS for a single blob instead of an entire container. This SAS grants access to `notes.txt` only, not other blobs in the container :  

```powershell
$blobSas = New-AzStorageBlobSASToken `
    -Container "documents" `
    -Blob "notes.txt" `
    -Permission "r" `
    -ExpiryTime (Get-Date).AddHours(1) `
    -Protocol HttpsOnly `
    -Context $keyContext

$directUrl = "https://teamadevstorage01.blob.core.windows.net/documents/notes.txt" + "?$blobSas"
Write-Output "Direct blob URL: $directUrl"
```
<details>
<summary>Output</summary>

```
StatusCode        : 200
StatusDescription : OK
Content           : Some notes about stuff
RawContent        : HTTP/1.1 200 OK
                    Accept-Ranges: bytes
                    ETag: "0x8DE5A582A5295B2"
                    Server: Windows-Azure-Blob/1.0
                    Server: Microsoft-HTTPAPI/2.0
                    x-ms-request-id: 49a33ec6-501e-0048-3795-91deda000000
                    x-ms-version: 20…
Headers           : {[Accept-Ranges, System.String[]], [ETag, System.String[]], [Server, System.String[]], [x-ms-request-id, System.String[]]…}
Images            : {}
InputFields       : {}
Links             : {}
RawContentLength  : 22
RelationLink      : {}
```
</details>

<br>

## 4. Generate a User Delegation SAS (the most secure type)

User Delegation SAS is signed with Azure AD credentials instead of the storage account key. This is the recommended approach by Microsoft because :
- No storage account keys are involved
- Permissions are tied to the Azure AD identity
- The delegation key can be revoked independently

### 4.1 Prerequisites : Storage Blob Delegator role

The identity generating the User Delegation SAS needs the **Storage Blob Delegator** role on the storage account. Let's verify and assign if needed.

Via Azure Portal :
1. Go to `teamadevstorage01` → **Access Control (IAM)**
2. Click **"+ Add"** → **"Add role assignment"**
3. Search for **Storage Blob Delegator**
4. Assign to your user account
5. **"Review + assign"**

Or via PowerShell :  
```powershell
$userId = (Get-AzADUser -UserPrincipalName "louis@lolecat.me").Id

New-AzRoleAssignment `
    -ObjectId $userId `
    -RoleDefinitionName "Storage Blob Delegator" `
    -Scope "/subscriptions/xxxx-xxxx-xxxx/resourceGroups/rg-storage-lab/providers/Microsoft.Storage/storageAccounts/teamadevstorage01"
```

**Note :** The **Storage Blob Data Reader** (or higher) role is also needed to actually read blobs. The Delegator role only allows creating the delegation key, not accessing data itself.

<br>

### 4.2 Generate the User Delegation SAS : Grant read + list on the images container

```powershell
Set-AzContext -Subscription "Team A - Dev"

$context = New-AzStorageContext -StorageAccountName "teamadevstorage01" -UseConnectedAccount

# Generate a User Delegation SAS for the images container
$userSas = New-AzStorageContainerSASToken `
    -Name "images" `
    -Permission "rl" `
    -ExpiryTime (Get-Date).AddHours(1) `
    -Protocol HttpsOnly `
    -Context $context
```

<details>
<summary>Output</summary>

```
skoid=xxxx-xxxx&sktid=xxxx-xxxx&skt=2026-01-30T03%3A14%3A25Z&ske=2026-01-30T04%3A14%3A25Z&sks=b&skv=2025-07-05&sv=2025-07-05&spr=https&se=2026-01-30T04%3A14%3A25Z&sr=c&sp=rl&sig=eMHb%2FJhIxqjx1lZMe5nXq%2Bxrtxon4buw3T5lnuPB9to%3D

$userSas.Split('&')

skoid=xxxx-xxxx
sktid=xxxx-xxxx
skt=2026-01-30T03%3A14%3A25Z
ske=2026-01-30T04%3A14%3A25Z
sks=b
skv=2025-07-05
sv=2025-07-05
spr=https
se=2026-01-30T04%3A14%3A25Z
sr=c
sp=rl
sig=eMHb%2FJhIxqjx1lZMe5nXq%2Bxrtxon4buw3T5lnuPB9to%3D
```
</details>

<br>

Notice the extra parameters (`skoid`, `sktid`, `skt`, `ske`) : these identify the Azure AD user who signed the token. That's the key difference with Account/Service SAS.

### 4.3 Test the User Delegation SAS

List blobs in images container :  
```powershell
$listUrl = "https://teamadevstorage01.blob.core.windows.net/images?restype=container&comp=list&" + $userSas.TrimStart("?")
Invoke-RestMethod -Uri $listUrl
```

<details>
<summary>Output (part of)</summary>

```xml
xml version="1.0" encoding="utf-8"
EnumerationResults ServiceEndpoint="https://teamadevstorage01.blob.core.windows.net/" ContainerName="images"
  Blobs
    Blob
      Name: img1.png
    Blob
      Name: img2.webp
```
</details>

<br>

> **Key takeaway :** User Delegation SAS works even with shared keys disabled (which was our Lab 1 configuration). This makes it the ideal choice for zero-trust environments.

<br>

## 5. Stored Access Policies

A Stored Access Policy is a named set of constraints (permissions, start/expiry times) defined on a container. You can then link a Service SAS to this policy instead of hardcoding the constraints in the SAS itself.  
See the Stored Access Policies as a set of predefined SAS parameters. So when we need to create a new SAS Token, we'll only have to specify a Stored Access Policy instead of manually setting the parameters (start / expiry times, permissions).

Why is this powerful ?
- **Revocation** : delete or modify the policy → all linked SAS tokens are immediately invalidated
- **Consistency** : centrally manage access rules for multiple SAS tokens
- **Flexibility** : change expiry or permissions without regenerating tokens

> **Important :** Each container can have a maximum of **5** stored access policies.

<br>

### 5.1 Create a Stored Access Policy via Azure Portal

1. Go to `teamadevstorage01` → **Data storage** → **Containers**
2. Click on the `documents` container
3. Click **"Access policy"** (left menu, under **Settings**)
4. Under **Stored access policies**, click **"+ Add policy"**
5. Configure :

| Setting | Value |
|---------|-------|
| Identifier | `documents-readonly` |
| Permissions | ☑️ Read ☑️ List |
| Start time | Now |
| Expiry time | +2 hours |

1. **"OK"**, then **"Save"**

<br>

### 5.2 Create via PowerShell

```powershell
$keys = Get-AzStorageAccountKey -ResourceGroupName "rg-storage-lab" -Name "teamadevstorage01"
$keyContext = New-AzStorageContext -StorageAccountName "teamadevstorage01" -StorageAccountKey $keys[0].Value

New-AzStorageContainerStoredAccessPolicy `
    -Container "documents" `
    -Policy "documents-readonly" `
    -Permission "rl" `
    -StartTime (Get-Date) `
    -ExpiryTime (Get-Date).AddHours(2) `
    -Context $keyContext
```

<br>

### 5.3 Verify the policy

```powershell
Get-AzStorageContainerStoredAccessPolicy -Container "documents" -Context $keyContext
```

<details>
<summary>Output</summary>

```
Policy             : documents-readonly
Permissions        : rl
StartTime          : 30/01/2026 03:30:00 +00:00
ExpiryTime         : 30/01/2026 05:30:00 +00:00
```
</details>

<br>

### 5.4 Generate a SAS linked to the policy

Now let's create a Service SAS that references our stored access policy instead of embedding permissions directly :

```powershell
$policySas = New-AzStorageContainerSASToken `
    -Name "documents" `
    -Policy "documents-readonly" `
    -Protocol HttpsOnly `
    -Context $keyContext

Write-Output "Policy-linked SAS: $policySas"
```

<details>
<summary>Output</summary>

```
Policy-linked SAS: sv=2025-07-05&spr=https&si=documents-readonly&sr=c&sig=GKih6w3gLYkmKQfk%2BXX0AdM%2FGHy7c%2FB38X5vSZ8ZMlY%3D
```
</details>

<br>

Notice the `si=documents-readonly` parameter : this is the policy identifier. The SAS token itself doesn't contain permissions or expiry, it references the policy instead.

### 5.5 Test the policy-linked SAS

```powershell
$docUrl = "https://teamadevstorage01.blob.core.windows.net/documents/doc1.html" + "?$policySas"
Invoke-WebRequest -Uri $docUrl
```

<details>
<summary>Output</summary>

```
StatusCode        : 200
StatusDescription : OK
Content           :  <!DOCTYPE html>
                    <html>
                    <head>
                    <title>AZ-104 Lab 2</title>
                    </head>
                    <body>

                    <h1>It's about storage</h1>
                    <p>And playing with access levels</p>

                    </body>
                    </html>
RawContent        : HTTP/1.1 200 OK
                    Accept-Ranges: bytes
                    ETag: "0x8DE5A357EB24DCC"
                    Server: Windows-Azure-Blob/1.0
                    Server: Microsoft-HTTPAPI/2.0
                    x-ms-request-id: 9a5bc5e7-f01e-00d7-019a-9192d8000000
                    x-ms-version: 20…
Headers           : {[Accept-Ranges, System.String[]], [ETag, System.String[]], [Server, System.String[]], [x-ms-request-id, System.String[]]…}
Images            : {}
InputFields       : {}
Links             : {}
RawContentLength  : 169
RelationLink      : {}
```
</details>

<br>

## 6. Revoking access

SAS tokens can't be individually revoked once generated. If a token is compromised, you need other mechanisms to invalidate it. Let's explore all the options.

### 6.1 Critical security limitation

 ⚠️ **MAJOR SECURITY CONCERN**

Azure does NOT track or list SAS tokens that have been generated. There is no way to:
- See which SAS tokens currently exist
- Know who generated a SAS token
- Know when a SAS token was created
- Know what permissions a SAS token has (unless you have the token itself to decode)
- List all active SAS tokens for audit purposes

This is a significant limitation because it's not possible :
- To audit SAS token usage retroactively
- To see if a former employee generated tokens before leaving
- To list all active access grants

**Mitigation strategies :**
- Always use **Stored Access Policies** when possible → you can at least see the policies (even if not the individual tokens)
- Use **User Delegation SAS** → tied to Azure AD identity, better audit trail via Azure AD logs
- Implement short expiry times (hours, not days/months)
- Use Azure Monitor and diagnostic logs to track SAS usage patterns (successful/failed authentications)
- Regularly rotate storage account keys (which invalidates all key-based SAS tokens)
- Disable shared key access and rely on Azure AD + User Delegation SAS exclusively
- Use Azure Policy to enforce SAS expiry limits and audit storage account configurations
- Implement a governance process where SAS generation is logged externally (e.g., via custom app logging)

**Real-world impact :**
Imagine a contractor generated 50 SAS tokens with 1-year expiry before their contract ended. There is no way to list those tokens. The only option is to regenerate thestorage account keys (which breaks all other key-based access) or wait for them to expire.

This is why Microsoft strongly recommends User Delegation SAS + Stored Access Policies + disabled shared keys for production environments.

<br>

### 6.2 Revocation methods overview

Since you can't see or individually revoke SAS tokens, you need indirect revocation methods :

| Method | Affects | When to Use | Limitations |
|--------|---------|-------------|-------------|
| **Delete / modify Stored Access Policy** | All SAS tokens linked to that policy | Compromised policy-linked SAS | Only works for Service SAS with `si=` parameter. Account SAS and ad-hoc Service SAS are unaffected |
| **Regenerate storage account key** | All Account SAS + Service SAS signed with that key | Major security breach, key compromise | Breaks all key-based access (connection strings, apps using keys). User Delegation SAS unaffected |
| **Revoke User Delegation Key** | All User Delegation SAS tokens | Compromised User Delegation SAS | Only affects User Delegation SAS. Requires Azure AD permissions |
| **Wait for expiry** | Single token | Token has short expiry, not urgent | Requires knowing when the token expires (not visible in Azure). Passive approach |

> **Exam tip :** Questions about SAS revocation often test whether you know that you **cannot** individually revoke a specific SAS token. The answer is always one of the indirect methods above. Also remember that **regenerating keys is disruptive** (breaks apps using those keys) while **deleting a Stored Access Policy is surgical** (only affects SAS tokens linked to that policy).

<br>

### 6.3 Revoke via Stored Access Policy (recommended)

Let's revoke access by modifying the stored access policy. This immediately invalidates all SAS tokens linked to it.

Via Azure Portal :
1. Go to `teamadevstorage01` → **Containers** → `documents`
2. Click **"Access policy"**
3. Find `documents-readonly` policy
4. Click the **"..."** menu → **"Delete"**
5. Click **"Save"**

Via PowerShell :
```powershell
Remove-AzStorageContainerStoredAccessPolicy `
    -Container "documents" `
    -Policy "documents-readonly" `
    -Context $keyContext
```

<br>

### 6.4 Verify revocation

Try accessing the blob with the same SAS token we generated in section 5.5 :

```powershell
$docUrl = "https://teamadevstorage01.blob.core.windows.net/documents/doc1.html" + "?$policySas"
Invoke-WebRequest -Uri $docUrl
```

<details>
<summary>Output</summary>

```
Invoke-WebRequest: Response status code does not indicate success: 403 (Server failed to authenticate the request.
Make sure the value of Authorization header is formed correctly including the signature.)
```
</details>

<br>

The SAS token is now invalid because the policy it referenced no longer exists.

<br>

### 6.5 Disable shared key access again

Now that we've completed our SAS exercises, let's re-disable shared key access as a security best practice :

```powershell
Set-AzStorageAccount -ResourceGroupName "rg-storage-lab" -Name "teamadevstorage01" -AllowSharedKeyAccess $false
```

<br>

## Key Takeaways

| Concept | Remember |
|---------|----------|
| **SAS hierarchy** | User Delegation > Service SAS with Stored Policy > Service SAS > Account SAS |
| **User Delegation SAS** | Most secure, signed with Azure AD, works without shared keys |
| **Account SAS** | Broadest scope, least secure, requires shared keys |
| **Stored Access Policy** | Named policy on container, enables SAS revocation, max 5 per container |
| **Revocation** | Delete policy, regenerate key, or revoke delegation key |
| **Shared keys** | Required for Account SAS and Service SAS, not for User Delegation SAS |

<br>

> **Exam tip :** Know the three SAS types and their signing methods. A common question pattern is "which SAS type should you use if the security policy requires Azure AD authentication?" → User Delegation SAS. Also remember that Stored Access Policies can only be attached to **Service SAS** (container or blob level), not Account SAS or User Delegation SAS.
