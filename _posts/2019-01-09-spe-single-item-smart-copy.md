---
layout: post
title: "SPE: Single item Smart Copy"
category: Sitecore
tags: Sitecore Powershell SPE
---

Let's start off 2019 with a ~~great~~ ~~good~~ ~~mediocre~~ average-at-best post!

Recently I was pinged by [John Rappel](https://twitter.com/jraps20) about [Sitecore Powershell Extensions](https://marketplace.sitecore.net/en/Modules/Sitecore_PowerShell_console.aspx) and if there was any "Smart Copy" scripts or modules out on the interwebs. Unfortunately (or fortunately for myself) there wasn't any that I knew of or could find so I started hammering it out in SPE. 
The need seemed pretty simple:
 - Copy the main item from a source item to a new item somewhere else in the tree
 - Copy the items that are referenced _from_ the main item to new items relative to the main copied item
 - Update the links/references from the new copied item to the newly copied relative items

Nail, meet sledgehammer:

### Single Item Smart Copy
```powershell
$copyOptions = New-Object System.Collections.Specialized.OrderedDictionary
$copyOptions.Add("Copy only linked children", 0 )
$copyOptions.Add("Copy all children", 1)


$dialogProps = @{
    Parameters = @(
        @{ Name = "sourceItem"; Title="Item To Copy"; Source="Datasource=/sitecore/content/"; editor="droptree"}
        @{ Name = "destinationPathParent"; Title="Place to copy to"; Source="Datasource=/sitecore/content/"; editor="droptree"}
        @{ Name = "copyChildrenOption"; Value="0"; Title="Copy Children Type"; Tooltip="Which children do you want to copy?"; editor="radio"; options=$copyOptions}
    )
    Description = "Copy item and the relative datasources to a new location" 
    Title = "Smart Copy" 
    Width = 500 
    Height = 280 
    OkButtonName = "Proceed" 
    CancelButtonName = "Abort"
    Icon = "office/32x32/data_copy.png"
}
$result = Read-Variable @dialogProps
if($result -ne "ok")
{
    Exit
}

$destinationPath = $destinationpathParent.FullPath
$sourceItemName = $sourceItem.Name
$sourceItemPath = $sourceItem.FullPath

if((Test-Path "$destinationPath/$sourceItemName") -eq $False)
{
    Write-Host "Creating main item " "$destinationPath/$sourceItemName"
    copy-item -Path $sourceItem.FullPath -Destination "$destinationPath/$sourceItemName"
}
else{
    Write-Host "Main item already exists " "$destinationPath/$sourceItemName"
}

$destinationItem = Get-Item -Path "$destinationPath/$sourceItemName"
Write-Host "Copied item " $destinationItem.FullPath

$allSourceChildrenItems = $sourceItem.Axes.GetDescendants() | Initialize-Item
$destinationItemRootPath = $destinationItem.FullPath
$allSourceItemLinks = $sourceItem | Get-ItemReference -ItemLink #get all links, regardless of relative

$datasourceFieldOptions = New-Object System.Collections.Specialized.OrderedDictionary
foreach($itemlink in $allSourceItemLinks){
    
    if([Sitecore.Data.ID]::IsNullOrEmpty($itemLink.SourceFieldID) -eq $False){
        $itemField = $sourceItem.Fields[$itemLink.SourceFieldID]
        $field = [Sitecore.Data.Fields.FieldTypeManager]::GetField($itemField)
        $datasourceFieldOptions[$itemField.Name] = $itemLink.SourceFieldID
        Write-Host $itemField.Name  $itemLink.SourceFieldID
    }
}

$result = Read-Variable -Parameters `
    @{ Name = "datasourceFieldOption"; Value="0"; Title="Copied Link types"; Tooltip="What linked fields do you care to copy?"; editor="checklist"; options=$datasourceFieldOptions} `
    -Description "Copy item and the relative datasources to a new location" -Title "Smart Copy" -Width 500 -Height 280 -OkButtonName "Proceed" -CancelButtonName "Abort"
if($result -ne "ok")
{
    Exit
}

$selectedLinkTypeDatasources = @($allSourceItemLinks | Where-Object {($datasourceFieldOption.Contains($_.SourceFieldID.ToString()) -eq $True)})
$sourceRelativeDatasourceReferences = $selectedLinkTypeDatasources | ForEach-Object{Get-Item -Path "master:/" -ID $_.TargetItemID} | Where-Object {$_.FullPath.StartsWith($sourceItemPath)}

if($copyChildrenOption -eq 0){
    foreach($sourceRelativeDatasourceItem in $sourceRelativeDatasourceReferences)
    {
        $currentRelativeDatasourceItemPath = $sourceRelativeDatasourceItem.FullPath.Replace($sourceItemPath, $destinationItemRootPath)
        
        #recreate tree leading to datasource item if it doesn't exist
        foreach($sourceChildItem in $allSourceChildrenItems)
        {
            $relativeChildItem = $sourceChildItem.FullPath.Replace($sourceItemPath, $destinationItemRootPath)
            if($currentRelativeDatasourceItemPath.StartsWith($relativeChildItem))
            {
                if((Test-Path $relativeChildItem) -eq $False)
                {
                    Write-Host "Creating needed " $relativeChildItem
                    copy-item -path $sourceChildItem.FullPath -destination $relativeChildItem
                }
            }
        }
    }
}

#copy all children
if($copyChildrenOption -eq 1){
    foreach($sourceChildItem in $allSourceChildrenItems)
    {
        $destinationChildItemPath = $sourceChildItem.FullPath.Replace($sourceItemPath, $destinationItemRootPath)
        if((Test-Path $destinationChildItemPath) -eq $False)
        {
            Write-Host "Creating child " $destinationChildItemPath
            copy-item -path $sourceChildItem.FullPath -destination $destinationChildItemPath
        }
        else{
            Write-Host "Child already exists " $destinationChildItemPath
        }
    }
}

#update links
$sourceRelativeDatasourceReferenceIDs = @($sourceRelativeDatasourceReferences |  Select-Object -ExpandProperty ID)
$destinationRelativeDatasourceReferences = $destinationItem | Get-ItemReference -ItemLink | Where-Object {$sourceRelativeDatasourceReferenceIDs.Contains($_.TargetItemID)}

#recreate links
foreach($destinationRelativeDatasourceReference in $destinationRelativeDatasourceReferences){
    #now that we found matching old ones, let's update to new items
    $currentOldDatasourceReference = Get-Item -Path "master:/" -ID $destinationRelativeDatasourceReference.TargetItemID
    $currentNewDatasourceReference = Get-Item -Path $currentOldDatasourceReference.FullPath.Replace($sourceItemPath, $destinationItemRootPath)
    Write-Host "Updating old link to "  $currentOldDatasourceReference.FullPath " to " $currentNewDatasourceReference.FullPath
    Update-ItemReferrer -Link $destinationRelativeDatasourceReference -NewTarget $currentNewDatasourceReference
}

Write-Host "Done I think?" -ForegroundColor Green
```

So the wall of code above performs the necessary lookups given a source and destination, creates new items, and updates the references to the newly copied items!

A few quick screenshots to maybe spark your interest and give some context:

![alt text](/assets/singleItemSmartCopy_1.png "Single Item Smart Copy Source and Target Prompt")

The above screenshot should show you a source item, target location, and a radio button allowing you to select if you want to copy just the linked children, or _all_ children under the source item.

![alt text](/assets/singleItemSmartCopy_2.png "Single Item Smart Copy Link Type Selection Prompt")

Given that you may want to be selective _which_ references you really care to copy over, there is a second prompt allowing you to pick which link types you care about. This will help you in a case where maybe you want to copy all rendering datasources but really don't care about links in a rich text field.

And thats about it. Maybe one day when I get a chance, I'll get this into a pull request to be pulled into SPE but for now, enjoy the mess above!

`</end of post>`
