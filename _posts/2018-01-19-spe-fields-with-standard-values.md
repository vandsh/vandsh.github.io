---
layout: post
title: "SPE: Get fields using Standard Values"
category: Sitecore
tags: Sitecore Powershell SPE
---

Every once and a while, I end up checking what fields on items have defined standard values.  I recently wrote the script below using [Sitecore Powershell Extensions](https://marketplace.sitecore.net/en/Modules/Sitecore_PowerShell_console.aspx).

### Get Fields using Standard Values
```powershell
$rootItem = "master:/sitecore/content/Home"
$hashTable = @{} 
Get-ChildItem $rootItem -Recurse | ForEach-Object {
    $currentItem = $_
    $inheritedFields = "";
    Get-ItemField -Item $currentItem -ReturnType Field -Name "*" `
    | Where-Object {$_.ContainsStandardValue} | Select-Object -Property Name,Value `
    | ForEach-Object{ $inheritedFields += ("[" + $_.Name + ": " + $_.Value + "]")}
    if($inheritedFields -ne "")
    {
        $hashTable.Add($currentItem.ContentPath, $inheritedFields)
    }
}

$hashTable | Format-Table -AutoSize
```

The above script will return something pretty basic like: 

```
Name                                                         Value
----                                                         -----
/Path/To/Item                     [Field 1 Name: Field 1 value][Field 2 Name: Field 2 value]
/Home/Words/Pizza/                [Hero Image: <image mediaid="{B309F112-E84A-433C-8E3F-33D3454B0D10}" />]
/Home/Words/Pizza/Pepperoni       [Hero Image: <image mediaid="{B309F112-E84A-433C-8E3F-33D3454B0D10}" />]
/Home/Words/Tacos                 [Option: {4F2F1CAE-F947-41F5-9A4A-C25DBE247F7A}][How Much Meat: 3]
/Home/Words/Burrito               [Number of Returned Options: 3]
```

My guess is it should probably turn into a report, but for now I will run it occasionally just as a sanity check.

`</end of post>`
