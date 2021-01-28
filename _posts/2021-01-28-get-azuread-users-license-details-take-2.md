---
title: Get AzureAD Users license details - take 2
categories:
    - Powershell
tags:
    - AzureAD
    - Office365
    - PowerShell
    - GraphAPI
excerpt: A bumpy road to get all users license assignment - second attempt! 
toc: true
toc_label: AzureAD users license usage
---

# Get AzureAD Users license details - take 2

Long time ago, in a ~~galaxy~~ [blog post](https://www.mczerniawski.pl/office365/get-azuread-license-membership/) far away I described how to get license details for all your AzureAD users.

It wasn't possible back then (25-01-2019) to use a PowerShell cmdlet to answer a question:

> Tell me of all users in my tenant who has license `EMS E3` assigned.

You could though answer these questions:

- You can get the information IF a userA has `a license` assigned
- You can get the information WHAT `licenses` userA has assigned
- You can get the information IF `licenseX` is assigned (but you will not know who uses it)

I've ended up building a custom solution that required a few steps:

- getting all users and their license details
- getting all licenses and matching them to more ... sensible names than these: `MCOMEETADV` doesn't ring a bell for me :smile: (this is Audio Conferencing license!)
- building a hash table with all results grouped by SKU

Then (2 years ago) I wanted to try with MS Graph API, wondering if that would be easier. Now I've come to revisit the idea. Bear with me, as this won't be as quick as 3 cmdlets! Oh, and forgive me a bit of rant here and there :(

## Set up the stage

My previous solution wasn't ideal. It returned only SKUs for licenses like `Azure AD Premium P1`. This can come in various ServicePlans (like `EM+S E3` or standalone). Those ServicePlans can be managed by assigning a user directly or by AzureAD Groups.

This time I wanted to get:

- all ServicePlans (so commercial products you can purchase)
- all groups assigned to those service plans
- users that have a service plan assigned
- information if the assignment is direct or through AzureAD group

### The research

I checked what changed in the past 2 years regarding this. I ~~belived~~ hoped, that such a basic functionality is achievable without much trouble. I've tried with available PowerShell cmdlets. There are some that looked promising:

```powershell
Get-Command get-*license*
CommandType     Name                                               Version    Source
-----------     ----                                               -------    ------
Cmdlet          Get-AzureADUserLicenseDetail                       2.0.2.128  AzureAD
Cmdlet          Get-AzureADUserLicenseDetail                       2.0.2.17   AzureADPreview
Cmdlet          Get-Groups_MembersWithLicenseErrors                6.1907.1.0 Microsoft.Graph.Intune
Cmdlet          Get-Groups_MembersWithLicenseErrorsReferences      6.1907.1.0 Microsoft.Graph.Intune
```

Still id did not provide the most necessary parts. It still requires enumerating through all users and no information if the license is assigned to a user by a group or directly exists.

I've looked through Microsoft Docs pages:

- First obvious link was this: [How to migrate users with individual licenses to groups for licensing](https://docs.microsoft.com/en-us/azure/active-directory/enterprise-users/licensing-groups-migrate-users). But there was no code examples. Two pictures. Right ...

- What about this one? [View licensed and unlicensed Microsoft 365 users with PowerShell](https://docs.microsoft.com/en-us/microsoft-365/enterprise/view-licensed-and-unlicensed-users-with-microsoft-365-powershell)? There's a nice example with PowerShell "to view the list of all user accounts in your organization that have NOT been assigned any of your licensing plans (unlicensed users), run the following command:"

```powershell
Get-AzureAdUser | ForEach{ $licensed=$False ; For ($i=0; $i -le ($_.AssignedLicenses | Measure).Count ; $i++) { If( [string]::IsNullOrEmpty(  $_.AssignedLicenses[$i].SkuId ) -ne $True) { $licensed=$true } } ; If( $licensed -eq $false) { Write-Host $_.UserPrincipalName} }
```

This is an example from Microsoft documentation site. I'll just leave it here... speachless.

Anyway, back to the research:

- So maybe this one? [PowerShell and Graph examples for group-based licensing in Azure AD](https://docs.microsoft.com/en-us/azure/active-directory/enterprise-users/licensing-ps-examples) - there are some examples, but they use an old module `MSOnline`:

> Note: this is the older MSOnline V1 PowerShell module for Azure Active Directory. Customers are encouraged to use the newer Azure Active Directory V2 PowerShell module instead of this module. For more information about the V2 module please see Azure Active Directory V2 PowerShell.

It contained though, exactly the bread crumbs I required:

 - [View product licenses assigned to a group](https://docs.microsoft.com/en-us/azure/active-directory/enterprise-users/licensing-ps-examples#view-product-licenses-assigned-to-a-group)
 - [Get all groups with licenses](https://docs.microsoft.com/en-us/azure/active-directory/enterprise-users/licensing-ps-examples#get-all-groups-with-licenses)
 - [Check if user license is assigned directly or inherited from a group](https://docs.microsoft.com/en-us/azure/active-directory/enterprise-users/licensing-ps-examples#check-if-user-license-is-assigned-directly-or-inherited-from-a-group)

Still not the `THING`. Yet there was a very important one line:

```http
GET https://graph.microsoft.com/v1.0/users/e61ff361-5baf-41f0-b2fd-380a6a5e406a?$select=licenseAssignmentStates
```

Apparently `I CAN` get the assignment state:

```json
HTTP/1.1 200 OK
{
  "value":[
    {
      "odata.type": "Microsoft.DirectoryServices.User",
      "objectType": "User",
      "id": "e61ff361-5baf-41f0-b2fd-380a6a5e406a",
      "licenseAssignmentState":[
        {
          "skuId": "157870f6-e050-4b3c-ad5e-0f0a377c8f4d",
          "disabledPlans":[],
          "assignedByGroup": null, # assigned directly.
          "state": "Active",
          "error": "None"
        },
        {
          "skuId": "1f3174e2-ee9d-49e9-b917-e8d84650f895",
          "disabledPlans":[],
          "assignedByGroup": "e61ff361-5baf-41f0-b2fd-380a6a5e406a", # assigned by this group.
          "state": "Active",
          "error": "None"
        }
```

> Graph doesn’t have a straightforward way to show the result, but it can be seen from this API (look above)

Although I've checked - there is no other way to get Assignment state than with Graph API. If it's otherwise - please prove me wrong!

At this point - I've decided to go full MS Graph API. Some of the information could be retrieved using AzureAD module, but this way (pure API) gives me more flexibility!

### Friendly Names

What about product licenses? Last time I've built a custom csv file with some of the SKUs translated. It wasn't accurate though. This time I've decided to lean on Microsoft. They have a resource with all [Product names and service plan identifiers for licensing](https://docs.microsoft.com/en-us/azure/active-directory/enterprise-users/licensing-service-plan-reference) ... in a form of a markdown table! (as you will see not all...:D)

I had to edit the page, get the markdown table and transfrom it to a csv file by removing all unnecessary characters and additional spaces at the beggining and end of SOME entries! Ouch!

It isn't... ideal but hey. I've done it so you don't have to :grin:. You can find the csv on my [github gist](https://gist.github.com/mczerniawski/950e5c102c704d628ce38522ef4ad0f9). The RAW form of this will be used in the script with a [bit.ly link](http://bit.ly/SKUFriendlyNames)

---

## Start the journey with MS Graph API

If you've used Graph API you can skip to the next section. If not - You'll need to perform a few steps first:

- create an AzureAD application and generate a secret (a password)
- assign permissions to query necessary resources

### Create an AzureAD application

To access the API of your tenant you need to authenticate. You can't use your own account though - create an AzureAD application.

1. Go to Azure Portal - [the Azure Active Directory Blade](https://aad.portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/Overview)
2. The go to `App registrations` and click `New Registration`  
![AppRegister]({{ site.url }}{{ site.baseurl }}/assets/images/posts/get-azuread-license-2/picture1.png)  
3. Enter a name and `Register`  
![AppRegister1]({{ site.url }}{{ site.baseurl }}/assets/images/posts/get-azuread-license-2/picture2.png)  
4. Note down the Application (client) ID (a username) and then go to `Certificates & secrets` to create a secret (a password)  
![AppRegister1]({{ site.url }}{{ site.baseurl }}/assets/images/posts/get-azuread-license-2/picture3.png)  
5. Once you create a secret - note it down. You can copy it only NOW:  
![AppRegister2]({{ site.url }}{{ site.baseurl }}/assets/images/posts/get-azuread-license-2/picture4.png)

### Assign permissions

Same as with regular accounts - you need to assign proper permissions. For Graph API purpose you should use [MS Docs](https://docs.microsoft.com/en-us/graph/api/overview?view=graph-rest-1.0) to find out and assign only the necessary ones!

Go to the `Api permissions` section (below the `Certificates and secrets`). For the purpose of these scripts you will need to assign permissions to:

- read user data
- read group data
- read organization data (to get current SKUs you're subscribed to)

To do so, click `Add a permission`, select `Microsoft Graph` and `application permissions`. These are the permissions I have assigned:  
![ApiPermissions]({{ site.url }}{{ site.baseurl }}/assets/images/posts/get-azuread-license-2/picture5.png)  
Once done, click `grant admin consent`.

I tend to `test` my Grap API query first using [Graph-Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer). This way I know the query works before I start disassembling my code :)

Now, you're ready to query Graph API. Finally - off to some coding!

## Are we there yet?

One last rant part. I come from PowerShell world. Windows PowerShell in particular. I'm used to getting a large amount of objects with cmdlets without much problem. Well, it seems it's not that easy with Graph API.

If a query results in more than 100 objects you get only first 100 ... and an object - `@odata.nextLink` - with an Uri that retrieves the next 100... and so on. I cannot use something like -All on the Invoke-RestMethod. It's called [paging](https://docs.microsoft.com/en-us/graph/paging).

I've tried bulding a recurrence function, but for some reason it kept asking the first NextLink without iterating further. It wasn't an issue with recurrence though!

Based on [Daniel Chronlund's blog](https://danielchronlund.com/2020/02/26/my-collection-of-basic-microsoft-graph-powershell-functions/) I've built a wrapper to Invoke-RestMethod that uses `while` loop to process the requests. It's quick and dirty but get's the job done:

```powershell
function Invoke-RecurenceRestMethod {
    param (
        $Uri,
        $Headers,
        $Method = 'Get',
        $ContentType = "application/json",
        $UseBasicParsing = $true
    )
    process {
        $irmSplat = @{
            Uri = $Uri
            Headers = $Headers
            Method = $Method
            ContentType = $ContentType
            UseBasicParsing = $UseBasicParsing
        }
        Write-Verbose ('Processing URI {0}' -f $irmSplat.Uri)
        
        $QueryRequest = @()
        $QueryResult = @()

        $QueryRequest = Invoke-RestMethod @irmSplat      
        if ($QueryRequest.value) {
            $QueryResult += $QueryRequest.value
        } else {
            $QueryResult += $QueryRequest
        }
        # Invoke REST methods and fetch data until there are no pages left.
        if ($Uri -notlike "*`$top*") {
            while ($QueryRequest.'@odata.nextLink') {
                
                $irmSplat.Uri = $QueryRequest.'@odata.nextLink'
                Write-Verbose ('Processing URI {0}' -f $irmSplat.Uri)
                $QueryRequest = Invoke-RestMethod @irmSplat
                $QueryResult += $QueryRequest.value
            }
        }
        $QueryResult
    }
}
#endregion 
```

## Code logic

Let me break down the code logic first:

1. Get Graph API token to use in the queries
2. Get current product licenses assigned to the tenant
3. Get all groups witch are used to assign licenses
4. Get all users
5. Import SKU friendly names from my gist
6. Parse all data into array of PowerShell custom objects
7. Slice and dice the data to your liking

### Get Graph API token

```powershell
$ApplicationID = 'b2b1c3e7-9125-4325.........'
$TenatDomainName = 'arcontest.onmicrosoft.com'
$AccessSecret = 'F3kpcBybdw1QE.......'
$Scope = "https://graph.microsoft.com/.default"


$LoginUrl = "https://login.microsoftonline.com/$TenatDomainName/oauth2/v2.0/token"
Add-Type -AssemblyName System.Web

$Body = @{
    client_id = $ApplicationID
	client_secret = $AccessSecret
	scope = $Scope
	grant_type = 'client_credentials'
}

$PostSplat = @{
    ContentType = 'application/x-www-form-urlencoded'
    Method = 'POST'
    Body = $Body
    Uri = $LoginUrl
}

$Request = Invoke-RestMethod @PostSplat

$Headers = @{
    Authorization = "$($Request.token_type) $($Request.access_token)"
}
```

A small test to check whether the query for `users` endpoint works:

```powershell
$Uri = 'https://graph.microsoft.com/v1.0/users'

$irmResult = Invoke-RestMethod -Uri $Uri -Headers $Headers -Method Get -ContentType "application/json"
$irmResult
```

### Get current product licenses

```powershell
$Uri = 'https://graph.microsoft.com/v1.0/subscribedSkus'
$SKUsResponse =  Invoke-RestMethod -Uri $Uri -Headers $Headers -Method Get -ContentType "application/json" 
$SKUs = if ($SKUsResponse) { $SKUsResponse | Select-Object -expand Value }
```

### Get all groups witch are used to assign licenses

If an AzureAD (or Active Directory synchronized) group is used to assign licenses it gets an attribute (`assignedLicenses`) which can be used to filter from other groups.

As you can notice I'm using my function to ask for all records. Because `assignedLicenses` is not retrieved by default, I need to get it using `$select` statement - thus building a Graph API Uri:

```powershell
$Uri = 'https://graph.microsoft.com/v1.0/groups?$select=id,displayName,assignedLicenses' 
$groupIrmResult = Invoke-RecurenceRestMethod -Uri $uri -Headers $headers
$LicensedGroups = if ($groupIrmResult) { $groupIrmResult.where({$PSitem.assignedLicenses}) }
```

### Get all users

One thing to clarify - all my employees have the `employeeid` attribute populated. If you want to get all users or use a different filter, this is the place to change it - modify the `$filter` variable.

Because the Uri contains both '' and $ inside the Uri, I need to `escape it`.  
If I would use "" for the whole Uri then `$filter` and `$select` would be considered as variables.  
If I would use '' then I couldn't go with `startsWith` statement which also requires to surround the searchable string with ''  
..... :grin:

```powershell
$filter = "startsWith(employeeId,'emp0'"
$Uri = 'https://graph.microsoft.com/v1.0/users?$filter={0})&$select=displayName,employeeID,licenseAssignmentStates' -f $filter
$AllEmployees = (Invoke-RecurenceRestMethod -Uri $uri -Headers $headers)
```

### Import SKU friendly names from my gist

Now, you can either download the csv by yourself (never trust a random guy from the Internet) from here [github gist](https://gist.github.com/mczerniawski/950e5c102c704d628ce38522ef4ad0f9) or use the code below to import the data:

```powershell
$UriGist = 'http://bit.ly/SKUFriendlyNames'
$SKUFriendlyNames = Invoke-RestMethod -Uri $UriGist -UseBasicParsing | Convertfrom-Csv -Delimiter ';' 
```

### Parse all data into arrays of PowerShell custom objects

Now all the necessary parts are in place. Combine it!

> "By the power of Grayskull... ...I have the power!"

```powershell
$output = foreach ($row in $AllEmployees.where({$PSitem.licenseAssignmentStates})) {
    [pscustomobject]@{
       UserName = $row.displayName
       EmployeeID = $row.EmployeeID
       Licenses = foreach ($license in $row.licenseAssignmentStates) {
           [pscustomobject]@{
               LicenseSKUID = $license.skuID
               LicensePartNumber = ($SKUs.where({$PSItem.skuid -eq $license.skuid})).skuPartNumber
               ProductName = $SKUFriendlyNames.where({$PSitem.LicenseSKUID -eq $license.skuid }).ProductName
               assignedbyGroup = if($license.assignedbyGroup) { $true } else {$false}
               assignedbyGroupName = ($LicensedGroups.where({$PSitem.id -eq $license.assignedbyGroup})).displayName
           }
   
       }
   
    }
}
```

### Slice and dice the data to your liking

This is the last part. Now if I want to find out all users with a license, I can either use its SKUName or full, friendly Product Name

```powershell
#SKU Name
$groupName = 'EMSPREMIUM'
$Details = $output.where({$PSItem.Licenses.LicensePartNumber -ieq $groupName}) 
#Friendly Name
$FriendlyGroupName = 'ENTERPRISE MOBILITY + SECURITY E5'
$Details = $output.where({$PSItem.Licenses.ProductName -ieq $FriendlyGroupName}) 

$Details

UserName                  EmployeeID Licenses
--------                  ---------- --------
User A               emp00006   {@{LicenseSKUID=26d45bd9-adf1-46cd-a9e1-51e9a5524128; LicensePartNumber=ENTERPRISEPREMIUM_NOPSTNCONF; ProductName=OFFICE 365 E5 WITHOUT AUDIO CONFERENCING; assignedbyGroup=True; assignedbyGroupName=Cloud_AudioConferencin...
User B               emp00007   {@{LicenseSKUID=26d45bd9-adf1-46cd-a9e1-51e9a5524128; LicensePartNumber=ENTERPRISEPREMIUM_NOPSTNCONF; ProductName=OFFICE 365 E5 WITHOUT AUDIO CONFERENCING; assignedbyGroup=True; assignedbyGroupName=Cloud_AudioConferencin... 
User C               emp00009   {@{LicenseSKUID=26d45bd9-adf1-46cd-a9e1-51e9a5524128; LicensePartNumber=ENTERPRISEPREMIUM_NOPSTNCONF; ProductName=OFFICE 365 E5 WITHOUT AUDIO CONFERENCING; assignedbyGroup=True; assignedbyGroupName=Cloud_AudioConferencin... 
User D               emp00010   {@{LicenseSKUID=b05e124f-c7cc-45a0-a6aa-8cf78c946968;
```

If I want to get WHAT licenses a user has and the way of assignment here's the code:

```powershell
$UserName = 'Mateusz Czerniawski'
$Details.where({$PSitem.UserName -eq $UserName}).Licenses | Format-Table

LicenseSKUID                         LicensePartNumber            ProductName                              assignedbyGroup assignedbyGroupName
------------                         -----------------            -----------                              --------------- -------------------
b05e124f-c7cc-45a0-a6aa-8cf78c946968 EMSPREMIUM                   ENTERPRISE MOBILITY + SECURITY E5                   True Cloud_EMS_E5
26d45bd9-adf1-46cd-a9e1-51e9a5524128 ENTERPRISEPREMIUM_NOPSTNCONF OFFICE 365 E5 WITHOUT AUDIO CONFERENCING            True Cloud_Office365_E5
c5928f49-12ba-48f7-ada3-0d743a3601d5 VISIOCLIENT                  VISIO Online Plan 2                                 True Cloud_Visio_Plan2
a403ebcc-fae0-4ca2-8c8c-7a907fd6c235 POWER_BI_STANDARD            POWER BI (FREE)                                     True Cloud_PowerBI_Free
f30db892-07e9-47e9-837c-80727f46fd3d FLOW_FREE                    MICROSOFT FLOW FREE                                 True Cloud_PowerAutomate_Free
f8a1db68-be16-40ed-86d5-cb42ce701560 POWER_BI_PRO                 POWER BI PRO                                        True Cloud_PowerBI_Pro
1e7e1070-8ccb-4aca-b470-d7cb538cb07e WIN_ENT_E5                                                                       True Cloud_Windows10Enterprise_E5
```

Did you notice no `Windows 10 Enterprise E5` friendly name (ProductName column)? I have it assigned as you can see with LicensePartNumber `WIN_ENT_E5` and SKUID `1e7e1070-8ccb-4aca-b470-d7cb538cb07e`.
In my csv there is an entry `Windows 10 Enterprise E5` with LicensePartNumber  `WIN10_VDA_E5` and SKUID `488ba24a-39a9-4473-8ee5-19291e71b002`.

I've double checked [MS site](https://docs.microsoft.com/en-us/azure/active-directory/enterprise-users/licensing-service-plan-reference0) - maybe I've made a mistake when cleaning up the table. But no, I've made no mistake here. Their site also states Windows 10 Enterprise E5 with LicensePartNumber `WIN10_VDA_E5`.

> Magic!

## Full Code

If you'd like to use this code here's a full listing :grin:. No need to copy-paste from different parts. Remember to change the tenant name, application ID and password though:

```powershell
#region Function
function Invoke-RecurenceRestMethod {
    param (
        $Uri,
        $Headers,
        $Method = 'Get',
        $ContentType = "application/json"
    )
    process {
        $irmSplat = @{
            Uri = $Uri
            Headers = $Headers
            Method = $Method
            ContentType = $ContentType
            UseBasicParsing = $true
        }
        Write-Verbose ('Processing URI {0}' -f $irmSplat.Uri)
        
        $QueryRequest = @()
        $QueryResult = @()

        $QueryRequest = Invoke-RestMethod @irmSplat      
        if ($QueryRequest.value) {
            $QueryResult += $QueryRequest.value
        } else {
            $QueryResult += $QueryRequest
        }
        # Invoke REST methods and fetch data until there are no pages left.
        if ($Uri -notlike "*`$top*") {
            while ($QueryRequest.'@odata.nextLink') {
                
                $irmSplat.Uri = $QueryRequest.'@odata.nextLink'
                Write-Verbose ('Processing URI {0}' -f $irmSplat.Uri)
                $QueryRequest = Invoke-RestMethod @irmSplat
                $QueryResult += $QueryRequest.value
            }
        }
        $QueryResult
    }
}
#endregion 

#region Get Token
$ApplicationID = 'b2b1c3e7-9125-4325.........'
$TenatDomainName = 'arcontest.onmicrosoft.com'
$AccessSecret = 'F3kpcBybdw1QE.......'
$Scope = "https://graph.microsoft.com/.default"


$LoginUrl = "https://login.microsoftonline.com/$TenatDomainName/oauth2/v2.0/token"
Add-Type -AssemblyName System.Web

$Body = @{
    client_id = $ApplicationID
	client_secret = $AccessSecret
	scope = $Scope
	grant_type = 'client_credentials'
}

$PostSplat = @{
    ContentType = 'application/x-www-form-urlencoded'
    Method = 'POST'
    Body = $Body
    Uri = $LoginUrl
}

$Request = Invoke-RestMethod @PostSplat

$Headers = @{
    Authorization = "$($Request.token_type) $($Request.access_token)"
}

#endregion

#region Test
$Uri = 'https://graph.microsoft.com/v1.0/users'

$irmResult = Invoke-RestMethod -Uri $Uri -Headers $Headers -Method Get -ContentType "application/json"
$irmResult

#endregion

#region License SKUs
$Uri = 'https://graph.microsoft.com/v1.0/subscribedSkus'
$SKUsResponse =  Invoke-RestMethod -Uri $Uri -Headers $Headers -Method Get -ContentType "application/json" 
$SKUs = if ($SKUsResponse) { $SKUsResponse| Select-Object -expand Value }
#endregion

#region Get All Groups with Licensing assigned

$Uri = 'https://graph.microsoft.com/v1.0/groups?$select=id,displayName,assignedLicenses' 
$groupIrmResult = Invoke-RecurenceRestMethod -Uri $uri -Headers $headers
$LicensedGroups = if ($groupIrmResult) { $groupIrmResult.where({$PSitem.assignedLicenses}) }

#endregion

#region Get All Users
$filter = "startsWith(employeeId,'emp0'"
$Uri = 'https://graph.microsoft.com/v1.0/users?$filter={0})&$select=displayName,employeeID,licenseAssignmentStates' -f $filter
$AllEmployees = (Invoke-RecurenceRestMethod -Uri $uri -Headers $headers)
#endregion

#region import SKU Friendly Names
$UriGist = 'http://bit.ly/SKUFriendlyNames'
$SKUFriendlyNames = Invoke-RestMethod -Uri $UriGist -UseBasicParsing | Convertfrom-Csv -Delimiter ';' 
#endregion

#region Get each user license assignment (and if through a group and group name)
$output = foreach ($row in $AllEmployees.where({$PSitem.licenseAssignmentStates})) {
    [pscustomobject]@{
       UserName = $row.displayName
       EmployeeID = $row.EmployeeID
       Licenses = foreach ($license in $row.licenseAssignmentStates) {
           [pscustomobject]@{
               LicenseSKUID = $license.skuID
               LicensePartNumber = ($SKUs.where({$PSItem.skuid -eq $license.skuid})).skuPartNumber
               ProductName = $SKUFriendlyNames.where({$PSitem.LicenseSKUID -eq $license.skuid }).ProductName
               assignedbyGroup = if($license.assignedbyGroup) { $true } else {$false}
               assignedbyGroupName = ($LicensedGroups.where({$PSitem.id -eq $license.assignedbyGroup})).displayName
           }
   
       }
   
    }
}
#endregion


$groupName = 'EMSPREMIUM'
$Details = $output.where({$PSItem.Licenses.LicensePartNumber -ieq $groupName}) 
$Details

$UserName = 'Mateusz Czerniawski'
$Details.where({$PSitem.UserName -eq $UserName}).Licenses | Format-Table

$FriendlyGroupName = 'ENTERPRISE MOBILITY + SECURITY E5'
$Details = $output.where({$PSItem.Licenses.ProductName -ieq $FriendlyGroupName}) 
$UserName = 'Mateusz Czerniawski'
$Details.where({$PSitem.UserName -eq $UserName}).Licenses | Format-Table
```

## Summary

This does the job for now. I'll probably clean it up a bit, add some error handling and more wrappers for slicing the data. Soon :grin: 

Hope this can be usefull to anyone out there! I definetely have a few more grey hairs cause of all these... dark *documentation* paths I had to travel :grin: 

It appears that after two years nothing has changed in the cmdlet/Graph options - one still has to build a custom solution for these kind of tasks!
