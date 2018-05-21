---
layout: post
title: "SPE: Upload and Import Users"
category: Sitecore
tags: Sitecore Powershell SPE
---

Recently on a project doing a content migration, we were tasked with importing existing users from an existing system (dumped into an xml document). So, of course, I dusted off [Sitecore Powershell Extensions](https://marketplace.sitecore.net/en/Modules/Sitecore_PowerShell_console.aspx) for such an occassion:
### User Export XML
```xml
<users type="users" count="22982">
	<user>
		<id>8675309</id>
		<username>first.last@yourwebmail.com</username>
		<display-name>first.last@yourwebmail.com</display-name>
		<first-name>First</first-name>
		<last-name>Last</last-name>
		<email>first.last@yourwebmail.com</email>
		<address></address>
		<accepted-terms>False</accepted-terms>
		<last-login-date></last-login-date>
		<groups>
			<group>Custom_Group</group>
		</groups>
	</user>
        <!-- countless other users below... -->
```
The below script is essentially iterating over the above xml file and creating new users (with a `Custom Profile` to store the properties nicely). 

### Upload and import users
```powershell
$inputFile = Receive-File -Path "C:\temp\upload"
[xml]$userData = Get-Content $inputFile
[Reflection.Assembly]::LoadWithPartialName("System.Web")
$userData.users.user[0] | % {
    $firstName = ($_.'first-name').Trim('"')
    $lastName = ($_.'last-name').Trim('"')
    $fullName = [string]::Format("{0} {1}",$firstName,$lastName)
    $identity = [string]::Format("extranet\{0}",$_.username)
    $password = [System.Web.Security.Membership]::GeneratePassword(15,2)
    $groups = $_.groups | %{$allGroups = ""}{$allGroups += ($_.group + ", ")}{$allGroups}
	#Custom Profile
    $profileItemId = "{42B75431-C3CE-4083-A2E1-775D2081C274}"
    $user = New-User -Identity $identity -Enabled -Password $password -Email $_.email -FullName $fullName -ProfileItemId $profileItemId
    $user.Profile.SetCustomProperty("Old System ID", $_.id)
    $user.Profile.SetCustomProperty("First Name", $firstName)
    $user.Profile.SetCustomProperty("Last Name", $lastName)
    $user.Profile.SetCustomProperty("Address", $_.addresss)
    $user.Profile.SetCustomProperty("Last Login Date", $_.'last-login-date')
    $user.Profile.SetCustomProperty("Groups", $groups)
    $user.Profile.SetCustomProperty("Accepted Terms", $_.'accepted-terms')
    write-host $identity $password
} 
```
As you probably notice, I am running `GeneratePassword` to create the user with a random password on import with the intention that on first login, imported users will be forced to reset/change password.  Anyways, hopefully this small but effective script can save someone a few hairs on their head.

Slowly but surely SPE will take over the world. 

Hope this helps someone.
