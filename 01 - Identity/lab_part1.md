# 1. Users and groups creation, admin access management

**Lab instructions reminder :**  
Create 10 users, split into 5 groups :

* 1 user in Super Admin (tenant creator user)
* 1 user in Subscriptions Manager (existing user)
* 5 users in Team A
* 5 users in Team B

Users must be assigned permissions via groups only.

> Since I'm too lazy to take screenshots and paste them here, I'll do most of my labs with Powershell.

Cmdlets used in this lab:
- [Connect-MgGraph](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/connect-mggraph?view=graph-powershell-1.0)
- [Disconnect-MgGraph](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/disconnect-mggraph?view=graph-powershell-1.0#related-links)
- [New-SelfSignedCertificate](https://learn.microsoft.com/en-us/powershell/module/pki/new-selfsignedcertificate?view=windowsserver2025-ps)
- [New-MgUser](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/new-mguser?view=graph-powershell-1.0)
- [Get-MgUser](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguser?view=graph-powershell-1.0)
- [New-MgGroup](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.groups/new-mggroup?view=graph-powershell-1.0)
- [New-MgGroupMember](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.groups/new-mggroupmemberbyref?view=graph-powershell-1.0)

<br>

## 1. Users creation  
For this first step, we'll create users via a bulk operation.  
In order to do it, we'll need the csv template, which is downloadable in the Entra ID portal. From there, go to the side blade -> **Manage** -> **Users**, and in the **Users management panel**, select **bulk operations**, then **bulk create**. From here, we can dowload the csv template file.  
I put this file [here](./.misc/UserCreateTemplate.csv).  
And the file populated with lab's users [here](./.misc/lab1_users.csv).

Here's the Powershell code provided by Microsoft for creating users based on a CSV :
```powershell
# Import the Microsoft Graph module 
Import-Module Microsoft.Graph 

# Authenticate to Microsoft Graph (you may need to provide your credentials) 
Connect-MgGraph -Scopes "User.ReadWrite.All" 

# Specify the path to the CSV file containing user data 
$csvFilePath = "C:\\Path\\To\\Your\\Users.csv" 

# Read the CSV file (adjust the column names as needed) 
$usersData = Import-Csv -Path $csvFilePath 

# Loop through each row in the CSV and create users \
foreach ($userRow in $usersData) { 
    $userParams = @{ 
        DisplayName = $userRow.'Name [displayName] Required' 
        UserPrincipalName = $userRow.'User name [userPrincipalName] Required' 
        PasswordProfile = @{ 
            Password = $userRow.'Initial password [passwordProfile] Required' 
        } 
        AccountEnabled = $true 
        MailNickName = $userRow.mailNickName 
    } 
    try { 
        New-MgUser @userParams 
        Write-Host "User $($userRow.UserPrincipalName) created successfully." 
    } catch { 
        Write-Host "Error creating user $($userRow.UserPrincipalName): $($_.Exception.Message)" 
    } 
} 

# Disconnect from Microsoft Graph 
Disconnect-MgGraph 

Write-Host "Bulk user creation completed." 
```
It won't work, because we won't be able to connect through Graph. Why ? Microsoft blocks high-privilege Graph operations for delegated users when Security Defaults are enabled. 0 trust / least privilege philosophy. 
2 Otpions here :
- Disable the "Security defaults" in the tenant's properties (OK for a lab, not OK for production)
- Use an App Registration and associate a certificate to it in order to connect

We're in a lab environment ... so of course we'll chose the 2nd option 🙃

<br>

### 1. Create the App Registration
In the Azure Portal, go to **Entra ID** -> **App registration** -> **New registration**.  
From there, give a name to our registration, here ***"graph_automation_lab"***.  
 Keep the rest as default (single tenant access / no Redirect URI)

<br>

 ### 2. Generate a certificate
 On our local machine, generate a self-signed certificate.  
 1 year validity / stored in the user personnal store.
 ```powershell
 $cert = New-SelfSignedCertificate `
  -Subject "CN=graph_automation_lab" `
  -CertStoreLocation "Cert:\CurrentUser\My" `
  -KeySpec KeyExchange `
  -NotAfter (Get-Date).AddYears(1)

Export-Certificate `
  -Cert $cert `
  -FilePath "~\Desktop\graph_cert.cer"
 ```
<br>

### 3. Associate the certificate to the Registered App in Azure
In the Azure Portal, go to :  
 **Entra ID** -> **App registration** -> **graph_automation_lab** -> **manage** -> **certificates and secrets**.  
Click on **certificates** tab, and **upload certificate**.
Then upload the previously created certificate (how unexpected !)

<br>

### 4. Give the right permissions to the Registered App
**API permissions** → **Add a permission** → **Microsoft Graph**

| API / Permissions name  | Type  | Description  | Admin consent required  |
|---|---|---|---|
| User.ReaWrite.All  |  Application | Read and write all users' full profiles  |  Yes |

Once the permission added, click on the **"grant admin consent"** tab to validate the changes. 

<br>

### 5. Connect with Powershell

Then, in the Registered App overview, get the client ID and Tenant ID, there are gonna be used along with the certificate thumbprint to connect to our Tenant with Graph.  

```powershell
Connect-MgGraph `
>   -TenantId "xxxx-xxxx-xxxx" `
>   -ClientId "xxxx-xxxx-xxxx" `
>   -CertificateThumbprint $cert.Thumbprint

Welcome to Microsoft Graph!

Connected via apponly access using xxxx-xxxx-xxxx
Readme: https://aka.ms/graph/sdk/powershell
SDK Docs: https://aka.ms/graph/sdk/powershell/docs
API Docs: https://aka.ms/graph/docs

NOTE: You can use the -NoWelcome parameter to suppress this message.
NOTE: Sign in by Web Account Manager (WAM) is enabled by default on Windows systems and cannot be disabled. Any setting stating otherwise will be ignored.
```

What an adventure !

So now the actually working code :
```powershell
# Import the Microsoft Graph module -> Not necessary
# Import-Module Microsoft.Graph

# Get the previously created certificate's thumprint
$certTp = (gci Cert:\CurrentUser\My\ | ?{$_.Subject -like "*graph*"}).Thumbprint

# Authenticate to Microsoft Graph (you may need to provide your credentials) 
Connect-MgGraph -TenantId "xxxx-xxxx-xxxx" -ClientId "xxxx-xxxx-xxxx" -CertificateThumbprint $certTp

# Specify the path to the CSV file containing user data 
$csvFilePath = (Get-Item -Path .\lab1_users.csv).FullName

# Read the CSV file
$usersData = Import-Csv -Path $csvFilePath -Delimiter ","

# Loop through each row in the CSV and create users
foreach ($userRow in $usersData) 
{ 
    $userParams = @{ 
        DisplayName = $userRow.'Name [displayName] Required' 
        UserPrincipalName = $userRow.'User name [userPrincipalName] Required' 
        PasswordProfile = @{ 
            Password = $userRow.'Initial password [passwordProfile] Required' 
        } 
        AccountEnabled = $true 
        MailNickName = $userRow.'Mail nickname [mailNickname] Required'
    } 
    
    try 
    { 
        New-MgUser @userParams 
        Write-Host "User $($userRow.UserPrincipalName) created successfully." 
    } 

    catch 
    { 
        Write-Host "Error creating user $($userRow.UserPrincipalName): $($_.Exception.Message)" 
    } 
} 
```
Verify
```
# Get-MgUser

DisplayName               Id               UserPrincipalName
-----------               --               -----------------
François Pignon           xxx-xxx-xxx      fpignon@lolecat.me
Harry Cover               xxx-xxx-xxx      hcover@lolecat.me
Homère Dalor              xxx-xxx-xxx      hdalor@lolecat.me
Jean Bombeur              xxx-xxx-xxx      jbombeur@lolecat.me
Jean Tanrien              xxx-xxx-xxx      jtanrien@lolecat.me
Louis Lecat (Super Admin) xxx-xxx-xxx      louis@lecatlouisoutlook.onmicrosoft.com
Louis Lecat               xxx-xxx-xxx      louis@lolecat.me
Mamadou Blanal            xxx-xxx-xxx      mblanal@lolecat.me
Roland Khülé              xxx-xxx-xxx      rkhule@lolecat.me
Sarah Pelle               xxx-xxx-xxx      spelle@lolecat.me
Sarah Pelpu               xxx-xxx-xxx      spelpu@lolecat.me
Yves Rogne                xxx-xxx-xxx      yrogne@lolecat.me
```

<br>

## 2. Groups creation

We need to create 4 groups : 
- Super admins
- Subscriptions managers
- Team A
- Team B

Now that we can connect with the Graph module, this step should be easy. Let's do this with this Powershell block, where I'm creating an array of objects, where each objects contains the group's name and its direct members.  


```powershell
# Import the Microsoft Graph module -> Not necessary
# Import-Module Microsoft.Graph

# Get the previously created certificate's thumprint
$certTp = (gci Cert:\CurrentUser\My\ | ?{$_.Subject -like "*graph*"}).Thumbprint

# Authenticate to Microsoft Graph (you may need to provide your credentials) 
Connect-MgGraph -TenantId "xxxx-xxxx-xxxx" -ClientId "xxxx-xxxx-xxxx" -CertificateThumbprint $certTp

# Create an array of hashtables, containing groups names and members
$groups = @(
    @{Name="Super Admins"; Members=@("louis@lecatlouisoutlook.onmicrosoft.com")},
    @{Name="Subscriptions Managers"; Members=@("louis@lolecat.me")},
    @{Name="Team A"; Members=@("jtanrien@lolecat.me","jbombeur@lolecat.me","hdalor@lolecat.me","rkhule@lolecat.me","yrogne@lolecat.me")},
    @{Name="Team B"; Members=@("hcover@lolecat.me","fpignon@lolecat.me","spelle@lolecat.me","spelpu@lolecat.me","mblanal@lolecat.me")}
)

$groups | % `
{
    $newGroup = New-MgGroup -DisplayName $_.Name `
        -MailEnabled:$false `
        -SecurityEnabled:$true `
        -MailNickname ($_.Name -replace ' ','')

    # Give some time  to Azure, let's not rush him
    Start-Sleep -Seconds 5

    foreach ($member in $_.Members)` 
    {
        $user = Get-MgUser -Filter "userPrincipalName eq '$member'"
        New-MgGroupMember -GroupId $newGroup.Id -DirectoryObjectId $user.Id
    }
}
```

Once more, it won't be that easy ! :)
Here's one of the errors we'll get if we try to execute the previous code :
```powershell
   ErrorIndex: 4

Exception             : 
    Type    : System.Exception
    Message : [Authorization_RequestDenied] : Insufficient privileges to complete the operation.
    HResult : -2146233088
TargetObject          : { Headers = , body = Microsoft.Graph.PowerShell.Models.MicrosoftGraphGroup }     
CategoryInfo          : InvalidOperation: ({ Headers = , body …crosoftGraphGroup
}:<>f__AnonymousType2`2) [New-MgGroup_CreateExpanded], Exception
FullyQualifiedErrorId : 
Authorization_RequestDenied,Microsoft.Graph.PowerShell.Cmdlets.NewMgGroup_CreateExpanded
ErrorDetails          : Insufficient privileges to complete the operation.

                        Status: 403 (Forbidden)
                        ErrorCode: Authorization_RequestDenied
                        Date: 2025-12-30T02:31:45

                        Headers:
                        Cache-Control                 : no-cache
                        Vary                          : Accept-Encoding
                        Strict-Transport-Security     : max-age=31536000
                        request-id                    : f120cfbe-d18c-43bb-9d8d-4089afaed543
                        client-request-id             : 9972a30c-9a0b-4328-8b13-8296ecebf6f2
                        x-ms-ags-diagnostic           : {"ServerInfo":{"DataCenter":"Southeast
Asia","Slice":"E","Ring":"5","ScaleUnit":"001","RoleInstance":"SI2PEPF00001643"}}
                        x-ms-resource-unit            : 1
                        Date                          : Tue, 30 Dec 2025 02:31:45 GMT


InvocationInfo        : 
    MyCommand        : New-MgGroup_CreateExpanded
    ScriptLineNumber : 3
    OffsetInLine     : 5
    HistoryId        : 17
    Line             :     $newGroup = New-MgGroup -DisplayName $_.Name -MailEnabled:$false
-SecurityEnabled:$true -MailNickname ($_.Name -replace ' ','')

    Statement        : $newGroup = New-MgGroup -DisplayName $_.Name -MailEnabled:$false
-SecurityEnabled:$true -MailNickname ($_.Name -replace ' ','')
    PositionMessage  : At line:3 char:5
                       +     $newGroup = New-MgGroup -DisplayName $_.Name -MailEnabled:$false  …
                       +     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    InvocationName   : New-MgGroup
    CommandOrigin    : Internal
ScriptStackTrace      : at New-MgGroup<Process>, C:\Users\Louis\Documents\PowerShell\Modules\Microsoft.G 
raph.Groups\2.34.0\exports\ProxyCmdletDefinitions.ps1: line 55720
                        at <ScriptBlock>, <No file>: line 3
                        at <ScriptBlock>, <No file>: line 1
PipelineIterationInfo : 
      0
      1
```

Why ? Simply because previoulsly, when we added our App Registration for Graph, we granted it only the ***User.ReadWrite.All*** permission. Not group !  
Following the 1.1.4 point of this lab, simply add the following permissions to the ***graph_automation_lab*** application in the App Registration panel :
- *Group.ReadWrite.All*
- *GroupMember.ReadWrite.All*

Should be better now :)