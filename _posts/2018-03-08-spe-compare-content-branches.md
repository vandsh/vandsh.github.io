---
layout: post
title: "SPE: Compare Content Branches"
category: Sitecore
tags: Sitecore Powershell SPE
---

Came across a [question](https://sitecore.stackexchange.com/questions/10663/any-way-to-compare-two-branches-of-a-sitecore-content-tree/10666) recently on Sitecore StackExchange by [Matthew Dresser](https://twitter.com/m_dresser) regarding the need to quickly compare to content branches in Sitecore.  There are a handful of ways this could be accomplished (serializing/packaging and using visual diff tools to compare etc).  [Sitecore Powershell Extensions](https://marketplace.sitecore.net/en/Modules/Sitecore_PowerShell_console.aspx) is something I lean on probably too often, but I thought what the heck, maybe it's something I could use in the future for the same exact reason.  I started this venture guided by a [post](https://gist.github.com/michaellwest/babe14646ba4f261004e394ce93c9b5b) from [Michael West](https://twitter.com/MichaelWest101).  The majority of this blog post will be a regurgitation of whats already in the SSE post, with some additional commentary:

### Compare Content Branches
```powershell
# Site 1 content path
$site1 = "/sitecore/content/Sites/Site1"
# Site 2 content path
$site2 = "/sitecore/content/Sites/Site2"
# Get site relative paths and IDs
$site1Desc = Get-ChildItem  -Path "master:$site1" -Recurse
# Flat lookup for quick ID grabbing
$site1Lookup = $site1Desc | ForEach-Object { $lookup = @{} } { $lookup[$_.FullPath.Replace($site1, "")] = $_.ID } { $lookup }
# Array of objects to compare directly, would normally use the list of descendands, but we needed to trim path to be relative.
$site1LookupRel = $site1Desc | ForEach-Object { $objArr = @() } { 
       $newObj = (New-Object -TypeName PSObject | Add-Member NoteProperty Name $_.Name -PassThru | Add-Member NoteProperty ID $_.ID -PassThru | Add-Member NoteProperty Path $_.FullPath.Replace($site1, "") -PassThru)
       $objArr += $newObj
} { $objArr }

#same stuff, different Site
$site2Desc = Get-ChildItem  -Path "master:$site2" -Recurse
$site2Lookup = $site2Desc | ForEach-Object { $lookup = @{} } { $lookup[$_.FullPath.Replace($site2, "")] = $_.ID } { $lookup }
$site2LookupRel = $site2Desc | ForEach-Object { $objArr = @() } { 
        $newObj = (New-Object -TypeName PSObject | Add-Member NoteProperty Name $_.Name -PassThru | Add-Member NoteProperty ID $_.ID -PassThru | Add-Member NoteProperty Path $_.FullPath.Replace($site2, "") -PassThru)
        $objArr += $newObj
    } { $objArr }
      
# Compare lists of relative paths/items
$itemComparison = Compare-Object -ReferenceObject $site1LookupRel -DifferenceObject $site2LookupRel -Property Path -IncludeEqual
# Use SideIndicator to grab those that match
$matchingItems = $itemComparison | Where-Object { $_.SideIndicator -eq "==" } |Select-Object -Expand Path
# Use SideIndicator to grab those that don't match
$nonMatchingItems = $itemComparison | Where-Object { $_.SideIndicator -ne "==" } |Select-Object -Expand Path
Write-Host " *** Matching Items *** "
foreach($matchingItem in $matchingItems) {
    $site1MatchingItem = $null;
    $site2MatchingItem = $null;
    # Quick check to make sure the do exist in both
    if($site1Lookup.Contains($matchingItem))
    {
        $site1MatchingItem = Get-Item -Path "master:" -ID $site1Lookup[$matchingItem]
    }
        
    if($site2Lookup.Contains($matchingItem)){
        $site2MatchingItem = Get-Item -Path "master:" -ID $site2Lookup[$matchingItem]
    }
    
    # If both exist, get all fields that are not using a Standard Value
    if($site1MatchingItem -ne $null -AND $site2MatchingItem -ne $null) {
       $site1ItemFieldValues = Get-ItemField -Item $site1MatchingItem -ReturnType Field -Name "*" | Where-Object {!$_.ContainsStandardValue}  | Select-Object -Property @{Name="SourceID"; Expression={$site1MatchingItem.ID}},Name,Value 
       $site2ItemFieldValues = Get-ItemField -Item $site2MatchingItem -ReturnType Field -Name "*"  | Where-Object {!$_.ContainsStandardValue} | Select-Object -Property @{Name="SourceID"; Expression={$site2MatchingItem.ID}},Name,Value 
       # If there are fields to compare, let's compare!
       if($site1ItemFieldValues -ne $null -AND $site2ItemFieldValues -ne $null){
           Write-Host "$matchingItem does exist in both sites, comparing now - (" $site1MatchingItem.ContentPath " <=>" $site2MatchingItem.ContentPath ")" -ForegroundColor Green
	   $comparedItems = Compare-Object -ReferenceObject $site1ItemFieldValues -DifferenceObject $site2ItemFieldValues -Property Name,Value -IncludeEqual -PassThru
	   $comparedItems | Format-Table -AutoSize
       }
       # There might not be any fields to compare (ie a _content folder that has no custom fields)
       else{
           Write-Host "$matchingItem does exist in both sites, but there is nothing to compare - (" $site1MatchingItem.ContentPath " <=>" $site2MatchingItem.ContentPath ")" -ForegroundColor Green
       }
    }
}
Write-Host " *** NonMatching Items *** "
foreach($nonMatchingItem in $nonMatchingItems) {
    $site1MatchingItem = $null;
    $site2MatchingItem = $null;
    # Ok, found a mismatch, let's see which Site has what
    if($site1Lookup.Contains($nonMatchingItem))
    {
        $site1MatchingItem = Get-Item -Path "master:" -ID $site1Lookup[$nonMatchingItem]
    }
    
    if($site2Lookup.Contains($nonMatchingItem)){
        $site2MatchingItem = Get-Item -Path "master:" -ID $site2Lookup[$nonMatchingItem]
    }
        
    if($site1MatchingItem -ne $null -AND $site2MatchingItem -eq $null){
        Write-Host "$nonMatchingItem exists in Site1 but not Site2 - (" $site1MatchingItem.ContentPath ")" -ForegroundColor Yellow
    }
    elseif($site1MatchingItem -eq $null -AND $site2MatchingItem -ne $null){
        Write-Host "$nonMatchingItem exists in Site2 but not Site1 - (" $site2MatchingItem.ContentPath ")" -ForegroundColor Yellow
    }
}
```

The above script will return color coded output like so: 

```
 *** Matching Items ***
.....
/_content does exist in both sites, but there is nothing to compare - ( /Site1/_content  <=> /Site2/_content )
/_content/Country Landing Intro does exist in both sites, comparing now - ( /Site1/_content/Intro  <=> /Site2/_content/Intro )

SourceID                               Name         Value  SideIndicator
--------                               -----        -----  -------------
{96892835-46BF-437A-8C9E-40B12EC58881} Image Is Map false  == 
{96892835-46BF-437A-8C9E-40B12EC58881} Is Header    true   =>
{98D165DB-25EA-4E80-B9A3-ABCC181A9FAF} Is Header    false  <=

 
/Cities does exist in both sites, but there is nothing to compare - ( /Site1/Cities  <=> /Site2/Cities )
 *** NonMatching Items ***
/Cities/Second-Test exists in Site1 but not Site2 - ( /Site1/Cities/Second-Test )
/Cities/Second-Test/_content exists in Site1 but not Site2 - ( /Site1/Cities/Second-Test/_content )
```

There are probably more efficient ways to execute this, but this seemed pretty straightforward and easy to read code-wise, hopefully easy enough to disect and augment.  Places I could see augmenting it would be swapping out what the branches match on (instead of path, maybe name or template type).  You can also limit the fields it compares on, above it is pulling all fields that do not use a `Standard Value`, you may only be concerned about a few particular fields.  Another small thing I could see doing is, on the item specific `Compare-Object`, filter the output by the `SideIndictor` to only show `Where-Object { $_.SideIndicator -ne "==" }` (only show the differing),

Slowly but surely SPE will take over the world. 

Hope this helps someone.
