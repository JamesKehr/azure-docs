---
title: Create and manage Windows VMs in Azure that use multiple NICs 
description: Learn how to create and manage a Windows VM that has multiple NICs attached to it by using Azure PowerShell or Resource Manager templates.
author: cynthn
ms.service: virtual-machines
ms.collection: windows
ms.topic: how-to
ms.workload: infrastructure
ms.date: 09/26/2017
ms.author: cynthn 
ms.custom: devx-track-azurepowershell

---
# Create and manage a Windows virtual machine that has multiple NICs

**Applies to:** :heavy_check_mark: Windows VMs :heavy_check_mark: Flexible scale sets 

Virtual machines (VMs) in Azure can have multiple virtual network interface cards (NICs) attached to them. A common scenario is to have different subnets for front-end and back-end connectivity. You can associate multiple NICs on a VM to multiple subnets, but those subnets must all reside in the same virtual network (vNet). This article details how to create a VM that has multiple NICs attached to it. You also learn how to add or remove NICs from an existing VM. Different [VM sizes](../sizes.md) support a varying number of NICs, so size your VM accordingly.

> [!NOTE]
> If multiple subnets are not required for a scenario, it may be more straightforward to utilize multiple IP configurations on a single NIC.  Instructions for this setup can be found [here](../../virtual-network/ip-services/virtual-network-multiple-ip-addresses-portal.md).

## Prerequisites

In the following examples, replace example parameter names with your own values. Example parameter names include *myResourceGroup*, *myVnet*, and *myVM*.

 

## Create a VM with multiple NICs
First, create a resource group. The following example creates a resource group named *myResourceGroup* in the *EastUs* location:

```powershell
New-AzResourceGroup -Name "myResourceGroup" -Location "EastUS"
```

### Create virtual network and subnets
A common scenario is for a virtual network to have two or more subnets. One subnet may be for front-end traffic, the other for back-end traffic. To connect to both subnets, you then use multiple NICs on your VM.

1. Define two virtual network subnets with [New-AzVirtualNetworkSubnetConfig](/powershell/module/az.network/new-azvirtualnetworksubnetconfig). The following example defines the subnets for *mySubnetFrontEnd* and *mySubnetBackEnd*:

    ```powershell
    $mySubnetFrontEnd = New-AzVirtualNetworkSubnetConfig -Name "mySubnetFrontEnd" `
        -AddressPrefix "192.168.1.0/24"
    $mySubnetBackEnd = New-AzVirtualNetworkSubnetConfig -Name "mySubnetBackEnd" `
        -AddressPrefix "192.168.2.0/24"
    ```

2. Create your virtual network and subnets with [New-AzVirtualNetwork](/powershell/module/az.network/new-azvirtualnetwork). The following example creates a virtual network named *myVnet*:

    ```powershell
    $myVnet = New-AzVirtualNetwork -ResourceGroupName "myResourceGroup" `
        -Location "EastUs" `
        -Name "myVnet" `
        -AddressPrefix "192.168.0.0/16" `
        -Subnet $mySubnetFrontEnd,$mySubnetBackEnd
    ```


### Create multiple NICs
Create two NICs with [New-AzNetworkInterface](/powershell/module/az.network/new-aznetworkinterface). Attach one NIC to the front-end subnet and one NIC to the back-end subnet. The following example creates NICs named *myNic1* and *myNic2*:

```powershell
$frontEnd = $myVnet.Subnets|?{$_.Name -eq 'mySubnetFrontEnd'}
$myNic1 = New-AzNetworkInterface -ResourceGroupName "myResourceGroup" `
    -Name "myNic1" `
    -Location "EastUs" `
    -SubnetId $frontEnd.Id

$backEnd = $myVnet.Subnets|?{$_.Name -eq 'mySubnetBackEnd'}
$myNic2 = New-AzNetworkInterface -ResourceGroupName "myResourceGroup" `
    -Name "myNic2" `
    -Location "EastUs" `
    -SubnetId $backEnd.Id
```

Typically you also create a [network security group](../../virtual-network/network-security-groups-overview.md) to filter network traffic to the VM and a [load balancer](../../load-balancer/load-balancer-overview.md) to distribute traffic across multiple VMs.

### Create the virtual machine
Now start to build your VM configuration. Each VM size has a limit for the total number of NICs that you can add to a VM. For more information, see [Windows VM sizes](../sizes.md).

1. Set your VM credentials to the `$cred` variable as follows:

    ```powershell
    $cred = Get-Credential
    ```

2. Define your VM with [New-AzVMConfig](/powershell/module/az.compute/new-azvmconfig). The following example defines a VM named *myVM* and uses a VM size that supports more than two NICs (*Standard_DS3_v2*):

    ```powershell
    $vmConfig = New-AzVMConfig -VMName "myVM" -VMSize "Standard_DS3_v2"
    ```

3. Create the rest of your VM configuration with [Set-AzVMOperatingSystem](/powershell/module/az.compute/set-azvmoperatingsystem) and [Set-AzVMSourceImage](/powershell/module/az.compute/set-azvmsourceimage). The following example creates a Windows Server 2016 VM:

    ```powershell
    $vmConfig = Set-AzVMOperatingSystem -VM $vmConfig `
        -Windows `
        -ComputerName "myVM" `
        -Credential $cred `
        -ProvisionVMAgent `
        -EnableAutoUpdate
    $vmConfig = Set-AzVMSourceImage -VM $vmConfig `
        -PublisherName "MicrosoftWindowsServer" `
        -Offer "WindowsServer" `
        -Skus "2016-Datacenter" `
        -Version "latest"
   ```

4. Attach the two NICs that you previously created with [Add-AzVMNetworkInterface](/powershell/module/az.compute/add-azvmnetworkinterface):

    ```powershell
    $vmConfig = Add-AzVMNetworkInterface -VM $vmConfig -Id $myNic1.Id -Primary
    $vmConfig = Add-AzVMNetworkInterface -VM $vmConfig -Id $myNic2.Id
    ```

5. Create your VM with [New-AzVM](/powershell/module/az.compute/new-azvm):

    ```powershell
    New-AzVM -VM $vmConfig -ResourceGroupName "myResourceGroup" -Location "EastUs"
    ```

6. Add routes for secondary NICs to the OS by completing the steps in [Configure the operating system for multiple NICs](#configure-guest-os-for-multiple-nics).

## Add a NIC to an existing VM
To add a virtual NIC to an existing VM, you deallocate the VM, add the virtual NIC, then start the VM. Different [VM sizes](../sizes.md) support a varying number of NICs, so size your VM accordingly. If needed, you can [resize a VM](../resize-vm.md).

1. Deallocate the VM with [Stop-AzVM](/powershell/module/az.compute/stop-azvm). The following example deallocates the VM named *myVM* in *myResourceGroup*:

    ```powershell
    Stop-AzVM -Name "myVM" -ResourceGroupName "myResourceGroup"
    ```

2. Get the existing configuration of the VM with [Get-AzVm](/powershell/module/az.compute/get-azvm). The following example gets information for the VM named *myVM* in *myResourceGroup*:

    ```powershell
    $vm = Get-AzVm -Name "myVM" -ResourceGroupName "myResourceGroup"
    ```

3. The following example creates a virtual NIC with [New-AzNetworkInterface](/powershell/module/az.network/new-aznetworkinterface) named *myNic3* that is attached to *mySubnetBackEnd*. The virtual NIC is then attached to the VM named *myVM* in *myResourceGroup* with [Add-AzVMNetworkInterface](/powershell/module/az.compute/add-azvmnetworkinterface):

    ```powershell
    # Get info for the back end subnet
    $myVnet = Get-AzVirtualNetwork -Name "myVnet" -ResourceGroupName "myResourceGroup"
    $backEnd = $myVnet.Subnets|?{$_.Name -eq 'mySubnetBackEnd'}

    # Create a virtual NIC
    $myNic3 = New-AzNetworkInterface -ResourceGroupName "myResourceGroup" `
        -Name "myNic3" `
        -Location "EastUs" `
        -SubnetId $backEnd.Id

    # Get the ID of the new virtual NIC and add to VM
    $nicId = (Get-AzNetworkInterface -ResourceGroupName "myResourceGroup" -Name "MyNic3").Id
    Add-AzVMNetworkInterface -VM $vm -Id $nicId | Update-AzVm -ResourceGroupName "myResourceGroup"
    ```

    ### Primary virtual NICs
    One of the NICs on a multi-NIC VM needs to be primary. If one of the existing virtual NICs on the VM is already set as primary, you can skip this step. The following example assumes that two virtual NICs are now present on a VM and you wish to add the first NIC (`[0]`) as the primary:
        
    ```powershell
    # List existing NICs on the VM and find which one is primary
    $vm.NetworkProfile.NetworkInterfaces
    
    # Set NIC 0 to be primary
    $vm.NetworkProfile.NetworkInterfaces[0].Primary = $true
    $vm.NetworkProfile.NetworkInterfaces[1].Primary = $false
    
    # Update the VM state in Azure
    Update-AzVM -VM $vm -ResourceGroupName "myResourceGroup"
    ```

4. Start the VM with [Start-AzVm](/powershell/module/az.compute/start-azvm):

    ```powershell
    Start-AzVM -ResourceGroupName "myResourceGroup" -Name "myVM"
    ```

5. Add routes for secondary NICs to the OS by completing the steps in [Configure the operating system for multiple NICs](#configure-guest-os-for-multiple-nics).

## Remove a NIC from an existing VM
To remove a virtual NIC from an existing VM, you deallocate the VM, remove the virtual NIC, then start the VM.

1. Deallocate the VM with [Stop-AzVM](/powershell/module/az.compute/stop-azvm). The following example deallocates the VM named *myVM* in *myResourceGroup*:

    ```powershell
    Stop-AzVM -Name "myVM" -ResourceGroupName "myResourceGroup"
    ```

2. Get the existing configuration of the VM with [Get-AzVm](/powershell/module/az.compute/get-azvm). The following example gets information for the VM named *myVM* in *myResourceGroup*:

    ```powershell
    $vm = Get-AzVm -Name "myVM" -ResourceGroupName "myResourceGroup"
    ```

3. Get information about the NIC remove with [Get-AzNetworkInterface](/powershell/module/az.network/get-aznetworkinterface). The following example gets information about *myNic3*:

    ```powershell
    # List existing NICs on the VM if you need to determine NIC name
    $vm.NetworkProfile.NetworkInterfaces

    $nicId = (Get-AzNetworkInterface -ResourceGroupName "myResourceGroup" -Name "myNic3").Id   
    ```

4. Remove the NIC with [Remove-AzVMNetworkInterface](/powershell/module/az.compute/remove-azvmnetworkinterface) and then update the VM with [Update-AzVm](/powershell/module/az.compute/update-azvm). The following example removes *myNic3* as obtained by `$nicId` in the preceding step:

    ```powershell
    Remove-AzVMNetworkInterface -VM $vm -NetworkInterfaceIDs $nicId | `
        Update-AzVm -ResourceGroupName "myResourceGroup"
    ```   

5. Start the VM with [Start-AzVm](/powershell/module/az.compute/start-azvm):

    ```powershell
    Start-AzVM -Name "myVM" -ResourceGroupName "myResourceGroup"
    ```   

## Create multiple NICs with templates
Azure Resource Manager templates provide a way to create multiple instances of a resource during deployment, such as creating multiple NICs. Resource Manager templates use declarative JSON files to define your environment. For more information, see [overview of Azure Resource Manager](../../azure-resource-manager/management/overview.md). You can use *copy* to specify the number of instances to create:

```json
"copy": {
    "name": "multiplenics",
    "count": "[parameters('count')]"
}
```

For more information, see [creating multiple instances by using *copy*](../../azure-resource-manager/templates/copy-resources.md). 

You can also use `copyIndex()` to append a number to a resource name. You can then create *myNic1*, *MyNic2* and so on. The following code shows an example of appending the index value:

```json
"name": "[concat('myNic', copyIndex())]", 
```

You can read a complete example of [creating multiple NICs by using Resource Manager templates](../../virtual-network/template-samples.md).

Add routes for secondary NICs to the OS by completing the steps in [Configure the operating system for multiple NICs](#configure-guest-os-for-multiple-nics).

## Configure guest OS for multiple NICs

Azure only assigns a default gateway to the first (primary) network interface attached to a virtual machine. Azure does not assign a default gateway to additional (secondary) network interfaces. As a result you cannot, by default, communicate with resources outside the local subnet of secondary network interfaces. This section covers the basic steps needed to enable cross subnet access for Windows. 

Please refer to vendor documentation regarding route creation for any non-Windows virtual machine.

1. From a PowerShell console, run the `Get-NetAdapter` command, which returns the attached network interfaces:

    ```PowerShell
    Get-NetAdapter
    ```
    ```PowerShell
    Name                      InterfaceDescription                    ifIndex Status       MacAddress             LinkSpeed
    ----                      --------------------                    ------- ------       ----------             ---------
    Ethernet 7                Microsoft Hyper-V Network Adapter #3         16 Up           60-45-BD-B3-22-C3        50 Gbps
    Ethernet 13               Mellanox ConnectX-4 Lx Virtual Et...#10      24 Up           60-45-BD-B3-22-C3        50 Gbps
    Ethernet 6                Microsoft Hyper-V Network Adapter #2          3 Up           60-45-BD-83-96-36        50 Gbps
    ```
 
    In this example, **Microsoft Hyper-V Network Adapter #2** (InterfaceIndex **3**, InterfaceAlias (Name) **Ethernet 6**) is the secondary network interface that does not have a default gateway assigned to it.
    
    The **Mellanox ConnectX...** (interface index 24) network interface is the virtual function (VF) adapter used by Azure Accelerated Networking. Do not attempt to add routes on VF adapters. Packet routing for Accelerated Networking is automatically handled by the OS.
    
    In this example, **Ethernet 7** uses Azure Accelerated Networking and **Ethernet 6** does not. These steps can be used regardless of the Accelerated Networking state because IP routing happens at the Hyper-V Network Adapter and not the VF adapter (Mellanox) level. 

2. From a PowerShell console, run the `Get-NetIPConfiguration -InterfaceIndex <idx>` command to see which IP address is assigned to the secondary network interface. The alias `gip` can be used as a alternate to `Get-NetIPConfiguration`.

    ```PowerShell
    Get-NetIPConfiguration -InterfaceIndex 3
    ```
    ```PowerShell
    InterfaceAlias       : Ethernet 6
    InterfaceIndex       : 3
    InterfaceDescription : Microsoft Hyper-V Network Adapter #2
    NetProfile.Name      : Network
    IPv6Address          : fd42:345d:af54:cc61::5
    IPv4Address          : 10.1.1.5
    IPv6DefaultGateway   : fe80::1234:5678:9abc
    IPv4DefaultGateway   :
    DNSServer            : 10.1.0.5
                           10.1.1.5
    ```

    Please note that the **IPv4DefaultGateway** property is blank. A default route (0.0.0.0/0) cannot be created when there is no gateway for an interface.


3. Now use `Get-NetRoute -InterfaceIndex <idx>` to view the existing routes for the network interface.

    ```PowerShell
    Get-NetRoute -InterfaceIndex 3
    ```
    ```PowerShell
    ifIndex DestinationPrefix                              NextHop                                  RouteMetric ifMetric PolicyStore
    ------- -----------------                              -------                                  ----------- -------- -----------
    3       255.255.255.255/32                             0.0.0.0                                          256 10       ActiveStore
    3       224.0.0.0/4                                    0.0.0.0                                          256 10       ActiveStore
    3       10.1.1.255/32                                  0.0.0.0                                          256 10       ActiveStore
    3       10.1.1.5/32                                    0.0.0.0                                          256 10       ActiveStore
    3       10.1.1.0/24                                    0.0.0.0                                          256 10       ActiveStore
    3       ff00::/8                                       ::                                               256 10       ActiveStore
    3       fe80::92e:20e8:fb:de4/128                      ::                                               256 10       ActiveStore
    3       fe80::/64                                      ::                                               256 10       ActiveStore
    3       fd42:345d:af54:cc61::5/128                     ::                                               256 10       ActiveStore
    3       fd42:345d:af54:cc61::/64                       ::                                               256 10       ActiveStore
    3       ::/0                                           fe80::1234:5678:9abc                             256 10       ActiveStore
    ```

    An IPv4 default route uses 0.0.0.0/0 as the DestinationPrefix. An operating system, Windows in this example, cannot route traffic outside the subnet when there is no route, such as the default route, pointing traffic to a gateway.
    
    Use `ping -S` (the -S is case sensitive) to confirm that cross subnet traffic is not working. 
    - Replace the IP's with the appropriate address for your test.
    - The -S parameter sets the Source address used by ping.exe.
    
    ```
    ping -S 10.1.1.5 10.1.0.4

    Pinging 10.1.0.4 from 10.1.1.5 with 32 bytes of data:
    PING: transmit failed. General failure.
    PING: transmit failed. General failure.
    PING: transmit failed. General failure.
    PING: transmit failed. General failure.
    
    Ping statistics for 10.1.0.4:
        Packets: Sent = 4, Received = 0, Lost = 4 (100% loss),
    ```
    
    Windows does not allow inbound ping (ICMP echo) by default. Use this command to allow pings.
    
    ```PowerShell
    Enable-NetFirewallRule -DisplayName 'File and Printer Sharing (Echo Request - ICMPv4-In)'
    ```

4. Create routes using `New-NetRoute`. Use `Get-NetRoute` to confirm changes (see step 3).

    a. Add a Default Route if you wish to send all off-subnet traffic to the gateway.
    
    - If the route exists, but is not working, use `Set-NetRoute` to adjust the route details.
    - `New-NetRoute` automatically stores the new route in both the Active and Persistent route store.
    - The `-NextHop` is the IP address of the gateway.

    ```PowerShell
    New-NetRoute -DestinationPrefix 0.0.0.0/0 -InterfaceIndex 3 -NextHop 10.1.1.1 -RouteMetric 5015
    ```
    ```PowerShell
    ifIndex DestinationPrefix                              NextHop                                  RouteMetric ifMetric PolicyStore
    ------- -----------------                              -------                                  ----------- -------- -----------
    3       0.0.0.0/0                                      10.1.1.1                                        5015 10       ActiveStore
    3       0.0.0.0/0                                      10.1.1.1                                        5015          Persiste...
    ```

    The gateway address for the subnet is the second IP address in the subnet. For example, if the subnet in Azure is defined as 10.1.1.0/24, the gateway IP address would be 10.1.1.1. The IP 10.1.1.0 is called the network ID or network address. If the subnet in Azure is 10.1.0.128/25, the gateway would be 10.1.0.129. The IP 10.1.0.128 would be the network ID.
    
    When in doubt, look at the IPv4DefaultGateway from a system that uses that subnet as the primary network interface. Or you can use a subnet calculator. 
    
    b. Specific routes can be used when you do not wish to route all traffic. 
    
      In this example, only traffic destined to 10.1.0.0/24 is routed to the gateway.

    ```PowerShell
    New-NetRoute -DestinationPrefix 10.1.0.0/24 -InterfaceIndex 3 -NextHop 10.1.1.1 -RouteMetric 5015
    ```
  
5. Confirm that the configuration works by using `ping -S` or the test process of your choice.

    ```
    ping -S 10.1.1.5 10.1.0.4
    
    Pinging 10.1.0.4 from 10.1.1.5 with 32 bytes of data:
    Reply from 10.1.0.4: bytes=32 time=1ms TTL=128
    Reply from 10.1.0.4: bytes=32 time=1ms TTL=128
    Reply from 10.1.0.4: bytes=32 time=1ms TTL=128
    Reply from 10.1.0.4: bytes=32 time<1ms TTL=128
    
    Ping statistics for 10.1.0.4:
        Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
    Approximate round trip times in milli-seconds:
        Minimum = 0ms, Maximum = 1ms, Average = 0ms
    ```


## Next steps
Review [Windows VM sizes](../sizes.md) when you're trying to create a VM that has multiple NICs. Pay attention to the maximum number of NICs that each VM size supports.
