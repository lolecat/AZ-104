https://learn.microsoft.com/en-us/azure/private-link/create-private-endpoint-powershell?tabs=dynamic-ip

We'll deploy the following architecture with Powershell:

![schema](.misc/network_lab3.png "schema")  

➡ This lab setup is very similar to the one in [lab1](./lab1_Bastion.md), except that :
- we'll have only one VM in the ***subnet-1***
- and we'll add a private endpoint pointing to the webapp deployed in the [compute_lab1](../03%20-%20Compute/lab1_AppService)  

  
Cmdlets used in this lab:
- [text](https://)
- [text](https://)
- [text](https://)
- [text](https://)

---

### 1/ Initial setup :

Since the architecture is very similar to the one we have in lab 1, let's deploy this quick in one block :  
```powershell
$rg = "network_lab3"
$location = "francecentral"
$vnet = "vnet-3"

New-AzResourceGroup -Name $rg -Location $location

# Virtual network
$vnet_param = @{
    Name = $vnet
    ResourceGroupName = $rg
    Location = $location
    AddressPrefix = '192.168.0.0/16'
}
$virtualNetwork = New-AzVirtualNetwork @vnet_param

# VMs subnet
$subnet_param = @{
    Name = 'subnet-1'
    VirtualNetwork = $virtualNetwork
    AddressPrefix = '192.168.40.0/24'
}
$subnetConfig = Add-AzVirtualNetworkSubnetConfig @subnet_param
$virtualNetwork | Set-AzVirtualNetwork

# Bastion subnet
$bastion_subnet = @{
    Name = 'AzureBastionSubnet'
    VirtualNetwork = $virtualNetwork
    AddressPrefix = '192.168.1.0/26'
}
$subnetConfig = Add-AzVirtualNetworkSubnetConfig @bastion_subnet
$virtualNetwork | Set-AzVirtualNetwork

# Bastion Public IP
$ip = @{
        ResourceGroupName = $rg
        Name = 'public-ip'
        Location = $location
        AllocationMethod = 'Static'
        Sku = 'Standard'
        Zone = 1,2,3
}
New-AzPublicIpAddress @ip

# VM
$securePassword = ConvertTo-SecureString -String "P@ssw0rd33!" -AsPlainText -Force
$user = "louis"
$cred = New-Object System.Management.Automation.PSCredential ($user, $securePassword)
$virtualNetwork = Get-AzVirtualNetwork -ResourceGroupName $rg -Name $vnet

$nic_param = @{
    Name              = 'vm1-nic1'
    ResourceGroupName = $rg
    Location          = $location
    Subnet            = $virtualNetwork.Subnets | ?{$_.Name -eq 'subnet-1'}
}
$nicVM = New-AzNetworkInterface @nic_param

$vmsz = @{
    VMName = 'vm1'
    VMSize = 'Standard_B2s_v2'  
}

$vmos = @{
    ComputerName = 'vm1'
    Credential   = $cred
}

$vmimage = @{
    PublisherName = 'Canonical'
    Offer         = '0001-com-ubuntu-server-jammy'
    Skus          = '22_04-lts-gen2'
    Version       = 'latest'    
}

$vmConfig = New-AzVMConfig @vmsz `
    | Set-AzVMOperatingSystem @vmos -Linux `
    | Set-AzVMSourceImage @vmimage `
    | Add-AzVMNetworkInterface -Id $nicVM.Id

$vm = @{
    ResourceGroupName = $rg
    Location          = $location
    VM                = $vmConfig
}
New-AzVM @vm

# Bastion host
$bastion = @{
    Name = 'bastion'
    ResourceGroupName = $rg
    PublicIpAddressRgName = $rg
    PublicIpAddressName = 'public-ip'
    VirtualNetworkRgName = $rg
    VirtualNetworkName = $vnet
    Sku = 'Basic'
}
New-AzBastion @bastion
```  

