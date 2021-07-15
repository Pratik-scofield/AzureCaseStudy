####################Resource Group#################################

$rg = @{
    Name = 'Casestudy-SEA'
    Location = 'southeastasia'
}
New-AzResourceGroup @rg

$vnet = @{
    Name = 'CSVNet'
    ResourceGroupName = 'Casestudy-SEA'
    Location = 'southeastasia'
    AddressPrefix = '172.1.0.0/16'    
}
$virtualNetwork = New-AzVirtualNetwork @vnet

$subnet = @{
    Name = 'websubnet'
    VirtualNetwork = $virtualNetwork
    AddressPrefix = '172.1.1.0/24'
}
$subnetConfig = Add-AzVirtualNetworkSubnetConfig @subnet

$subnet = @{
    Name = 'mgmtsubnet'
    VirtualNetwork = $virtualNetwork
    AddressPrefix = '172.1.2.0/24'
}
$subnetConfig = Add-AzVirtualNetworkSubnetConfig @subnet

$virtualNetwork | Set-AzVirtualNetwork

##public IP and Loadbalancer###

$publicip = @{
    Name = 'SEAPublicIP'
    ResourceGroupName = 'Casestudy-SEA'
    Location = 'southeastasia'
    Sku = 'standard'
    AllocationMethod = 'static'
    Zone = 1,2,3
}
New-AzPublicIpAddress @publicip

$publicIp = Get-AzPublicIpAddress -Name 'SEAPublicIP' -ResourceGroupName 'Casestudy-SEA'

$feip = New-AzLoadBalancerFrontendIpConfig -Name 'FrontEnd' -PublicIpAddress $publicIp


$bepool = New-AzLoadBalancerBackendAddressPoolConfig -Name 'BackEndPool'


$probe = @{
    Name = 'HealthProbe1'
    Protocol = 'TCP'
    Port = '3389'
    IntervalInSeconds = '360'
    ProbeCount = '5'
    RequestPath = '/'
}
$healthprobe = New-AzLoadBalancerProbeConfig @probe


$lbrule =@{
    Name = 'RDPRule'
    Protocol = 'tcp'
    FrontendPort = '8000'
    BackendPort = '3389'
    IdleTimeoutInMinutes = '15'
    FrontendIpConfiguration = $feip
    BackendAddressPool = $bePool
}


$rule = New-AzLoadBalancerRuleConfig @lbrule -EnableTcpReset -DisableOutboundSNAT


$loadbalancer = @{
    ResourceGroupName = 'Casestudy-SEA'
    Name = 'LoadBalancer'
    Location = 'southeastasia'
    Sku = 'Standard'
    FrontendIpConfiguration = $feip
    BackendAddressPool = $bePool
    LoadBalancingRule = $rule
    Probe = $healthprobe
}

New-AzLoadBalancer @loadbalancer

#############NAT Rules####

$slb = Get-AzLoadBalancer -Name "LoadBalancer" -ResourceGroupName "Casestudy-SEA"
$slb | Add-AzLoadBalancerInboundNatRuleConfig -Name "NatRule" -FrontendIPConfiguration $slb.FrontendIpConfigurations[0] -Protocol "Tcp" -FrontendPort 3350 -BackendPort 3389 -EnableFloatingIP
$slb | Set-AzLoadBalancer



#########WEB NSG###############

$rule1 = New-AzNetworkSecurityRuleConfig -Name rdp-rule -Description "Allow RDP" `
    -Access Allow -Protocol Tcp -Direction Inbound -Priority 100 -SourceAddressPrefix `
    Internet -SourcePortRange * -DestinationAddressPrefix * -DestinationPortRange 3389

$rule2 = New-AzNetworkSecurityRuleConfig -Name web-rule -Description "Allow HTTP" `
    -Access Allow -Protocol Tcp -Direction Inbound -Priority 101 -SourceAddressPrefix `
    Internet -SourcePortRange * -DestinationAddressPrefix * -DestinationPortRange 80

$rule3 = New-AzNetworkSecurityRuleConfig -Name web-rule1 -Description "Allow HTTPS" `
    -Access Allow -Protocol Tcp -Direction Inbound -Priority 102 -SourceAddressPrefix `
    Internet -SourcePortRange * -DestinationAddressPrefix * -DestinationPortRange 443

$rule4 = New-AzNetworkSecurityRuleConfig -Name LB-rule -Description "Allow LB" `
    -Access Allow -Protocol Tcp -Direction Inbound -Priority 103 -SourceAddressPrefix `
    AzureLoadBalancer -SourcePortRange * -DestinationAddressPrefix VirtualNetwork -DestinationPortRange *

$nsg = New-AzNetworkSecurityGroup -ResourceGroupName Casestudy-SEA -Location southeastasia -Name `
    "WebNSG" -SecurityRules $rule1,$rule2,$rule3,$rule4

####mgnt NSG##############
$rule1 = New-AzNetworkSecurityRuleConfig -Name rdp-rule -Description "Allow RDP" `
    -Access Allow -Protocol Tcp -Direction Inbound -Priority 100 -SourceAddressPrefix `
    Internet -SourcePortRange * -DestinationAddressPrefix * -DestinationPortRange 3389


$rule2 = New-AzNetworkSecurityRuleConfig -Name LB-rule -Description "Allow LB" `
    -Access Allow -Protocol Tcp -Direction Inbound -Priority 101 -SourceAddressPrefix `
    AzureLoadBalancer -SourcePortRange * -DestinationAddressPrefix VirtualNetwork -DestinationPortRange *

$nsg = New-AzNetworkSecurityGroup -ResourceGroupName Casestudy-SEA -Location southeastasia -Name `
    "mgmtNSG" -SecurityRules $rule1,$rule2


$nsg = Set-AzureNetworkSecurityGroupToSubnet -Name WebNSG -VirtualNetworkName "CSVNet" -SubnetName "websubnet"  - manual
$nsg= Get-AzureNetworkSecurityGroupAssociation -VirtualNetworkName "CSVNet" -SubnetName "websubnet


###############Create VM####################################### -- Default size is not available

New-AzAvailabilitySet `
   -Location "westus2" `
   -Name "CSAvailabilitySet" `
   -ResourceGroupName "Casestudy-SEA" `
   -Sku aligned `
   -PlatformFaultDomainCount 2 `
   -PlatformUpdateDomainCount 2

$cred = Get-Credential

for ($i=1; $i -le 2; $i++)
{
    New-AzVm `
        -ResourceGroupName "CSAvailabilitySet" `
        -Name "VM1$i" `
        -Location "southeastasia" `
        -VirtualNetworkName "CSVnet" `
        -SubnetName "websubnet" `
        -SecurityGroupName "webNSG" `
        -PublicIpAddressName "publicIp$i" `
        -AvailabilitySetName "CSAvailabilitySet" `
        -Credential $cred
}

$cred = Get-Credential

for ($i=1; $i -le 2; $i++)
{
    New-AzVm `
        -ResourceGroupName "CSAvailabilitySet" `
        -Name "VM2$i" `
        -Location "southeastasia" `
        -VirtualNetworkName "CSVnet" `
        -SubnetName "websubnet" `
        -SecurityGroupName "webNSG" `
        -PublicIpAddressName "publicIp$i" `
        -AvailabilitySetName "CSAvailabilitySet" `
        -Credential $cred
}

$cred = Get-Credential

{
    New-AzVm `
        -ResourceGroupName "CSAvailabilitySet" `
        -Name "VM3" `
        -Location "southeastasia" `
        -VirtualNetworkName "CSVnet" `
        -SubnetName "mgmtsubnet" `
        -SecurityGroupName "webNSG" `
        -PublicIpAddressName "publicIp$i" `
        -Credential $cred
}

##################EAST###################

$rg = @{
    Name = 'Casestudy-EastUS'
    Location = 'eastus'
}
New-AzResourceGroup @rg

$vnet = @{
    Name = 'EUSVnet'
    ResourceGroupName = 'Casestudy-EastUS'
    Location = 'eastus'
    AddressPrefix = '172.101.0.0/16'    
}
$virtualNetwork = New-AzVirtualNetwork @vnet

$subnet = @{
    Name = 'eastussubnet'
    VirtualNetwork = $virtualNetwork
    AddressPrefix = '172.101.1.0/24'
}
$subnetConfig = Add-AzVirtualNetworkSubnetConfig @subnet

####NSG#######

$nsg = New-AzNetworkSecurityGroup -ResourceGroupName Casestudy-EastUS -Location eastus -Name `
    "EastUSNSG"

######VM Creation############ Quota reached error

$vm1 = @{
    ResourceGroupName = 'Casestudy-EastUS'
    Location = 'eastus'
    Name = 'Server11VM'
    VirtualNetworkName = 'EUSVnet'
    SubnetName = 'eastussubnet'
}
New-AzVM @vm1


####VNET PEERING##############
$vnet1 = Get-AzVirtualNetwork -Name CSVnet -ResourceGroupName Casestudy-SEA
$vnet2 = Get-AzVirtualNetwork -Name EUSVnet -ResourceGroupName Casestudy-EastUS

Add-AzVirtualNetworkPeering -Name SEAtoEUS -VirtualNetwork $vnet1 -RemoteVirtualNetworkId $vnet2.Id
Add-AzVirtualNetworkPeering -Name EUStoSEA -VirtualNetwork $vnet2 -RemoteVirtualNetworkId $vnet1.Id

###Backup of VM###### ---- I am doing it for other VM as i was not able to create due to quota limit reached error

New-AzRecoveryServicesVault `
    -ResourceGroupName "Vms" `
    -Name "RecoveryServicesVaultEast" `
-Location "EastUS"
##context##
Get-AzRecoveryServicesVault `
 -Name "RecoveryServicesVaultEast" | Set-AzRecoveryServicesVaultContext

##Setting the policy:

 $policy = Get-AzRecoveryServicesBackupProtectionPolicy     -Name "DefaultPolicy"

##Enabling the Backu##

Enable-AzRecoveryServicesBackupProtection `
    -ResourceGroupName "Vms" `
    -Name "Windows-VM" `
    -Policy $policy


Get-AzRecoveryservicesBackupJob

