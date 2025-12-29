# 1. Users and groups creation, admin access management

**Lab instructions reminder :**  
Create 10 users, split into 5 groups :

* 1 user in Super Admin (tenant creator user)
* 1 user in Subscriptions Manager (existing user)
* 5 users in Team A
* 5 users in Team B

Users must be assigned permissions via groups only.

> Since I'm too lazy to take screenshots and paste them here, I'll do most of my labs with Powershell.

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

### 1. Create the App Registration
In the Azure Portal, go to **Entra ID** -> **App registration** -> **New registration**.  
From there, give a name to our registration, here ***"graph_automation_lab"***.  
 Keep the rest as default (single tenant access / no Redirect URI)

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
### 3. Associate the certificate to the Registered App in Azure
In the Azure Portal, go to **Entra ID** -> **App registration** -> **graph_automation_lab** -> **manage** -> **certificates and secrets**.  
Click on **certificates** tab, and **upload certificate**.
Then upload the previously created certificate (how unexpected !)

### 4. Give the right permissions to the Registered App
**API permissions** → **Add a permission** → **Microsoft Graph**

| API / Permissions name  | Type  | Description  | Admin consent required  |
|---|---|---|---|
| User.ReaWrite.All  |  Application | Read and write all users' full profiles  |  Yes |

Once the permission added, click on the **"grant admin consent"** tab to validate the changes. 


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

## 2. Groups creation

We need to create 4 groups : 
- Super admins
- Subscriptions managers
- Team A
- Team B