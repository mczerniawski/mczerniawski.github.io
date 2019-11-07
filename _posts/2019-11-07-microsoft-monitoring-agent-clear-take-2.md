---
title: Microsoft Monitoring Agent - Clear cache and HybridWorker Group
categories:
    - Azure
tags:
    - Azure
    - Log Analytics
    - Microsoft Monitoring Agent
    - Powershell
excerpt: When clearing the cache is not enough. Kill it with fire
---

# The Issue

Some time ago I wrote how to [clear cache of Microsoft Monitoring Agent on your on-premises servers](https://www.mczerniawski.pl/azure/microsoft-monitoring-agent-clear-cache). A few days back I experienced a weird issue with Azure Automation.  
Currently I have around 200 OSes (mainly VMs) connected to one of our workspaces. Some of them `partialy` lost connectivity. 

![Troubleshoot]({{ site.url }}{{ site.baseurl }}/assets/images/posts/microsoft-monitoring-agent-clear/picture1.png)

They could report back to Azure Automation about patch status, but were unable to receive any jobs - including patch deployment schedules!  
Those machines were stuck in `Failed to start` status:

![Buhuu]({{ site.url }}{{ site.baseurl }}/assets/images/posts/microsoft-monitoring-agent-take-2/picture1.png)

I've checked network connectivity, run on-premises troubleshooting script, but that didn't get me anywhere. All tests came green.

So I went to `System hybrid worker groups` tab to check if my worker is there:  
1. go to your Azure Automation account
1. Select Hybrid Worker groups under `Process Automation` menu:  
![WorkerGroup]({{ site.url }}{{ site.baseurl }}/assets/images/posts/microsoft-monitoring-agent-take-2/picture2.png)

This got me the list of all of my systems connected, and most important - the last time they were seen.  
![LastTime]({{ site.url }}{{ site.baseurl }}/assets/images/posts/microsoft-monitoring-agent-take-2/picture3.png)

When I compared all the `Not configured` instances and those worker groups that didnt report in last few hours - they all matched.

We have a lead...

## Clean up time

There were also OSes that didn't report becuase... they were already decommissioned. I could delete the worker group from the portal but... that would work for a single instance. I had more of them and I wanted to automate it for my decommission process as well.

If you'd like to remove from the GUI, then click on the server and then click delete:  
![LastTime]({{ site.url }}{{ site.baseurl }}/assets/images/posts/microsoft-monitoring-agent-take-2/picture4.png)

### AzureRM.Automation

There are functions in AzureRM.Automation module that I used to query all Hybrid Worker Groups and delete them. Those are [Get-AzureRmAutomationHybridWorkerGroup](https://docs.microsoft.com/bs-cyrl-ba/powershell/module/azurerm.automation/get-azurermautomationhybridworkergroup?view=azurermps-5.7.0) and [Remove-AzureRmAutomationHybridWorkerGroup](https://docs.microsoft.com/en-us/powershell/module/azurerm.automation/remove-azurermautomationhybridworkergroup?view=azurermps-6.13.0). As you can see at the docs pages `Get-` is available in 5.7.0 version and `Remove-` in 6.13.0.

I wrote a small script that will:

1. Check if required AzureRM module is installed
1. If so, will connect to Azure (interactive logon - mainly because I'm using MFA and that cannot be 'bypassed programaticaly for non-service accounts')
1. Using Out-GridView as interactive menu will:
    1. Select proper subscription
    1. Select resource groupName
    1. Select azure automation account
1. Query for all Hybrid Worker Group accounts
1. Select only those that didn't respond in `PastDays` days or more
1. Remove Hybrid Worker group account
1. Use the same list of servers to connect to them using Invoke-Command and clear cache (code from my [previous post](https://www.mczerniawski.pl/azure/microsoft-monitoring-agent-clear-cache))
    1. Will use errorAction preference of `SilentlyContinue` to avoid issues with already deleted machines.

So here it is:

```powershell
$WorkspaceID = 'xxxxx-da5f-yyyy-bfbf-zzzzzzzzzz'
$WorkspaceKey = 'YourWorkspaceSuperSecretKey'
$PastDays = 3 #How aggressive to clean up

#region Remove Hybrid Worker from Azure
if ((Get-Command Get-AzureRmAutomationHybridWorkerGroup).Version.Major -le 5) {
    Write-Host "Please Update AzureRM.Automation module by running : 'Update-Module azurerm.automation -force' as Administrator"
} else {
    Connect-AzureRmAccount
    Get-AzureRmSubscription | Out-GridView -passthru | Select-AzureRmSubscription
    $ResourceGroupName = Get-AzureRmResourceGroup | Out-GridView -PassThru | Select-Object -ExpandProperty ResourceGroupName
    $AutomationAccountName = Get-AzureRmAutomationAccount | Out-GridView -PassThru | Select-Object -ExpandProperty AutomationAccountName
    $date = (Get-Date).adddays(-$PastDays)
    $workers = Get-AzureRmAutomationHybridWorkerGroup -ResourceGroupName $ResourceGroupName -AutomationAccountName $AutomationAccountName 
    $workers | where-object {$PSItem.RunbookWorker.LastSeenDateTime -le $date} | foreach-object {
        Write-Host "Processing {$($PSItem.RunbookWorker.Name)}"
        $PSItem | Remove-AzureRmAutomationHybridWorkerGroup 
    }
}
#endregion

$servers = (($workers | where-object {$PSItem.RunbookWorker.LastSeenDateTime -le $date}).RunbookWorker).Name

#region Reload configuration and report back
Invoke-Command -ComputerName $servers -ErrorAction SilentlyContinue -ScriptBlock {
    $WorkspaceID=$USING:WorkspaceID
    $WorkspaceKey = $USING:WorkspaceKey

    #Create COM Object to manipulate MMA configuration
    $AgentCfg = New-Object -ComObject AgentConfigManager.MgmtSvcCfg
    # Remove desired OMS Workspace
    if($AgentCfg.GetCloudWorkspace($WorkspaceID)) {
        $AgentCfg.RemoveCloudWorkspace($WorkspaceID)
    }

    Stop-Service -ServiceName 'HealthService'
    Start-Sleep -Seconds 3
    #Remove files
    Remove-item -path 'C:\Program Files\Microsoft Monitoring Agent\Agent\Health Service State' -Force -confirm:$false -Recurse
    #Remove registry
    Get-ChildItem 'HKLM:\software\microsoft\hybridrunbookworker' | Remove-Item -Force -confirm:$false -Recurse -ErrorAction SilentlyContinue
    #let it rest a while. It was a hard task! :)
    Start-sleep -Seconds 5
    Start-Service -ServiceName 'HealthService'
    Start-Sleep -Seconds 3
    # Add OMS Workspace
    $AgentCfg.AddCloudWorkspace($WorkspaceID,$WorkspaceKey)
    $AgentCfg.ReloadConfiguration()
    Start-Sleep -seconds 3

    $AgentStatus = $AgentCfg.GetCloudWorkspaces() | select-object WorkspaceID,ConnectionStatus,ConnectionStatusText
    $ServiceStatus = Get-Service 'HealthService' | select-object Status,Name
    $HybridWorkerRegister = Get-ChildItem 'HKLM:\software\microsoft\hybridrunbookworker'
    [pscustomobject]@{
        ComputerName = $env:ComputerName
        AgentWorkspaceID = $AgentStatus.WorkspaceID
        AgentConnectionStatus = $AgentStatus.ConnectionStatus
        AgentConnectionStatusText = $AgentStatus.ConnectionStatusText
        ServiceName = $ServiceStatus.Name
        ServiceStatus = $ServiceStatus.Status
        HybridWorkerName = $HybridWorkerRegister.Name
    }
}
#endregion
```

## Summary

Sometimes clearing the cache only on the agent is not enough.

I still don't know what caused the issue where some of my workers stopped responding. I have my blind guess that it was a network fault at some point. It's always the network. Or DNS.

Anyway - my Azure Automation account is back in full swing and my schedules are patching and rebooting while I can do something else - like watch Ignite sessions!

![PatchTime]({{ site.url }}{{ site.baseurl }}/assets/images/posts/microsoft-monitoring-agent-take-2/picture5.png)

Cheers!
