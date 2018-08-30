---
layout: post
title: "SPE: Rendering Definition Search and Replace"
category: Sitecore
tags: Sitecore Powershell SPE
---

A while back I had to go thru and resolve an issue where content editors where hand-keying (for whatever reason) the wrong name for a `placeholder` key.  Rather than going thru manually and updating all these incorrect `placeholder` keys, I decided to leverage [Sitecore Powershell Extensions](https://marketplace.sitecore.net/en/Modules/Sitecore_PowerShell_console.aspx) to do the heavy lifting for me:

### Rendering Definition Search and Replace
```powershell
#Get all properties from the `RenderingDefinition`
$searchOptions = New-Object System.Collections.Specialized.OrderedDictionary
(new-object Sitecore.Layouts.RenderingDefinition) | Get-Member | Where-Object {$_.MemberType -eq "Property"} | Foreach-Object {$searchOptions.Add($_.Name,$_.Name)}

#Let user select which version of the rendering they'd like to search
$layoutOptions = New-Object System.Collections.Specialized.OrderedDictionary
$layoutOptions.Add("Shared Layout", $false)
$layoutOptions.Add("Final Layout", $true )

$result = Read-Variable -Parameters `
    @{ Name = "haystackRoot"; Title="Haystack Root"; Source="Datasource=/sitecore/content/"; editor="droptree"}, `
    @{ Name = "searchOption"; Value="Placeholder"; Title="Search Field"; Tooltip="What Rendering field do you want to search"; editor="radio"; options=$searchOptions},  `
    @{ Name = "layoutOption"; Value="Final Layout"; Title="Layout Type"; Tooltip="What Layout do you want to run the search against"; editor="radio"; options=$layoutOptions},  `
    @{ Name = "needle"; Value="Needle"; Title="Needle"; Tooltip="The text you are looking for"; Placeholder="Needle"},  `
    @{ Name = "newNeedle"; Value="Replacement Needle"; Title="Replacement Needle"; Tooltip="The text you are replacing the needle with"; Placeholder="Replacement Needle"}  `
    -Description "Find and replace values in Renderings" -Title "Rendering Definition Search and Replace" -Width 500 -Height 480 -OkButtonName "Proceed" -CancelButtonName "Abort"
if($result -ne "ok")
{
    Exit
}

$count = 0;
Get-ChildItem -ID $haystackRoot.ID -Recurse | ForEach-Object { 
   $item = $_;
   Write-Host $item.Paths.FullPath
   
   #Find all renderings
   Get-Rendering -Item $item -FinalLayout:([System.Convert]::ToBoolean($layoutOption)) | Foreach-Object { 
	#Run Contains on the selected Rendering property/field
        if($_."$searchOption".Contains($needle)){
            $count = $count + 1
            $_."$searchOption" = $_."$searchOption".Replace($needle, $newNeedle); 
            Set-Rendering -Item $item -Instance $_  -FinalLayout:([System.Convert]::ToBoolean($layoutOption))
        }
    }
}

Write-Host "Count: " $count
```

The actual prompt that shows when the script runs will look like: 

![alt text](/assets/renderingSearchAndReplace.png "Rendering Search and Replace UI")

So at face value, you can obviously change any of the properties of a `RenderingDefinition` with this script.  My hope for this long term is to allow people to not fear refactoring front end layouts (`Placeholders`, `Datasources` etc). Being able to swap out a new `placeholder` key for a `Rendering` or a new `Datasource` to a restructured content tree, really just trying to make the daunting easier to accomplish.

Hope this helps!

`</end of post>`
