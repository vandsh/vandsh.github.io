---
layout: post
title: "SPE: Search and Replace in Field Values"
category: Sitecore
tags: Sitecore Powershell SPE
---

Below is an example script to find and replace all instances in all fields below the root item using [Sitecore Powershell Extensions](https://marketplace.sitecore.net/en/Modules/Sitecore_PowerShell_console.aspx).

### Get Fields using Standard Values
```powershell
$rootItem = "master:/sitecore/content/Home"
$needle = "vegetables"
$newNeedle = "pizza"
$count = 0
Get-ChildItem $rootItem -Recurse | ForEach-Object {
    $currentItem = $_
    Get-ItemField -Item $currentItem -ReturnType Field -Name "*" `
    | ForEach-Object{ 
        if($_.Value.Contains($needle)){
            #output itemid and field name
            Write-Host ("[" + $currentItem.ID + ": " + $_.Name + "]")
            $newHaystack = $_.Value.Replace($needle, $newNeedle);
            $currentItem.Editing.BeginEdit()
            $currentItem[$_.Name] = $newHaystack
            $currentItem.Editing.EndEdit()
            $count++
        }
    }
}

Write-Host "Updated " $count " Found Values"
```

The above script will return something pretty simple like: 

```
[{A351623A-003E-486B-A35B-EDD69E335451}: Description]
True
[{A351623A-003E-486B-A35B-EDD69E335451}: Page Title]
True
[{586C7A62-FE78-4C00-9816-0DF8E3FE5849}: Html]
True
[{5A9BC349-5BDB-4D14-8FB4-D4F6EBB593B1}: Html]
True
Updated  137  Found Values
```

There may be many other ways to do this same task. Heck, probably several of them more efficient than this (and maybe less agressive as well) but I thought I'd take a look-see how it could be done.  There ya go.

`</end of post>`
