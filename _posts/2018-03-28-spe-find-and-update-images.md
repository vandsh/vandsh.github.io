---
layout: post
title: "SPE: Find and Update Images in Rich Text fields"
category: Sitecore
tags: Sitecore Powershell SPE
---

I and another developer were tasked with the job of importing content from another CMS. We soon came to realize that the fields that were coming over (primarily the rich text fields) were littered with image tags with relative sources `<img src=/path/to/file.jpg />`.  We were using [Data Exchange Framework](https://dev.sitecore.net/Downloads/Data_Exchange_Framework.aspx) to do the bulk of the import but something like updating the image tags seemed like such an edge case we decided to leverage [Sitecore Powershell Extensions](https://marketplace.sitecore.net/en/Modules/Sitecore_PowerShell_console.aspx) for such a daunting task:

### Find and Update Images in Rich Text Fields
```powershell
$result = Read-Variable -Parameters `
    @{ Name = "contentRoot"; Title="Imported Content Root"; Source="Datasource=/sitecore/content/"; editor="droptree"}, `
    @{ Name = "mediaLibraryRoot"; Title="Media Library Root"; Source="Datasource=/sitecore/media library"; editor="droptree"}, `
    @{ Name = "templateToParseItem"; Title="Template to Parse"; Source="Datasource=/sitecore/templates"; editor="droptree"}, `
    @{ Name = "fieldToParse"; Title="Field To Parse"  } `
    -Description "Pick your roots" `
    -Title "Update images in imported RTE" -Width 500 -Height 480 -OkButtonName "Proceed" -CancelButtonName "Abort"
if($result -ne "ok")
{
    Exit
}
## Global Vars
$templateToParse = $templateToParseItem.ID.ToString()

#Get the media link prefix from settings
$mediaLinkPrefix = [Sitecore.Configuration.Settings]::GetSetting("Media.MediaLinkPrefix")
$defaultLinkPrefix = [Sitecore.Configuration.Settings]::GetSetting("Media.DefaultMediaPrefix")
$fallthru = if([string]::IsNullOrEmpty($defaultLinkPrefix)){"-/media/"} else {$defaultLinkPrefix}
$finalLinkPrefix = if([string]::IsNullOrEmpty($mediaLinkPrefix)){$fallthru} else {$mediaLinkPrefix}

# Get all items in the tree given a particular template type
$itemsToParseHtml = Get-ChildItem -Item $contentRoot | Where-Object {$_.TemplateID -eq $templateToParse}

$itemsToParseHtml | %{
    $currentItem = $_
    $fieldValueWithHtml = $currentItem[$fieldToParse] 
    if($fieldValueWithHtml -ne $null){
        $needToUpdateHtml = $false;
        $source = [System.Web.HttpUtility]::HtmlDecode($fieldValueWithHtml)
        $html = New-Object -ComObject "HTMLFile";
        $html.IHTMLDocument2_write($source); # Reading in to a proper HTML document
        $html.Images | % {
	    # You can put a Write-Host $_.Src here to see the actual src value
            $trimmedSource = $_.Src.Replace(".jpg","").Replace(".png","").Split('?')[0]
	    $relativeMediaItemPath = ("{0}{1}" -f $mediaLibraryRoot.Path, $trimmedSource)
            $mediaItemExists = Test-Path -Path $relativeMediaItemPath #test first to see if its there...
            if($mediaItemExists){
                $matchingLibraryItem = Get-Item -Path $relativeMediaItemPath
                if($matchingLibraryItem -ne $null)
                {
		    # Create new link, should be recognized by Rich Text Editor
                    $linkSource = ("{0}{1}.ashx" -f $finalLinkPrefix, $matchingLibraryItem.ID.ToString().Replace("-","").Replace("{","").Replace("}","") )
                    write-host ("found match, swapping {0} for {1}" -f $trimmedSource, $linkSource)
                    # Update the original <img> tag with new vals recognized by RTE
                    $_.Src = $linkSource
                    $_.Title = $matchingLibraryItem.Name
                    $_.LongDesc = $matchingLibraryItem["Title"]
                    $_.Alt = $matchingLibraryItem["Alt"]
                    $_.Width = $matchingLibraryItem["Width"]
                    $_.Height = $matchingLibraryItem["Height"]
                }
                $needToUpdateHtml = $true
            }
            else{
                write-host "no matched items for " $currentItem.ID
            }
        }
        
        # If a media library item found, we need to update the original item rich text
        if($needToUpdateHtml){
            $updatedHtml = ($html.Body | % InnerHtml)
            $currentItem.Editing.BeginEdit()
            $currentItem[$fieldToParse] = $updatedHtml
            $currentItem.Editing.EndEdit()
        }
    }
}

```

The above script is essentially iterating over a set of items and parsing a field you choose that has HTML in it for a path that matches a relative path in the media library.  This allows you to zip up your media folder from whatever other CMS or site you might be porting from and uploading it as a zip into a single location in the media library.  Once its there, run the script to associate the Media Library items and once that association is made, you can reorganize the library to your liking.  

The only other thing I would like to mention, since this is utilizing the `HtmlDocument` in PowerShell, you should also be able to leverage this script to parse out other things like links to external sites, internal pages, or strip off undesired classes/tags.

Slowly but surely SPE will take over the world. 

Hope this helps someone.
