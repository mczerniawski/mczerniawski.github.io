---
title: PowerShell 7 and the Remote Endpoint
categories:
    - PowerShell
tags:
    - PowerShell7
    - Remoting
excerpt: How to connect to remote PowerShell 7 endpoint?
---

# PowerShell 7 is here

So, everyone are talking about PowerShell 7 being `the new black`.  There are a lot of improvements with the [stable release](https://devblogs.microsoft.com/powershell/announcing-powershell-7-0/)

We will be having our [Polish PowerShell User Group Meetup - `Fully Remote` on 21st of April](https://www.meetup.com/Polish-PowerShell-Group-PPoSh/events/268652884/).

## Get PowerShell 7

There are various ways you can install PowerShell on your Windows machine:

- Using PowerShell and MSI

```powershell
iex "& { $(irm https://aka.ms/install-powershell.ps1) } -UseMSI"
```

- Using Chocolatey

```powershell
choco install pwsh
```

- Using standalone installer with GUI - this you can get from the [GitHub release page](https://github.com/PowerShell/PowerShell/releases)

## Remoting to the box

All is fun and working on my machine, but I'm mainly remoting. I try to connect to one of my test servers:

```powershell
# That's my machine
PS C:\AdminTools> $PSVersionTable

Name                           Value
----                           -----
PSVersion                      7.0.0
PSEdition                      Core
GitCommitId                    7.0.0
OS                             Microsoft Windows 10.0.14393
Platform                       Win32NT
PSCompatibleVersions           {1.0, 2.0, 3.0, 4.0…}
PSRemotingProtocolVersion      2.3
SerializationVersion           1.1.0.1
WSManStackVersion              3.0

PS C:\AdminTools> Invoke-Command -ComputerName TestServer1 -ScriptBlock {$PSVersionTable}

#And that's remote
Name                           Value
----                           -----
CLRVersion                     4.0.30319.42000
PSRemotingProtocolVersion      2.3
PSVersion                      5.1.14393.3471
PSEdition                      Desktop
WSManStackVersion              3.0
PSCompatibleVersions           {1.0, 2.0, 3.0, 4.0…}
SerializationVersion           1.1.0.1
BuildVersion                   10.0.14393.3471

```

> Because I'm running `Windows 2016` I get the default `Windows PowerShell 5.1` endpoint. 

Let's install PowerShell 7 on the remote endpoint:

```powershell
PS C:\AdminTools> Invoke-Command -ComputerName TestServer1 -ScriptBlock {iex "& { $(irm https://aka.ms/install-powershell.ps1) } -UseMSI"}
VERBOSE: About to download package from 'https://github.com/PowerShell/PowerShell/releases/download/v7.0.0/PowerShell-7.0.0-win-x64.msi'
```

Now let's try and remote to the server, what will happen?:

```powershell
PS C:\AdminTools> Invoke-Command -ComputerName TestServer1 -ScriptBlock {$PSVersionTable}

Name                           Value
----                           -----
CLRVersion                     4.0.30319.42000
PSRemotingProtocolVersion      2.3
PSVersion                      5.1.14393.3471
PSEdition                      Desktop
WSManStackVersion              3.0
PSCompatibleVersions           {1.0, 2.0, 3.0, 4.0…}
SerializationVersion           1.1.0.1
BuildVersion                   10.0.14393.3471

```

> Hm. I didn't expect that.

Quick search and there's an [open issue for this already](https://github.com/PowerShell/PowerShell/issues/11616).

As you read through the conversation in that issue it is by design. To maintain legacy compatibility - even when running PowerShell 7 on your machine you connect to default endpoint which is PowerShell 5.1 (in this case).

## Trials and errors

We can force remote connection to use PowerShell 7, but running it like this is... sub-optimal:

```powershell
PS C:\AdminTools> Invoke-Command TestServer1 -ScriptBlock { pwsh -command {$env:ComputerName ; $PSVersionTable} }
TestServer1

Name                           Value
----                           -----
PSVersion                      7.0.0
PSEdition                      Core
GitCommitId                    7.0.0
OS                             Microsoft Windows 10.0.14393
Platform                       Win32NT
PSCompatibleVersions           {1.0, 2.0, 3.0, 4.0}
PSRemotingProtocolVersion      2.3
SerializationVersion           1.1.0.1
WSManStackVersion              3.0
```

To connect to remote PowerShell 7 endpoint you first need to create it. Run PowerShell 7 and then `Enable-PSRemoting`.

```powershell
PS C:\AdminTools> Invoke-Command TestServer1 -ScriptBlock { pwsh -command {$env:ComputerName ; Enable-PSRemoting -Force }}
TestServer1
WARNING: PowerShell remoting has been enabled only for PowerShell 6+ configurations and does not affect Windows PowerShell remoting configurations. Run this cmdlet in Windows PowerShell to affect all PowerShell remoting configurations.
WinRM is already set up to receive requests on this computer.

OpenError: [TestServer1] Processing data from remote server TestServer1 failed with the following error message: The I/O operation has been aborted because of either a thread exit or an application request. For more information, see the about_Remote_Troubleshooting Help topic.
```

As you can see - it errors out at the end. It tries to configure `Remoting` but stops the service and is unable to process. Also, as warning states above **it enables remoting for PowerShell 6+ and does not affect default PowerShell 5.1**

## Trials continues

So I've started the service and tried different ways. I've even tried with `WinRM` command as well:

```powershell
WinRS -r:TestServer1 -noprofile pwsh -command {Enable-PSremoting -force}
```

But it also stucks. When I RDP to the server and check the service status this is what I get:

```powershell
PS C:\AdminTools> Get-Service winrm

Status   Name               DisplayName
------   ----               -----------
Stopped  winrm              Windows Remote Management (WS-Managem…
```

Starting it allows me to connect remotely again but PowerShell.7 endpoint is still not created:

```powershell
PS C:\AdminTools> Invoke-Command TestServer1 -ScriptBlock { Get-PSSessionConfiguration | Select-Object Name,PSVersion}

Name           : microsoft.powershell
PSVersion      : 5.1
PSComputerName : TestServer1
RunspaceId     : a91fbf3d-3d39-4623-99eb-3786c1b64927

Name           : microsoft.powershell.workflow
PSVersion      : 5.1
PSComputerName : TestServer1
RunspaceId     : a91fbf3d-3d39-4623-99eb-3786c1b64927

Name           : microsoft.powershell32
PSVersion      : 5.1
PSComputerName : TestServer1
RunspaceId     : a91fbf3d-3d39-4623-99eb-3786c1b64927

Name           : microsoft.windows.servermanagerworkflows
PSVersion      : 3.0
PSComputerName : TestServer1
RunspaceId     : a91fbf3d-3d39-4623-99eb-3786c1b64927
```

## FIX

During my trials I've though about different ways of achieving it:

- schedule a task with PowerShell 7 script to run  `Enable-PSRemoting -Force`
- connect to each machine with RDP, launch Pwsh.exe as admin and `Enable-PSRemoting -Force`
- use psexec.exe

```powershell
PS C:\AdminTools> .\PsExec.exe \\TestServer1 pwsh.exe -command {Enable-PSRemoting -Force}
```

This works as it doesn't use WinRM to connect and... configure WinRM :smile:

But finally I've found that I can just run a script ( `Install-PowerShellRemoting.ps1` ) from PowerShell Team :grin:. That script is located in the install path of PowerShell 7.

```powershell
PS C:\AdminTools> Invoke-Command -ComputerName TestServer1 -ScriptBlock { pwsh -command {& 'C:\Program Files\PowerShell\7\Install-PowerShellRemoting.ps1'}}

PS C:\AdminTools> Invoke-Command TestServer1 -ScriptBlock { pwsh {Get-PSSessionConfiguration | Select-Object Name,PSVersion} }

Name           : PowerShell.7
PSVersion      : 7.0
PSComputerName : TestServer1
RunspaceId     : 6b63b9ce-ddf5-4a5c-8869-74c1404f967f

Name           : PowerShell.7.0.0
PSVersion      : 7.0
PSComputerName : TestServer1
RunspaceId     : 6b63b9ce-ddf5-4a5c-8869-74c1404f967f
```

We can see that with Windows PowerShell 5.1 as well:

```powershell
PS C:\AdminTools> Invoke-Command TestServer1 -ScriptBlock  {Get-PSSessionConfiguration | Select-Object Name,PSVersion}


Name           : microsoft.powershell
PSVersion      : 5.1
PSComputerName : TestServer1
RunspaceId     : 9dab3418-01ac-4beb-b70b-77cd1e2eafe5

Name           : microsoft.powershell.workflow
PSVersion      : 5.1
PSComputerName : TestServer1
RunspaceId     : 9dab3418-01ac-4beb-b70b-77cd1e2eafe5

Name           : microsoft.powershell32
PSVersion      : 5.1
PSComputerName : TestServer1
RunspaceId     : 9dab3418-01ac-4beb-b70b-77cd1e2eafe5

Name           : microsoft.windows.servermanagerworkflows
PSVersion      : 3.0
PSComputerName : TestServer1
RunspaceId     : 9dab3418-01ac-4beb-b70b-77cd1e2eafe5

Name           : PowerShell.7
PSVersion      : 7.0
PSComputerName : TestServer1
RunspaceId     : 9dab3418-01ac-4beb-b70b-77cd1e2eafe5

Name           : PowerShell.7.0.0
PSVersion      : 7.0
PSComputerName : TestServer1
RunspaceId     : 9dab3418-01ac-4beb-b70b-77cd1e2eafe5
```

## It's alive

Now I can connect from PowerShell 7 (running local) to PowerShell 7 (running remote) and benefit from all the goodies in both environments. I just need to use the ConfigurationName `PowerShell.7`

```powershell
PS C:\AdminTools> $PSVersionTable

Name                           Value
----                           -----
PSVersion                      7.0.0
PSEdition                      Core
GitCommitId                    7.0.0
OS                             Microsoft Windows 10.0.14393
Platform                       Win32NT
PSCompatibleVersions           {1.0, 2.0, 3.0, 4.0…}
PSRemotingProtocolVersion      2.3
SerializationVersion           1.1.0.1
WSManStackVersion              3.0

PS C:\AdminTools> Invoke-Command TestServer1 -ConfigurationName 'PowerShell.7' -ScriptBlock {$PSVersionTable}

Name                           Value
----                           -----
PSRemotingProtocolVersion      2.3
GitCommitId                    7.0.0
PSVersion                      7.0.0
OS                             Microsoft Windows 10.0.14393
WSManStackVersion              3.0
PSEdition                      Core
SerializationVersion           1.1.0.1
PSCompatibleVersions           {1.0, 2.0, 3.0, 4.0…}
Platform                       Win32NT
```

I can also do it from Windows PowerShell 5.1 (local) to remote PowerShell 7 - which is obvious. This is just another `PowerShell` endpoint we can use:

```powershell
PS C:\AdminTools> $PSVersionTable

Name                           Value
----                           -----
PSVersion                      5.1.14393.3471
PSEdition                      Desktop
PSCompatibleVersions           {1.0, 2.0, 3.0, 4.0...}
BuildVersion                   10.0.14393.3471
CLRVersion                     4.0.30319.42000
WSManStackVersion              3.0
PSRemotingProtocolVersion      2.3
SerializationVersion           1.1.0.1


PS C:\AdminTools> Invoke-Command TestServer1 -ConfigurationName 'PowerShell.7' -ScriptBlock {$PSVersionTable}

Name                           Value
----                           -----
PSRemotingProtocolVersion      2.3
Platform                       Win32NT
PSVersion                      7.0.0
PSCompatibleVersions           {1.0, 2.0, 3.0, 4.0...}
OS                             Microsoft Windows 10.0.14393
PSEdition                      Core
WSManStackVersion              3.0
GitCommitId                    7.0.0
SerializationVersion           1.1.0.1
```

## Summary

I do understand why PowerShell.7 endpoint is not the default one after you install PowerShell.
I don't understand though why I can't enabel