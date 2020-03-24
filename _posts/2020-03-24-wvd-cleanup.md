---
title: Windows Virtual Desktop - Cleanup
categories:
    - Azure
tags:
    - WindowsVirtualDesktop
excerpt: How to remove all artifacts from WVD
toc: true
toc_label: Remove WVD
---

---

1. [How to](https://www.mczerniawski.pl/azure/wvd-how-to/)
1. [Monitor](https://www.mczerniawski.pl/azure/wvd-monitor/)
1. [Cleanup](https://www.mczerniawski.pl/azure/wvd-cleanup/)

---

## Cleanup time

Recently we'de [deployed](https://www.mczerniawski.pl/azure/wvd-how-to/) and set up [basic monitoring](https://www.mczerniawski.pl/azure/wvd-monitor/) with Azure App Insight and Log Analytics.

As production environment is up and running - time to destroy the PoC environment.

It's always easy to say - `clean it up`. I prefer to PLAN and write down (in a form of a checklist) all steps **BEFORE** I dive into. That helps in case you're interrupted, but also serves as a `basic documentation`.  
Together with `deployment steps` it's a good start :grin:.

So let's get it started. What needs to be done?

1. All Sessions that are currently open will be shutdown.
1. RDS application group removed.
1. All hosts removed from Host Pool.
1. Host Pool removed.
1. RDS Tenant removed.
1. Azure AD application account removed.
1. Resource group with all resources - removed as well.
1. AD computer objects with DNS and DHCP entries - removed.
1. Entries in monitoring system (in our case Zabbix) also cleaned up.

I'm not removeing assigned licenses, as we use Windows E5 not only for WVD.

## Automate

I'll be destroying a PoC environment, but in some time - probably production WVD will go down. That is why I prefer to automate it a bit.

### WVD tenant

Let's first take down the WVD tenant. I'm using `Out-GridView -PassThru` as interactive interface for scripts. I want to remove PoC environment, not the production one :grin:

```powershell
Install-module microsoft.RDInfra.RDPowerShell
Import-Module Microsoft.RDInfra.RDPowerShell
Add-RDSAccount -DeploymentUrl https://rdbroker.wvd.microsoft.com

#Gather information about our current RDS tenant
$currentRDSTenant = Get-RDSTenant | Out-GridView -PassThru
$CurrentRDSHostPool = Get-RDSHostPool -TenantName $currentRDSTenant.TenantName
$CurrentRDSSessionHost = Get-RdsSessionHost -TenantName $currentRDSTenant.TenantName -HostPoolName $CurrentRDSHostPool.HostPoolName
$CurrentRDSAppGroup = Get-RdsAppGroup -TenantName $currentRDSTenant.TenantName -HostPoolName $CurrentRDSHostPool.HostPoolName

#Remove any RD sessions users may still have
$CurrentRDSSessions = Get-RdsUserSession -TenantName $currentRDSTenant.TenantName -HostPoolName $CurrentRDSHostPool.HostPoolName
$CurrentRDSSessions | ForEach-Object {
    $RdsUserSessionLogoffProperties = @{
        TenantName = $currentRDSTenant.TenantName
        HostPoolName = $CurrentRDSHostPool.HostPoolName
        SessionHostName = $PSItem.SessionHostName
        SessionId = $PSItem.SessionId
        NoUserPrompt = $true
    }
    Invoke-RdsUserSessionLogoff  @RdsUserSessionLogoffProperties
}

#Remove RDS Application Group, Session Hosts, Pool and finally tenant
$CurrentRDSAppGroup | Remove-RdsAppGroup
$CurrentRDSSessionHost | Remove-RdsSessionHost -Force
$CurrentRDSHostPool | Remove-RdsHostPool
$currentRDSTenant | Remove-RDSTenant
```

### Azure AD Application

Now, delete the Azure AD Application we used to configure WVD:

```powershell
Import-Module AzureAD
$aadContext = Connect-AzureAD
$DisplayName = "Windows Virtual Desktop Svc Principal"
$currentAzureADApplication = Get-AzureADApplication -All $true | Where-Object { $_.DisplayName -eq $DisplayName }
$currentAzureADApplication | Remove-AzureADApplication
```

### Azure Resource Group

After this, Resource Group will not exist anymore :smile:. Bear in mind this may take a while to complete:

```powershell
#region Remove Resource Group
Connect-AzureRmAccount
$Subscription = Get-AzureRmSubscription | Out-GridView -PassThru | Select-AzureRmSubscription
$resourceGroup = Get-AzureRmResourceGroup | Out-GridView -PassThru
Remove-AzureRmResourceGroup -Name $ResourceGroup.ResourceGroupName
#endregion
```

### Active Directory

Now, some finishing touches in Active Directory. Thare are still some computer objects, DNS and DHCP entries to take care of. Again, I'll be using Out-GridView, so that Operator can confirm what to delete.  
Your environment may vary and you may want to include other steps in here or remove some of the existing.

> This is a `quick-and-dirty` solution to simplify the removal in OUR environment.

We create '{ComputerName}-Admins' and a few other AD groups for each AD Computer to better manage ACLs. I'll hunt every group that looks like 'Computer-*', Out-GridView it for the Operator to select which ones should be removed.

In the end - we'll connect to Zabbix (if selected so in the variables section) and hunt for any entries.

```powershell
#region Variables section
$WVDVMsNamePrefix = 'WVD1'
$FilterString = 'Name -like "{0}*"' -f $WVDVMsNamePrefix
$VMsToRemove = Get-ADComputer -filter $FilterString | Out-GridView -PassThru

$DNSServer = Get-ADDomainController | Select-Object -ExpandProperty HostName # Or provide other DNS
$DHCPServers = Get-DhcpServerInDC | select-Object -ExpandProperty DNSName | Out-GridView -PassThru # Or provide other DHCP

$IncludeZabbix = $true # $true, $false

$AdditionalDNSRecords = $null # @('somename') #if there are any additional dns records to hunt and delete

$Domain = 'contoso.com'
$ZabbixURI = 'https://zabbix.{0}/zabbix/api_jsonrpc.php' -f $Domain

$serverprops = @{
    ComputerName = $DNSServer
    ZoneName     = $Domain
}
#endregion

#region RunTime section
foreach ($ComputerName in $VMsToRemove.Name) {

    #region DNS Entries
    $DNSrecords = Get-DnsServerResourceRecord  @serverprops | Where-Object { $_.HostName -match $ComputerName } | Out-GridView -PassThru
    if ($DNSrecords) {
        $DNSrecords | ForEach-Object {
            Write-Host "Removing DNS entry {$($PSItem.HostName)} - IP {$($PSItem.RecordData.IPv4Address.ToString())} - record type {$($PSItem.RecordType)}"
            Remove-DnsServerResourceRecord @serverprops -Name $PSItem.HostName -RRType $PSItem.RecordType -confirm:$false
        }
    }
    #endregion

    #region DNS delete additional records matching criteria
    if ($null -ne $AdditionalDNSRecords) {
        $AdditionalDNS = foreach ($dnsRecord in $AdditionalDNSRecords) {
            Get-DnsServerResourceRecord  @serverprops | where { $PSItem.hostName -match $AdditionalDNSRecord } | Out-GridView -PassThru
        }
        if ($AdditionalDNS) {
            $AdditionalDNS | ForEach-Object {
                Write-Host "Removing DNS entry {$($PSItem.HostName)} - IP {$($PSItem.RecordData)} - record type {$($PSItem.RecordType)}"
                Remove-DnsServerResourceRecord @serverprops -Name $PSItem.HostName -RRType $PSItem.RecordType -confirm:$false
            }
        }
    }
    #endregion

    #region DHCP entries
    foreach ($dhcpComputer in $DHCPServers) {
        $dhcpScopes = Get-DhcpServerv4Scope -computername $dhcpComputer
        $DHCPLeases = $dhcpScopes | ForEach-Object { Get-DhcpServerv4Lease -ComputerName $dhcpComputer -ScopeId $PSItem.ScopeID | Where-Object { $PSItem.HostName -match $ComputerName } } | Out-GridView -PassThru

        if ($DHCPLeases) {
            foreach ($dhcplease in $DHCPLeases) {
                if ($dhcplease.AddressState -match 'Reservation') {
                    Write-Host "Removing DHCP Reservation {$($dhcplease.HostName)} - IP {$($dhcplease.IPAddress)} - address state {$($dhcplease.AddressState)}"
                    Remove-DhcpServerv4Reservation -ComputerName $dhcpComputer -ScopeId $dhcplease.scopeid -ClientId $dhcplease.ClientID -Confirm:$false -PassThru
                }
                else {
                    Write-Host "Removing DHCP Lease {$($dhcplease.HostName)} - IP {$($dhcplease.IPAddress)} - address state {$($dhcplease.AddressState)}"
                    Remove-DhcpServerv4Lease -ComputerName $dhcpComputer -ScopeId $dhcplease.scopeid -ClientId $dhcplease.ClientID -Confirm:$false -PassThru
                }
            }
        }
    }
    #endregion

    #region Delete Computer Object
    $ComputerObject = Get-ADComputer -filter { Name -like $ComputerName } | Out-GridView -PassThru
    if ($ComputerObject) {
        Write-Host "Found AD Object for Computer {$computerName} - {$($ComputerObject.Name)} with state {$($ComputerObject.Enabled)}"
        $ComputerObject | foreach-object {
            Write-Host "    Removing Computer Object {$($PSItem.Name)}"
            $PSItem | Get-ADObject | ForEach-Object {  
                Remove-ADObject -Recursive -Identity $PSItem
            }
        }
    }
    else {
        Write-Host "No Computer Objects found for Computer {$ComputerName}"
    }
    #endregion

    #region Remove Groups
    Write-Host "Enumerating groups for computer [$ComputerName]"
    $FilterString = 'Name -like "{0}-*"' -f $ComputerName
    $ComputerGroups = Get-ADGroup -filter $FilterString | Out-GridView -Title 'Found Group Records' -PassThru
    if ($ComputerGroups) {
        $ComputerGroups | ForEach-Object {
            Write-Host "Removing ADGroup {$($PSItem.Name)} with DN  {$($PSItem.DistinguishedName)}"
            $PSItem | Remove-ADGroup -Confirm:$false
        }
    }
    else {
        Write-Host "No Groups found for Computer {$ComputerName}"
    }
    #endregion

    #region remove from Zabbix
    if ($IncludeZabbix) {
        Write-Host "Processing Zabbix for computer {$ComputerName}"
        if (-not ($zabbixSession)) {
            Import-Module PSZabbix
            $zabbixSession = New-ZbxApiSession $ZabbixURI (Get-Credential $env:Username -Message "Provide Zabbix credentials")
        }
        #region cleanup zabbix
        $zbxHost = Get-ZbxHost -Session $zabbixSession | Where-Object { $PSItem.Name -match "$computername" } | Out-GridView -PassThru
        if ($zbxHost) {
            Write-Host "    Removing host {$ComputerName} from zabbix - hostID {$($zbxHost.HostId)}"
            Remove-ZbxHost -Session $zabbixSession -HostId $zbxHost.hostid
        }
        else {
            Write-Host "    No Zabbix host found for Computer {$ComputerName}"
        }
        #endregion
    }
    #endregion
#endregion
}

```

## Summary

After this, we should have all the objects deleted. If I forgot about something - please let me know!
