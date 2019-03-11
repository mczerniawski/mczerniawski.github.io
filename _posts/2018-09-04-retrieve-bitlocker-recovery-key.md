---
title: Retrieve BitLocker Recovery Key
categories:
    - BitLocker
tags:
    - PowerShell
    - Office365
    - ActiveDirectory
    - BitLocker
toc: true
toc_label: BitLocker
toc_sticky: true
---

# Quick intro

BitLocker is like backup. It's good to have it. It's better to have the restore verified as well. 

If you're planning to implement BitLocker into your organization (or already have that), it's good to know what's the choice of storing the recovery password:
- print 
- save to a file - either usb stick or unc share
- backup to ActiveDirectory 
- backup to Azure ActiveDirectory
- use MBAM

More information can be found [here](https://docs.microsoft.com/en-us/windows/security/information-protection/bitlocker/bitlocker-device-encryption-overview-windows-10). 

For me, the best approach would be to:

- use GPO to encrypt end user device AND store the password in Active Directory 
This can be configured here: *`Computer Configuration -> Policies -> Administrative Templates -> Windows Components -> BitLocker Drive Encryption -> Store BitLocker recovery information in Acive Directory Domain Services`*. 

This also ensures that encryption won't start if recovery key failed to be backed up to AD.

- use Intune and encrypt user device AND store the password in Azure Active Directory with self-service key recovery feature

This doesn't introduce the cost of MBAM or SCCM. 

What if you already have your drives encrypted, and now want to improve the process of recovering information? As always - `PowerShell` to the rescue. To send information to AD we can use `Backup-BitLockerKeyProtector`. It can accept either `KeyProtectorID` or the ID itself. Retrieving those is simple.

# Ways to get BitLocker recovery key information to AD and Azure AD

## Manage-BDE
We can get the information using manage-bde tool:
1. Retrieve information
![manage-bde retrieve](/assets/images/posts/retrieve-bitlocker/picture1.png)
2. Send to AD
![manage-bde backup](/assets/images/posts/retrieve-bitlocker/picture2.png)

## PowerShell
This is more fun (objects not strings!). Let's first get information about our volumes:

![Get-BitLocker volume](/assets/images/posts/retrieve-bitlocker/picture3.png)

As you can see I have only one drive, encrypted with TPM. To get the same information as before let's `select-object`
![Get-BitLocker details](/assets/images/posts/retrieve-bitlocker/picture4.png)

This returns two objects for each drive. We're interested in the second object
![Get-BitLocker details 2](/assets/images/posts/retrieve-bitlocker/picture5.png)

### Active Directory
Let's store the information to ActiveDirectory now:
![Get-BitLocker backup](/assets/images/posts/retrieve-bitlocker/picture6.png)

If I would have more drives, this would come in handy:

![More volumes](/assets/images/posts/retrieve-bitlocker/picture7.png) 

{% gist 1cb3e68370a65e12b43ffc6e0d767e7e %}

### Azure Active Directory
Same goes with sending RecoveryKey to Azure AD, this time with `BackupToAAD-BitLockerKeyProtector`:

![Backup to Azure AD](/assets/images/posts/retrieve-bitlocker/picture8.png)

{% gist e6847ad65ad3d1c3dfeda1ba6e2a2a1f %}

# Retrieve RecoveryKey

## From Active Directory

Now the best part - how to get the information back. Since Windows 2008 BitLocker Recovery Key is stored in AD in `msFVE-RecoveryInformation` objectclass aassociated to Computer. To get that we first need to get Computer Object and then search Active Directory for ObjecClass of given type. This is assuming your account have rights to read the information from AD in the first place! (great article [here](http://www.alexandreviot.net/2015/06/10/active-directory-how-to-display-bitlocker-recovery-key/))

![Retrieve key](/assets/images/posts/retrieve-bitlocker/picture9.png)

{% gist 0f42b91b7bee6033474463f49a43bf3e %}

I'm using custom objects for better readability. The date shows when the drive was *encrypted*!, not when the information was backed up.

## From Azure AD
There is a delay between using `BackupToAAD-BitLockerKeyProtector` and the information showing on AzureAD. Give it time to synchronize :)

Navigate to https://myapps.microsoft.com, go to the `Profile` page and see all the registered devices:

![Profile page](/assets/images/posts/retrieve-bitlocker/picture10.png)

From there You can view the recovery password for You devices.
Btw - it's not very intuitive - You cannot access this informatorom directly from office.com -> profie pages. 

