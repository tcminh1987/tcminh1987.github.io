#Requires -Version 5 -Modules VMware.PowerCLI

# Import server info from given CSV
# Header:
#Hostname,DefaultGateway,IPAddress,SubnetMask,Cluster,Network,Folder,CPU,MemoryGB
$Servers = Import-CSV -Path "${env:userprofile}\Documents\linux_deploy.csv"

# Required options for Linux NIC customization spec
$NICSpec = @{
    IPMode = 'UseStaticIP'
    Position = 1
}

# Iterate through details listed in CSV
foreach ($Server in $Servers) {
    # pull out hostname portion of FQDN
    $VMName = $Server.Hostname.Split('.')[0]
    Write-Output "Processing $VMName"

    # retrieve cluster object
    $Cluster = Get-Cluster -Name $Server.Cluster

    # retrieve public distributed vSwitch
    $VDSwitchName = $Server.Cluster + "-DSwitch-Public"
    $VDSwitch = Get-VDSwitch -Name $VDSwitchName

    # retrieve corresponding distributed portgroup according to CSV Network
    $PortGroup = Get-VDPortGroup -VDSwitch $VDSwitch -Name $Server.Network

    # I perform some basic heuristics based on the location of the VM
    if ($Server.Cluster -match "CLUSTER01") {
        # we assign different internal resolvers based on site, adjust as needed
        $DNSServer = @('192.168.1.100', '192.168.2.100')

        # choose template appropriate for datacenter
        $Template = Get-Template -Name "rhel-template-cluster01"

        # select first folder found with specified folder namer
        $Folder = Get-Folder -Server vcenter01.example.com -Name $Server.Folder | Select-Object -First 1

        # choose datastore that has the most remaining free space
        if ($VMName -match "tst") {
            $Datastore = Get-Datastore "cluster01_tst*" | Sort-Object FreeSpaceGB | Select-Object -Last 1
        } elseif ($VMName -match "prd") {
            $Datastore = Get-Datastore "cluster01_prd*" | Sort-Object FreeSpaceGB | Select-Object -Last 1
        }
    } elseif ($Server.Cluster -match "CLUSTER02") {
        # assign private resolvers for datacenter
        $DNSServer = @('192.168.2.100', '192.168.1.100')

        # choose template appropriate for datacenter
        $Template = Get-Template -Name "rhel7-template-cluster02"

        # select first folder found with specified folder namer
        $Folder = Get-Folder -Server vcenter02.example.com -Name $Server.Folder | Select-Object -First 1

        # choose datastore that has the most remaining free space
        if ($VMName -match "tst") {
            $Datastore = Get-Datastore "cluster02_tst*" | Sort-Object FreeSpaceGB | Select-Object -Last 1
        } elseif ($VMName -match "prd") {
            $Datastore = Get-Datastore "cluster02_prd*" | Sort-Object FreeSpaceGB | Select-Object -Last 1
        }
    }

    # some basic validation that these values were at least assigned
    if ( -not $DNSServer ) {
        Write-Warning "The DNSServer variable has not been assigned"
        exit
    }
    if ( -not $Template ) {
        Write-Warning "The Template variable has not been assigned"
        exit
    }
    if ( -not $Datastore ) {
        Write-Warning "The Datastore variable has not been assigned"
        exit
    }
    if ( -not $Folder ) {
        Write-Warning "The Folder variable has not been assigned"
        exit
    }
    if ( $PortGroup -eq $null ) {
        Write-Warning "The PortGroup variable has not been assigned"
        exit
    }

    # create temporary customization spec based off of VM's name
    $SpecName = $VMName + "-vmspec"
    $VMSpec = New-OSCustomizationSpec -Name $SpecName `
        -NamingScheme 'fixed' `
        -OSType 'Linux' `
        -Domain 'ad.example.com' `
        -DNSSuffix 'ad.example.com' `
        -DNSServer $DNSServer `
        -Type 'NonPersistent' `
        -NamingPrefix $VMName

    # assign NIC customization details from CSV info
    $NICSpec.IPAddress = $Server.IPAddress
    $NICSpec.DefaultGateway = $Server.DefaultGateway
    $NICSpec.SubnetMask = $Server.SubnetMask

    # Clear out any extant NIC customizations on the OS customization object
    Get-OSCustomizationNicMapping -OSCustomizationSpec $VMSpec | Remove-OSCustomizationNicMapping -Confirm:$false
    # assign the newly created NIC customization to our OS customization
    New-OSCustomizationNicMapping -OSCustomizationSpec $VMSpec @NICSpec

    # clone the VM with the information gathered earlier
    $VM = New-VM -Name $VMName -Template $Template -ResourcePool $Cluster -OSCustomizationSpec $VMSpec -Datastore $Datastore -Location $Folder

    # update hardware specs
    Set-VM -VM $VM -NumCPU $Server.CPU -MemoryGB $Server.MemoryGB -Confirm:$false

    # retrieve the VM's NICs and set the appropriate details
    $NIC = Get-NetworkAdapter -VM $VM
    Set-NetworkAdapter -NetworkAdapter $NIC -StartConnected:$true -Confirm:$false
    Set-NetworkAdapter -NetworkAdapter $NIC -PortGroup $PortGroup -Confirm:$false

    # The following was adapted from https://www.altaro.com/vmware/powercli-script-deploy-vms-and-configure-the-guest-os/
    # Start the VM and wait for the customization to complete
    Start-VM -VM $VM
    while($True) {
        $VMEvents = Get-VIEvent -Entity $VM
        $VMStartedEvent = $VMEvents | Where { $_.GetType().Name -eq "CustomizationStartedEvent" }
        if ($VMStartedEvent) {
            break
        } else {
            Start-Sleep -Seconds 2
        }
    }
    Write-Output "Customization of VM $VMName has started. Checking for Completed Status..."
    while($True) {
        $VMEvents = Get-VIEvent -Entity $VM
        $SucceededEvent = $VMEvents | Where { $_.GetType().Name -eq "CustomizationSucceeded" }
        $FailureEvent = $VMEvents | Where { $_.GetType().Name -eq "CustomizationFailed" }
        if ($FailureEvent) {
            Write-Warning "Customization of VM $VMName failed"
            return $False
        }
        if ($SucceededEvent) {
            break
        }
        Start-Sleep -Seconds 2
    }
    Write-Output "Customization of VM $VMName Completed Successfully!"

    # Remove the temporary customization
    Remove-OSCustomizationSpec -OSCustomizationSpec $VMSpec -Confirm:$false
}
