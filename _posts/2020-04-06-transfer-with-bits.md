---
title: Simple file and folder transfer using Bits
categories:
    - Powershell
tags:
    - Tips
    - Bits
    - File Transfer
excerpt: How to transfer files and folder from remote machine using Bits 
---

# Bits and pieces

We all heard of Bits. It's the `demo` service most scripts fiddle with, when Windows services are concerned. But what it is and how we can benefit from it?

> BITS a.k.a Background Intelligent Transfer Service  
> Background Intelligent Transfer Service (BITS) is used by programmers and system administrators to download files from or upload files to HTTP web servers and SMB file shares.  
[docs.microsoft.com](https://docs.microsoft.com/en-us/windows/win32/bits/background-intelligent-transfer-service-portal)

For so many years I haven't got into using Bits in my PowerShell scripts. I either copied files directly to remote systems using shares, administrative shares (`c$, d$`) or PowerShell sessions and `Copy-Item -ToSession`. The main problem in all three is network. If it breaks or disconnects during the transfer - all is lost. Another issue with Copy-Item -ToSession is that it requires a lot of memory with big files.

And this time I had to copy 10GB file over a VPN connection with a flaky Internet. It ain't fun :grin:

## Collect crumbs

[Start-BitsTransfer](https://docs.microsoft.com/en-us/powershell/module/bitstransfer/start-bitstransfer?view=win10-ps) will be the command to use here. Bits can copy files to or from shares and HTTP/REST web servers. Administrative share (`c$`) is also a share but for some reason Bits doesn't like that, even if used with `-Credential` parameter.

BUT, it works when remote administrative share is mapped using [New-PSDrive](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/new-psdrive?view=powershell-7).

`Start-BitsTransfer` will not copy folders recursively. It can copy all files from given folder though. I will have to create folder manually - and thus I will loose security settings on those (thing to remember and to fix later on).

Finished transfer isn't finished, until we complete it. Weird, right? After the file is transfered, we need to complete it using [Complete-BitsTransfer](https://docs.microsoft.com/en-us/powershell/module/bitstransfer/complete-bitstransfer?view=win10-ps). I didn't know that and it took me a few minutes to figure it out. Well... I gave up after a few minutes and just went and read the docs.

So, what I will need to do?

- Map remote administrative drive using New-PSDrive and (optionaly) Credential parameter
- Get all files and folders from remote location
- Create all folders (recursively) in destination location
- Copy all files
- Complete the transfer

## Get the code

This will be `quick-and-dirty` script, but there are a few variables that will allow for more `general` usage:

```powershell
$RemoteComputer = 'RemoteServer1'
$RemotePath = 'C:\AdminTools'
$RemoteFile = 'TestFile.tmp'
#$RemoteFile = '*'
$LocalPath = 'C:\AdminTools\BitsTransfer'
$PSDriveLetter = 'K'
```

Through the script I will require three versions of `RemotePath`:

- local (`c:\AdminTools`)
- short administrative (`c$\AdminTools`)
- and full remote administrative (`\\RemoteServer1\c$\AdminTools`)

So this will creat it for me:

```powershell
$ParsedRemotePath = $RemotePath.Replace(':\','$\')
$ParsedRemotePathFinal = '\\{0}\{1}\' -f $RemoteComputer, $ParsedRemotePath
```

Map the hidden, remote administrative share as PSDrive:

```powershell
$PsDriveSplat = @{
    Name = $PSDriveLetter
    PSProvider = 'FileSystem'
    Root = '\\{0}\{1}' -f $RemoteComputer, $ParsedRemotePath
    Credential = $Credential
}
New-PSDrive @PsDriveSplat
```

Here's the fun part:
New-PSDrive will run with Root as `\\RemoteServer1\c$AdminTools\` BUT it won't accept it if `-Credential` or `-Persist` parameter is added. Then it requires Root WITHOUT trailing backslash. Look:

Trailing backslash and no -Credential:

![Works](/assets/images/posts/transfer-with-bits/picture1.png)

Trailing backslash with -Credential:
![Ops](/assets/images/posts/transfer-with-bits/picture2.png)

No trailing backslash with -Credential:
![Works](/assets/images/posts/transfer-with-bits/picture3.png)

Weird? That is why I'm creating the Root path above again without the `trailing backslash`.

I'll make sure we have our 'local' path created before the whole operation:

```powershell
if(-not (Test-Path $LocalPath -ErrorAction SilentlyContinue)) { New-Item -Path $LocalPath -ItemType Directory}
```

I want to allow copying a single file (`$RemoteFile = 'TestFile.tmp'`) or all files and folders from given location (`$RemoteFile = '*'`).  
I'll use a simple if.  
In case of a `*` I'll get all directories in remote share, create them locally and get all files full names.  
Even if I'll be listing them from `K:\` drive, the full name will be returned as `\\RemoteServer1\c$\AdminTools\TestFile.tmp`. 

![FilePath](/assets/images/posts/transfer-with-bits/picture4.png)

I will 'replace' the first part (`\\RemoteServer1\c$`) with an empty string :grin:

```powershell
$FilesToProcess = if ($RemoteFile -eq '*'){
    $SourcePath = '{0}:\' -f $PSDriveLetter
    $RemoteFiles = Get-ChildItem -Path $SourcePath -Recurse

    ($RemoteFiles | Where-Object { $PSItem.PSIsContainer -eq $true } | Select-Object -ExpandProperty FullName ).Replace($ParsedRemotePathFinal,'') | ForEach-Object {
        #Create folders locally
        if (-not (Test-Path (Join-Path $LocalPath -ChildPath $PSitem))) { [Void](New-Item -Path $LocalPath -Name $PSitem -ItemType Directory)}
    }
    #return only file names
    ($RemoteFiles | Where-Object { $PSItem.PSIsContainer -eq $false } |Select-Object -ExpandProperty FullName).Replace($ParsedRemotePathFinal,'')

}
else {
    #if RemoteFile is a file only, return it
    if(Test-Path ('{0}:\{1}' -f $PSDriveLetter , $RemoteFile) -PathType Leaf) { $RemoteFile }
}
```

Now the copy time:

```powershell
$BitsJobs = foreach ($File in $FilesToProcess) {
    $BitTransferSplat = @{
        Source = Join-Path -Path ('{0}:\' -f $PSDriveLetter ) -ChildPath $File
        Credential = $Credential
        Destination = Join-Path -Path $LocalPath -ChildPath $File
        Asynchronous = $true
    }
    Start-BitsTransfer @BitTransferSplat
}
```

After the transfer is finished, I'll complete it and remove created PSDrive:

```powershell
$BitsJobs | Get-BitsTransfer | Where-Object {$PSItem.JobState -ne 'Transferred'}

#When all files are copied we can Complete
$BitsJobs | Get-BitsTransfer | Where-Object {$PSItem.JobState -eq 'Transferred'} | Complete-BitsTransfer

#Remove the Drive
Get-PSDrive -Name $PsDriveSplat.Name | Remove-PSDrive
```

## Full script

The completed code looks like this:

```powershell
$Credential = Get-Credential
$RemoteComputer = 'RemoteServer1'
$RemotePath = 'C:\AdminTools'
$RemoteFile = 'TestFile.tmp'
#$RemoteFile = '*'
$LocalPath = 'C:\AdminTools\BitsTransfer'
$PSDriveLetter = 'K'


$ParsedRemotePath = $RemotePath.Replace(':\','$\')
$ParsedRemotePathFinal = '\\{0}\{1}\' -f $RemoteComputer, $ParsedRemotePath

$PsDriveSplat = @{
    Name = $PSDriveLetter
    PSProvider = 'FileSystem'
    Root = $ParsedRemotePathFinal
    Credential = $Credential
}

New-PSDrive @PsDriveSplat

if(-not (Test-Path $LocalPath -ErrorAction SilentlyContinue)) { New-Item -Path $LocalPath -ItemType Directory }

$FilesToProcess = if ($RemoteFile -eq '*'){
    $SourcePath = '{0}:\' -f $PSDriveLetter
    $RemoteFiles = Get-ChildItem -Path $SourcePath -Recurse

    #get only folders
    ( $RemoteFiles | Where-Object { $PSItem.PSIsContainer -eq $true } | Select-Object -ExpandProperty FullName ).Replace($ParsedRemotePathFinal,'') | ForEach-Object {
        #create folders locally
        if (-not (Test-Path (Join-Path $LocalPath -ChildPath $PSitem))) {
            [Void](New-Item -Path $LocalPath -Name $PSitem -ItemType Directory)
        }
    }
    #return only file names
    ($RemoteFiles | Where-Object { $PSItem.PSIsContainer -eq $false } |Select-Object -ExpandProperty FullName).Replace($ParsedRemotePathFinal,'')

}
else {
    #if RemoteFile is a file only, return it
    if(Test-Path ('{0}:\{1}' -f $PSDriveLetter , $RemoteFile) -PathType Leaf) { $RemoteFile }
}

$BitsJobs = foreach ($file in $FilesToProcess) {
    $BitTransferSplat = @{
        Source = Join-Path -Path ('{0}:\' -f $PSDriveLetter ) -ChildPath $File
        Credential = $Credential
        Destination = Join-Path -Path $LocalPath -ChildPath $File
        Asynchronous = $true
    }
    Start-BitsTransfer @BitTransferSplat
}

$BitsJobs | Get-BitsTransfer | Where-Object {$PSItem.JobState -ne 'Transferred'}

#When all files are copied we can Complete
$BitsJobs | Get-BitsTransfer | Where-Object {$PSItem.JobState -eq 'Transferred'} | Complete-BitsTransfer

#Remove the Drive
Get-PSDrive -Name $PsDriveSplat.Name | Remove-PSDrive
```

## Summary

This isn't complicated but got me a few _angry_ thoughts! Especially with that trailing backslash and the Complete-BitsTransfer :grin:.
