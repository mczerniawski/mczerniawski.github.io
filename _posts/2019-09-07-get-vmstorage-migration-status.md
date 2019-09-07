---
title: Hyper-V storage migration status
categories:
    - HyperV
tags:
    - HyperV
    - PowerShell
    - StorageMigration
excerpt: How to easily view current VM migration status using PowerShell
---

# Migrate all the things

One of the common tasks with Hyper-V is to migrate a VM from one node to another. You can do it the hard way (export/copy/import) or the easy way - with [Live Migrate](https://docs.microsoft.com/en-us/windows-server/virtualization/hyper-v/manage/live-migration-overview). Sometimes you may want to migrate only storage (one LUN/CSV to another). As always - there are multiple ways to achieve that:

1. Using GUI
1. Using PowerShell

## Migrate using GUI

This is a fairly simple yet boring process.

Select a VM you want to migrate, right click and then `move`:  
![GUI1]({{ site.url }}{{ site.baseurl }}/assets/images/posts/get-vmstorage-migration/picture1.png)  
Move the complete VM or just the storage:  
![GUI2]({{ site.url }}{{ site.baseurl }}/assets/images/posts/get-vmstorage-migration/picture2.png)
Select the host you want to migrate the VM to:  
![GUI3]({{ site.url }}{{ site.baseurl }}/assets/images/posts/get-vmstorage-migration/picture3.png)  
Select what you want to migrate. Here I'm migrating everything (compute and storage) together:  
![GUI4]({{ site.url }}{{ site.baseurl }}/assets/images/posts/get-vmstorage-migration/picture4.png)  
Provide the path where VM files should be stored:  
![GUI5]({{ site.url }}{{ site.baseurl }}/assets/images/posts/get-vmstorage-migration/picture5.png)  
Verify all settings and click `finish`:  
![GUI6]({{ site.url }}{{ site.baseurl }}/assets/images/posts/get-vmstorage-migration/picture6.png)  

## Migrate using PowerShell

The same can be achieved using one line:

```powershell
Move-VM -Name 'Router-VyOS' -DestinationHost 'OBJPLTHV1' -IncludeStorage -DestinationStoragePath 'D:\VMs\Router-VyOS'
```

# Status of migration

Once you initiate the migration it would be good know its current status, right?

1. Using GUI
1. Using PowerShell

## Check status using GUI

Using Hyper-V Manager console you can track this per Hyper-V host:  
![GUI7]({{ site.url }}{{ site.baseurl }}/assets/images/posts/get-vmstorage-migration/picture7.png)  

## Check status using PowerShell

This becomes less convenient if you have multiple hosts or cluster and more migrations going on. Or you want to automate it e.g. some response once a migration is done.

To get the required information, we'll use `Get-CimInstance`:

```powershell
Get-CimInstance -Namespace root\virtualization\v2 -ClassName Msvm_MigrationJob
```

I've created a small function that works with remote hosts and credential parameter. It will generate a nice output with following information:

- `VMName`
- `State` of VM
- `Status` of migration
- `Description` - a more verbose step of current migration
- `JobStatus` - is it running, completed or cancelled
- `PercentComplete`
- `Owner` - who initiated the migration
- `SourceHost` and `DestinationHost`
- `DestinationPath` - where VM files will be stored
- `VMID` - ID of a vm - may be handy for automation
- `StartTime` and `ElapsedTime`
- `ErrorDescription` and `ErrorSummaryDescription` in case of error

```powershell
function Get-VMStorageMigrationStatus {
    <#
    .SYNOPSIS
    Get current status of VM migration
    
    .DESCRIPTION
    Uses Invoke-Command to connect to Hyper-V host and check any VMs running storage migration and status of it
    
    .PARAMETER ComputerName
    Computername of Hyper-V host
    
    .PARAMETER Credential
    Optional credential parameter
    
    .EXAMPLE
    Get-VMStorageMigrationStatus -ComputerName Host1 -Verbose
    VERBOSE: Processing with default credentials of user {mczerniawski_admin}

    VMName                  : VM1
    State                   : Off
    Status                  : Migrating virtual machine
    Description             : Moving Virtual Machine and Storage
    JobStatus               : Job is running
    JobName                 : Moving Virtual Machine and Storage
    PercentComplete         : 22
    Owner                   : CONTOSO\mczerniawski_admin
    SourceHost              : Host1
    DestinationHost         : Host2
    DestinationPath         : D:\VMs\Router-VyOS
    VMID                    : 42B54F68-5E1D-48D9-96CE-3370175F22C4
    StartTime               : 07/09/2019 09:31:35
    ElapsedTime             : 00:04:00.7814980
    ErrorDescription        : 
    ErrorSummaryDescription : 
    PSComputerName          : Host2
    RunspaceId              : 775632c4-be8e-40d0-94d3-38615d457840

    VMName                  : VM2
    State                   : Off
    Status                  : Migrating virtual machine
    Description             : Moving Virtual Machine and Storage
    JobStatus               : Job is running
    JobName                 : Moving Virtual Machine and Storage
    PercentComplete         : 32
    Owner                   : CONTOSO\mczerniawski_admin
    SourceHost              : Host1
    DestinationHost         : Host2
    DestinationPath         : D:\VMs\Router-VyOS
    VMID                    : A2E38807-43BF-4C7D-86C3-4DFF0A8FE5F1
    StartTime               : 07/09/2019 09:32:35
    ElapsedTime             : 00:02:00.7814980
    ErrorDescription        : 
    ErrorSummaryDescription : 
    PSComputerName          : Host2
    RunspaceId              : 775632c4-be8e-40d0-94d3-38615d457840
    
    #>
    
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, HelpMessage = 'Provide ComputerName Name',
            ValueFromPipeline, ValueFromPipelineByPropertyName)]
        [ValidateNotNullOrEmpty()]
        [System.String[]]
        $ComputerName,
    
        [Parameter(Mandatory = $false, HelpMessage = 'Provide Credentials for ComputerName',
            ValueFromPipeline, ValueFromPipelineByPropertyName)]
        [System.Management.Automation.PSCredential]
        $Credential
       

    )
    process {
        foreach ($Computer in $ComputerName) {
            #region PSSession parameters
            $connectionParams = @{
                ComputerName = $Computer
            }
            if ($PSBoundParameters.ContainsKey('Credential')) {
                $connectionParams.Credential = $Credential
                Write-Verbose -Message "Processing with provided credentials {$($Credential.UserName)}"
            }
            else {
                Write-Verbose -Message "Processing with default credentials of user {$($env:USERNAME)}"
            }
            #endregion
            Write-Verbose -Message "Processing computer {$Computer}"
            Invoke-Command @connectionParams -ScriptBlock {
                $StorageMigrations = Get-CimInstance -Namespace root\virtualization\v2 -ClassName Msvm_MigrationJob
                if ($StorageMigrations) {
                    foreach ($stMigration in $StorageMigrations) {
                        $VMDetails = Get-VM | Where-Object {$PSItem.ID -eq $stMigration.VirtualSystemName}
                        $xml = [xml]($stMigration.NewSystemSettingData)
                        [pscustomobject]@{
                            VMName          = $VMDetails.Name
                            State           = $VMDetails.State
                            Status          = $VMDetails.Status
                            Description     = $stMigration.Description
                            JobStatus       = $stMigration.JobStatus
                            JobName         = $stMigration.Name
                            PercentComplete = $stMigration.PercentComplete
                            Owner           = $stMigration.Owner
                            SourceHost      = $env:COMPUTERNAME
                            DestinationHost = $stMigration.DestinationHost
                            DestinationPath = $xml.Instance.property | Where-Object {$psitem.Name -eq 'ConfigurationDataRoot'} | Select-Object -ExpandProperty Value
                            VMID            = $stMigration.VirtualSystemName
                            StartTime       = $stMigration.StartTime
                            ElapsedTime     = $stMigration.ElapsedTime
                            ErrorDescription = $stMigration.ErrorDescription
                            ErrorSummaryDescription = $stMigration.ErrorSummaryDescription
                        }
                    }
                }
            }
        }
    }
}
```

If I need to query all nodes of a failover cluster, I'm using this snippet:

```powershell
$HyperVHost = 'hvcluster0' #clustername 'hvcluster0' or hostname 'host1'
#$Credential = Get-Credential
$Cluster = $true
$connProperties = @{
    ComputerName = $HyperVHost
}
if ($Credential) {
    $connProperties.Credential = $Credential
}
if ($Cluster) {
    $Nodes = Invoke-Command @connProperties -ScriptBlock {
        Get-ClusterNode | where-object {$_.State -eq 'Up'} | select-object -ExpandProperty Name
    }
    $connProperties.ComputerName = $Nodes
}


Get-VMStorageMigrationStatus @connProperties | Format-Table -AutoSize
```

And the output will be similar to this:  
![get-migration]({{ site.url }}{{ site.baseurl }}/assets/images/posts/get-vmstorage-migration/picture8.png)
