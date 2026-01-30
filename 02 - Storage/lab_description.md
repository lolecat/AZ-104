# AZ-104 - Implement and Manage Storage Labs

### 1. Storage Account Creation & Configuration

Create and configure a Storage Account in the ***Team A - Dev*** and ***Team B - Dev*** subscriptions following enterprise compliance requirements:

* Data must be replicated to another region for redundancy (minimize costs)
* Block public access to all blobs or containers
* Disable shared key access
* Allow HTTPS traffic only
* Minimum TLS version: 1.2
* Hot tier as the default access tier

➡️ [Solution here](./lab_part1.md)

---

### 2. Storage Redundancy & Object Replication

Configure storage redundancy and set up cross-region object replication :

1. Understand all redundancy options (LRS, ZRS, GRS, RA-GRS, GZRS, RA-GZRS)
2. Change redundancy on the ***teambdevstorage01*** account : switch to ***RA-GRS***
3. Create a new storage account in ***Team A - Prod*** subscription
4. Configure Object Replication from ***Team A Prod*** to ***Team A - Dev*** for backup purposes
5. Understand failover scenarios and RPO/RTO

➡️ [Solution here](./lab_part2.md)

---

### 3. Blob Storage & Containers

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

➡️ [Solution here](./lab_part3.md)

---

### 4. Access Tiers & Lifecycle Management

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

➡️ [Solution here](./lab_part4.md)

---

### 5. SAS Tokens & Stored Access Policies

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

➡️ [Solution here](./lab_part5.md)

---

### 6. [In Progress] Azure Files & File Shares

Deploy and configure Azure Files for cloud file sharing:

* Create file shares with quotas and tiers
* Mount file shares on Windows and Linux
* Configure identity-based authentication
* Create and restore snapshots
* Enable soft delete for file shares

➡️ [Solution here](./lab_part6.md)

---

### 7. [In Progress] Storage Security

Implement advanced security for storage accounts:

* Configure Storage Firewalls and Virtual Network rules
* Deploy Private Endpoints for storage
* Compare Service Endpoints vs Private Endpoints
* Configure customer-managed keys (CMK)
* Enable infrastructure encryption

➡️ [Solution here](./lab_part7.md)

---

### 8. [In Progress] Data Management Tools

Master data management and protection tools:

* Install and use AzCopy for data migration
* Use Azure Storage Explorer for GUI-based management
* Configure soft delete for blobs and containers
* Enable blob versioning
* Configure point-in-time restore

➡️ [Solution here](./lab_part8.md)