---
layout: post
title: "SPE: Search and Replace in Field Values (Version 2)"
category: Sitecore
tags: Sitecore Powershell SPE
---

In a [previous post](/sitecore/2018/01/27/spe-search-replace.html), I wrote an example script to find and replace all instances in all fields below the root item using [Sitecore Powershell Extensions](https://marketplace.sitecore.net/en/Modules/Sitecore_PowerShell_console.aspx).  The one in this post is just merely an extension on that leveraging a few things the other did not: `Read-Variables`, `Find-Item` (index based search), and also had it focus on a single field rather than _all_.  In my recent usage, it found and replaced roughly 9k items and updated them in under 10 minutes (all whilst writing output to the Powershell Console).

### Search and Replace (Version 2)
```powershell
$executionOptions = New-Object System.Collections.Specialized.OrderedDictionary
$executionOptions.Add("Search Only", 1)
$executionOptions.Add("Search and Replace", 2)

$searchOptions = New-Object System.Collections.Specialized.OrderedDictionary
$searchOptions.Add("Value Contains", "Contains")
$searchOptions.Add("Exact Match", "Equals")
$searchOptions.Add("Close Match", "Fuzzy")

$result = Read-Variable -Parameters `
    @{ Name = "haystackRoot"; Title="Haystack Root"; Source="Datasource=/sitecore/content/"; editor="droptree"}, `
    @{ Name = "fieldForSearch"; Value="Field for Search"; Title="Field for Search"; Tooltip="The field you are searching for"; Placeholder="Needle"}, `
    @{ Name = "needle"; Value="Needle"; Title="Needle"; Tooltip="The text you are looking for"; Placeholder="Needle"},  `
    @{ Name = "newNeedle"; Value="Replacement Needle"; Title="Replacement Needle"; Tooltip="The text you are replacing the needle with"; Placeholder="Replacement Needle"},  `
    @{ Name = "targetIndex"; Value="sitecore_master_index"; Title="Target Index"; Tooltip="The index to search"; Placeholder="sitecore_master_index"}, `
    @{ Name = "executeReplace"; Value="1"; Title="Execute Replace"; Tooltip="Radio button to say whether or not you actually want to replace yet"; editor="radio"; options=$executionOptions},  `
    @{ Name = "searchOption"; Value="Contains"; Title="Search Type"; Tooltip="What type of search do you want to run"; editor="radio"; options=$searchOptions}  `
    -Description "Search and replace vars:" -Title "Search and Replace" -Width 500 -Height 480 -OkButtonName "Proceed" -CancelButtonName "Abort"
if($result -ne "ok")
{
    Exit
}

$count = 0

$itemsToSearch = Find-Item -Index $targetIndex  -Criteria @{Filter = "DescendantOf"; Field = ("master:/" + $haystackRoot.Paths.Path) },  @{Filter = $searchOption; Field = $fieldForSearch; Value = $needle} | Initialize-Item 
$itemsToSearch | ForEach-Object {
    $currentItem = $_
	#output itemid and field name
	Write-Host ("[" + $currentItem.ID + " - " + $currentItem.Paths.Path + " :" + $currentItem.Language +": " + $_.Name + ": " + $_.Version + "] - " + $count)
	
	if($executeReplace -eq 2){
            $newHaystack = $currentItem[$fieldForSearch].Replace($needle, $newNeedle);
            $currentItem.Editing.BeginEdit()
            $currentItem[$fieldForSearch] = $newHaystack
            $currentItem.Editing.EndEdit()
	}
	$count++
}

Write-Host "Updated " $count " Found Values"
```

The actual prompt that shows when the script runs will look like: 

![alt text](/assets/searchandreplace.png "Search and Replace UI")


The above script will return something pretty simple like: 

```
[{A351623A-003E-486B-A35B-EDD69E335451} - /sitecore/content/Home/About Us :en-US: Module 1] - 0
True
[{A351623A-003E-486B-A35B-EDD69E335451} - /sitecore/content/Home/About Us :en-US: Module 2] - 1
True
[{586C7A62-FE78-4C00-9816-0DF8E3FE5849} -  /sitecore/content/Home/Content Page :en-US: Module 1] - 2
True
...
Updated  137  Found Values
```

Obviously there are ways that you can do this same thing [without Powershell right out of box](https://doc.sitecore.net/sitecore_experience_platform/content_authoring/searching/filters_and_operations/the_search_operations), but sometimes there are times where you would like just a little more control. And lets face it, who doesn't like Powershell (that's a rhetorical question).

`</end of post>`
