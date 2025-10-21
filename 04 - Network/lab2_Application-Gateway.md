https://learn.microsoft.com/en-us/azure/application-gateway/quick-create-powershell

We'll deploy the following architecture with Powershell:

![schema](.misc/network_lab2.png "schema")

We'll have an Application Gateway exposed to the Internet, listening on HTTP:80 and distributing the requests on 2 backend VMs.  

➡ Those VMs will be in an Availibility Set and on a different subnet than the Application Gateway, non-exposed to the Internet.  

➡ The Jump host VM-JUMP, with its NSG and Public IP, is just here to configure the two backends virtual machines, VM1 and VM2.  
It will run bash a script as a VM Extension, to install apache and configure a default page on the backend VMs, each with different color, so we can see on which VM the Loadbalancer lands us.


Cmdlets associated with the lab topic:
- [New-AzAvailabilitySet](https://)
- [New-AzNetworkSecurityRuleConfig](https://)
- [New-AzNetworkSecurityGroup](https://)
- [Set-AzVMExtension](https://)
- [New-AzApplicationGatewayIPConfiguration](https://)
- [New-AzApplicationGatewayFrontendIPConfig](https://)
- [New-AzApplicationGatewayFrontendPort](https://)
- [New-AzApplicationGatewayBackendAddressPool ](https://)
- [New-AzApplicationGatewayBackendHttpSetting](https://)
- [New-AzApplicationGatewayHttpListener](https://)
- [New-AzApplicationGatewayRequestRoutingRule](https://)
- [New-AzApplicationGatewaySku](https://)
- [New-AzApplicationGateway](https://)

---

### 1/ Resource group

```powershell
New-AzResourceGroup -Name "network_lab2" -Location "francecentral"
```  

Output :
```
ResourceGroupName : network_lab2
Location          : francecentral
ProvisioningState : Succeeded
Tags              :
ResourceId        : /subscriptions/<...>/resourceGroups/network_lab2
``` 

### 2/ Virtual network and subnet

```powershell
# Create the virtual network
$vnet = @{
    Name = 'vnet-2'
    ResourceGroupName = 'network_lab2'
    Location = 'francecentral'
    AddressPrefix = '192.168.0.0/16'
}

$virtualNetwork = New-AzVirtualNetwork @vnet

# Create the subnets in the previously created virtual network
$subnet = @{
    Name = 'subnet-ag'
    VirtualNetwork = $virtualNetwork
    AddressPrefix = '192.168.33.0/24'
}

$subnetAG = Add-AzVirtualNetworkSubnetConfig @subnet

$subnet = @{
    Name = 'subnet-vm'
    VirtualNetwork = $virtualNetwork
    AddressPrefix = '192.168.40.0/24'
}

$subnetVM = Add-AzVirtualNetworkSubnetConfig @subnet

# Apply the subnets configuration to the virtual network
$virtualNetwork | Set-AzVirtualNetwork

```  

Output :
``` 
$virtualNetwork | fl -Property Name, ResourceGroupName,Location, ProvisioningState, Type, Subnets

Name              : vnet-2
ResourceGroupName : network_lab2
Location          : francecentral
ProvisioningState : Succeeded
Type              : Microsoft.Network/virtualNetworks
Subnets           : {subnet-ag, subnet-vm}
```

### 3/ Backend servers

```powershell
# Availability set
$as = @{
    Name                        = 'av-set1'
    ResourceGroupName           = 'network_lab2'
    Location                    = 'francecentral'
    PlatformUpdateDomainCount   = 2
    PlatformFaultDomainCount    = 2
    Sku                         = 'Aligned'
}

$avSet = New-AzAvailabilitySet @as


# Backend Pool VMs
$vnet = Get-AzVirtualNetwork -Name 'vnet-2' -ResourceGroupName 'network_lab2'
$password = ConvertTo-SecureString -String "G3neriCP@ssw0rd!" -AsPlainText -Force
$user = "louis"
$cred = New-Object System.Management.Automation.PSCredential ($user, $password)

foreach ($i in 1..2) `
{
    $nic = @{
        Name              = "vm{0}-nic1" -f $i
        ResourceGroupName = 'network_lab2'
        Location          = 'francecentral'
        Subnet            = $vnet.Subnets | ?{ $_.Name -eq 'subnet-vm' }
    }
    
    $nicVM = New-AzNetworkInterface @nic

    $vmsz = @{
        VMName              = "vm{0}" -f $i
        VMSize              = 'Standard_B1ms' 
        AvailabilitySetId   = $avSet.Id
    }

    $vmos = @{
        ComputerName    = "vm{0}" -f $i
        Credential      = $cred
        DisablePasswordAuthentication   = $true
    }

    $vmimage = @{
        PublisherName = 'Canonical'
        Offer         = '0001-com-ubuntu-server-jammy'
        Skus          = '22_04-lts-gen2'
        Version       = 'latest'    
    }

    $vmdisk = @{
        Name                = "disk{0}" -f $i
        DiskSizeInGB        = 30
        StorageAccountType  = 'Standard_LRS'
        CreateOption        = 'FromImage'
    }

    $vmConfig = New-AzVMConfig @vmsz `
    | Set-AzVMOperatingSystem @vmos -Linux `
    | Set-AzVMSourceImage @vmimage `
    | Set-AzVMOSDisk @vmdisk `
    | Add-AzVMNetworkInterface -Id $nicVM.Id

    $vm = @{
        ResourceGroupName   = 'network_lab2'
        Location            = 'francecentral'
        SshKeyName          = 'sshkey_lolecat'
        VM                  = $vmConfig
    }

    New-AzVM @vm
}
```

### 4/ Jump VM and backend server configuration

```powershell
$vnet = Get-AzVirtualNetwork -Name 'vnet-2' -ResourceGroupName 'network_lab2'
$password = ConvertTo-SecureString -String "G3neriCP@ssw0rd!" -AsPlainText -Force
$user = "louis"
$cred = New-Object System.Management.Automation.PSCredential ($user, $password)

# Public IP
$ip = @{
    Name              = 'pip-jump'
    ResourceGroupName = 'network_lab2'
    Location          = 'francecentral'
    Sku               = 'Standard'
    AllocationMethod  = 'Static'
    IpAddressVersion  = 'IPv4'
    Zone              = 1, 2, 3
}
$pip = New-AzPublicIpAddress @ip

$nic = @{
    Name              = 'vm-jump-nic'
    ResourceGroupName = 'network_lab2'
    Location          = 'francecentral'
    SubnetId          = ($vnet.Subnets | ?{ $_.Name -eq 'subnet-vm' }).id
    PublicIpAddressId = $pip.id
} 
$nicVM = New-AzNetworkInterface @nic


# Jump VM
$vmsz = @{
    VMName  = "vm-jump"
    VMSize  = 'Standard_B1ms' 
}

$vmos = @{
    ComputerName                    = "vm-jump" 
    Credential                      = $cred
    DisablePasswordAuthentication   = $true
}

$vmimage = @{
    PublisherName = 'Canonical'
    Offer         = '0001-com-ubuntu-server-jammy'
    Skus          = '22_04-lts-gen2'
    Version       = 'latest'    
}

$vmdisk = @{
    Name                = "diskjump" 
    DiskSizeInGB        = 30
    StorageAccountType  = 'Standard_LRS'
    CreateOption        = 'FromImage'
}

$vmConfig = New-AzVMConfig @vmsz `
| Set-AzVMOperatingSystem @vmos -Linux `
| Set-AzVMSourceImage @vmimage `
| Set-AzVMOSDisk @vmdisk `
| Add-AzVMNetworkInterface -Id $nicVM.Id

$vm = @{
    ResourceGroupName   = 'network_lab2'
    Location            = 'francecentral'
    SshKeyName          = 'sshkey_lolecat'
    VM                  = $vmConfig
}
New-AzVM @vm


# NSG: SSH from my IP
$rule = @{
    Name                        = 'ssh-inbound'
    Description                 = 'allow ssh myIP'   
    Access                      = 'Allow'
    Direction                   = 'Inbound'
    SourceAddressPrefix         = "$(Invoke-RestMethod -Uri 'https://api.ipify.org')/32"
    SourcePortRange             = '*'
    DestinationAddressPrefix    = '*'
    DestinationPortRange        = 22
    Priority                    = 100
    Protocol                    = 'tcp'
}
$nsgConf = New-AzNetworkSecurityRuleConfig @rule

$nsg = @{
    ResourceGroupName   = 'network_lab2'
    Location            = 'francecentral'
    Name                = 'nsg-jump'
    SecurityRules       = $nsgConf
}

New-AzNetworkSecurityGroup @nsg

$nic = Get-AzNetworkInterface -Name 'vm-jump-nic' -ResourceGroupName 'network_lab2'
$nsg = Get-AzNetworkSecurityGroup -Name 'nsg-jump' -ResourceGroupName 'network_lab2'
$nic.NetworkSecurityGroup = $nsg
$nic | Set-AzNetworkInterface
```
<br/>

Next, since I configured authentication on backend VMs to support SSH with my public key, I copy my ssh keys to the jump VM  
Once copied, I disabled my private key passphrase on the vm-jump for testing / automation purpose (*ssh-keygen -p*)  
```powershell
$publicIP = (Get-AzPublicIpAddress -Name "pip-jump" -ResourceGroupName "network_lab2").IpAddress

cd ~\.ssh\
scp .\id_rsa .\id_rsa.pub louis@$($publicIp):/home/louis/.ssh
ssh louis@$($publicIp) "chmod 600 /home/louis/.ssh/id_rsa && chmod 644 /home/louis/.ssh/id_rsa.pub"

# Then connect to the vm-jump via ssh, and disable the private key passphrase with "ssh-keygen -p"
```
<br/>

When it's done, launch a [bash script](./.misc/network_lab2.sh) from the jump-vm with via a VM Extension, to configure the backend VMs.  
Each VM will run an Apache2 server, displaying a HelloWorld with a different background color depending on the server.  

```powershell
$settings = @{
    fileUris = @("https://raw.githubusercontent.com/LsLct/AZ-104/refs/heads/main/04%20-%20Network/.misc/network_lab2.sh")
    commandToExecute = "bash network_lab2.sh"
}

$params = @{
    ResourceGroupName   = 'network_lab2'
    VMName              = 'vm-jump'
    Name                = 'WebserversConfig'
    Publisher           = 'Microsoft.Azure.Extensions'
    ExtensionType       = 'CustomScript'
    TypeHandlerVersion  = '2.1'
    Settings            = $settings
    Location            = 'francecentral'
    ErrorAction         = 'Stop'
}

try 
{
    Set-AzVMExtension @params
    Write-Host "Webserver setup script executed on $($params.VMName)."
}
catch { Write-Error "Failed to execute script: $_" }
```

### 5/ Application Gateway

Now, the interesting part: the App Gateway deployment.
We'll need to:
- Create a Public IP Address
- Create the App Gateway Configuration: Subnet association, Frontend IP association, Backend pool association, Listener and Rule association
- Create the Application Gateway
- Add the VMs' NICs to the backend pool  
  
  
```powershell
# Public IP
$ip = @{
    Name              = 'pip-ag'
    ResourceGroupName = 'network_lab2'
    Location          = 'francecentral'
    Sku               = 'Standard'
    AllocationMethod  = 'Static'
    IpAddressVersion  = 'IPv4'
    Zone              = 1, 2, 3
}
$pip_ag = New-AzPublicIpAddress @ip


# Subnet Association
$sub_param = @{
    Name    = 'ag-subnet'
    Subnet = Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $(Get-AzVirtualNetwork -Name 'vnet-2' -ResourceGroupName 'network_lab2') -Name 'subnet-ag'
}
$gipconfig = New-AzApplicationGatewayIPConfiguration @sub_param


# Public IP association
$pip_param = @{
    Name            = 'pip-ag'
    PublicIPAddress = $pip_ag
}
$pipconfig = New-AzApplicationGatewayFrontendIPConfig @pip_param


# Frontend port assignation
$fport_param = @{
    Name    = 'ag-frontend'
    Port    = 80
}
$frontendport = New-AzApplicationGatewayFrontendPort @fport_param


# Backend pool definition
$pool_param = @{
    Name                = 'ag-pool'
    BackendIPAddresses = @(
        "$((Get-AzNetworkInterface -name 'vm1-nic1').IPConfigurations.PrivateIPAddress)", 
        "$((Get-AzNetworkInterface -name 'vm2-nic1').IPConfigurations.PrivateIPAddress)"
    )
}
$backendPool = New-AzApplicationGatewayBackendAddressPool @pool_param

$pool_param = @{
    Name                = 'ag-settings'
    Port                = 80
    Protocol            = 'Http'
    CookieBasedAffinity = 'Enabled'
    RequestTimeout      = 30
}
$poolSettings = New-AzApplicationGatewayBackendHttpSetting @pool_param


# Listener
$listener_param = @{
    Name                    = 'ag-listener'
    Protocol                = 'Http'
    FrontendIpConfiguration = $pipconfig
    FrontendPort            = $frontendport
}
$defaultlistener = New-AzApplicationGatewayHttpListener @listener_param

$frontrule_param = @{
    Name                = 'ag-frontrule'
    RuleType            = 'Basic'
    Priority            = 100
    HttpListener        = $defaultlistener
    BackendAddressPool  = $backendPool
    BackendHttpSettings = $poolSettings
}
$frontendRule = New-AzApplicationGatewayRequestRoutingRule @frontrule_param


# App Gateway creation
$sku_param = @{
    Name        = 'Standard_v2'
    Tier        = 'Standard_v2'
    Capacity    = 2
}
$sku = New-AzApplicationGatewaySku @sku_param

$bigBeautifulParam = @{
    Name                            = 'ag1'
    ResourceGroupName               = 'network_lab2'
    Location                        = 'francecentral'
    BackendAddressPools             = $backendPool
    BackendHttpSettingsCollection   = $poolSettings
    FrontendIpConfigurations        = $pipconfig
    GatewayIpConfigurations         = $gipconfig
    FrontendPorts                   = $frontendport
    HttpListeners                   = $defaultlistener
    RequestRoutingRules             = $frontendRule
    Sku                             = $sku
}
New-AzApplicationGateway @bigBeautifulParam
```
<br/>

Same code, different writing, just testing various syntaxes.  
I think I prefer this one, more compact and somehow more readable.

```powershell
# Public IP Address
$ip = @{
    Name              = 'pip-ag'
    ResourceGroupName = 'network_lab2'
    Location          = 'francecentral'
    Sku               = 'Standard'
    AllocationMethod  = 'Static'
    IpAddressVersion  = 'IPv4'
    Zone              = 1, 2, 3
}
$pip_ag = New-AzPublicIpAddress @ip

# Subnet Association
$vnet = Get-AzVirtualNetwork -Name 'vnet-2' -ResourceGroupName 'network_lab2'
$subnet = Get-AzVirtualNetworkSubnetConfig -Name 'subnet-ag' -VirtualNetwork $vnet
$gipconfig = New-AzApplicationGatewayIPConfiguration -Name 'ag-subnet' -Subnet $subnet

# Public IP Association
$pipconfig = New-AzApplicationGatewayFrontendIPConfig -Name 'pip-ag' -PublicIPAddress $pip_ag

# Frontend Port
$frontendport = New-AzApplicationGatewayFrontendPort -Name 'ag-frontend' -Port 80

# Backend Pool (added ResourceGroupName for NIC retrieval)
$nic1 = Get-AzNetworkInterface -Name 'vm1-nic1' -ResourceGroupName 'network_lab2'
$nic2 = Get-AzNetworkInterface -Name 'vm2-nic1' -ResourceGroupName 'network_lab2'
$backendPool = New-AzApplicationGatewayBackendAddressPool -Name 'ag-pool' -BackendIPAddresses @($nic1.IPConfigurations.PrivateIPAddress, $nic2.IPConfigurations.PrivateIPAddress)

# Backend HTTP Settings
$poolSettings = New-AzApplicationGatewayBackendHttpSetting -Name 'ag-settings' -Port 80 -Protocol Http -CookieBasedAffinity Enabled -RequestTimeout 30

#  Listener
$defaultlistener = New-AzApplicationGatewayHttpListener -Name 'ag-listener' -Protocol Http -FrontendIpConfiguration $pipconfig -FrontendPort $frontendport

# Routing Rule
$frontendRule = New-AzApplicationGatewayRequestRoutingRule -Name 'ag-frontrule' -RuleType Basic -Priority 100 -HttpListener $defaultlistener -BackendAddressPool $backendPool -BackendHttpSettings $poolSettings

# SKU
$sku = New-AzApplicationGatewaySku -Name 'Standard_v2' -Tier 'Standard_v2' -Capacity 2

# Application Gateway creation
$agParams = @{
    Name                          = 'ag1'
    ResourceGroupName             = 'network_lab2'
    Location                      = 'francecentral'
    BackendAddressPools           = $backendPool
    BackendHttpSettingsCollection = $poolSettings
    FrontendIpConfigurations      = $pipconfig
    GatewayIpConfigurations       = $gipconfig
    FrontendPorts                 = $frontendport
    HttpListeners                 = $defaultlistener
    RequestRoutingRules           = $frontendRule
    Sku                           = $sku
}

try 
{
    $ag = New-AzApplicationGateway @agParams
    Write-Output "Application Gateway '$($ag.Name)' successfully created."
    Write-Output "Public IP: $($pip_ag.IpAddress)"
}
catch { Write-Error "Failed to create Application Gateway: $_" }
```

### 6/ Testing

```powershell
Invoke-WebRequest -Uri $((Get-AzPublicIpAddress -Name 'pip-ag').IpAddress)
```

We can request the application gateway more than once, and see that it is received by the different backend servers, output:
```
PS C:\Users\Louis> Invoke-WebRequest -Uri $((Get-AzPublicIpAddress -Name 'pip-ag').IpAddress)

StatusCode        : 200
StatusDescription : OK
Content           : <html>
                    <body style="background-color: green">
                        <h1 style="color: white">Hello world</h1>
                    </body>
                    </html>

RawContent        : HTTP/1.1 200 OK
                    Date: Thu, 09 Oct 2025 12:54:19 GMT
                    Connection: keep-alive
                    Set-Cookie: ApplicationGatewayAffinity=207acb22edcd3b13e9ae6d429a569e62; Path=/
                    Server: Apache/2.4.52
                    Server: (Ubuntu)
                    …

PS C:\Users\Louis> Invoke-WebRequest -Uri $((Get-AzPublicIpAddress -Name 'pip-ag').IpAddress)

StatusCode        : 200
StatusDescription : OK
Content           : <html>
                    <body style="background-color: blue">
                        <h1 style="color: white">Hello world</h1>
                    </body>
                    </html>

RawContent        : HTTP/1.1 200 OK
                    Date: Thu, 09 Oct 2025 12:55:02 GMT
                    Connection: keep-alive
                    Set-Cookie: ApplicationGatewayAffinity=64ee104cd068c4904863674e232dd3a0; Path=/
                    Server: Apache/2.4.52
                    Server: (Ubuntu)
                    …
```

### 7/ Cleaning

```powershell
Get-AzResourceGroup -Name "network_lab2" | Remove-AzResourceGroup -Force
```