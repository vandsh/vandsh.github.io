---
layout: post
title: "SPE: Batch Update Rendering Controllers"
category: Sitecore
tags: Sitecore Powershell SPE
---

Recently I came across a site that needed some Project and Solution level refactoring.  Having needed to do this in the past, I was dreading having to go thru the `Controller Renderings` and having to update the values for the `Controller` and/or `Action` to match the new namespace.  Thankfully, using a very similar setup as I have in the past, I was able to pull this off easily using [Sitecore Powershell Extensions](https://marketplace.sitecore.net/en/Modules/Sitecore_PowerShell_console.aspx).

### Batch Update Rendering Controllers
```powershell
$rootItem = "master:/sitecore/layout/Renderings/YourSite"
$controllerHash = @{
    "YourSite.Controllers.Tenant.ModulesController, YourSite" = "YourSite.Web.Tenant.Controllers.Tenant.ModulesController, YourSite.Web.Tenant"
    "YourSite.Controllers.Shared.ModulesController, YourSite" = "YourSite.Web.Shared.Controllers.Shared.ModulesController, YourSite.Web.Shared"
    "YourSite.Controllers.Shared.NavigationController, YourSite" = "YourSite.Web.Shared.Controllers.Shared.NavigationController, YourSite.Web.Shared"
    "YourSite.Controllers.Shared.PageFeaturesController, YourSite" = "YourSite.Web.Shared.Controllers.Shared.PageFeaturesController, YourSite.Web.Shared"
    "YourSite.Controllers.Search.SearchPageController, YourSite" = "YourSite.Web.Shared.Controllers.Search.SearchPageController, YourSite.Web.Shared"
}
$count = 0
Get-ChildItem $rootItem -Recurse | ForEach-Object {
    $currentItem = $_
    Get-ItemField -Item $currentItem -ReturnType Field -Name "Controller" `
    | ForEach-Object{ 
        $_.Value
        $newControllerVal = $controllerHash[$_.Value]
        if(!!$newControllerVal){
            #output itemid and field name
            Write-Host ("[" + $currentItem.ID + ": " + $currentItem.Name + "::" + $_.Name + "]")
            $currentItem.Editing.BeginEdit()
            $currentItem[$_.Name] = $newControllerVal
            $currentItem.Editing.EndEdit()
            $count++
        }
    }
}

Write-Host "Updated " $count " Found Values"
```

The above script will return something pretty boring like: 

```
.....
YourSite.Controllers.Renderings.YourSite.Shared.ModulesController, YourSite
[{B7E9BA6C-9961-4D99-922C-C11C1D896F26}: Locations::Controller]
True
YourSite.Controllers.Renderings.YourSite.Shared.NavigationController, YourSite
[{57804F35-BAF7-4D91-BA36-4EBC7ACFE0D6}: Navigation::Controller]
True
YourSite.Controllers.Renderings.YourSite.Shared.ModulesController, YourSite
[{89F94535-26CD-483D-9D5E-1D427773B914}: 301 - List with Filter::Controller]
True
YourSite.Controllers.Renderings.YourSite.Shared.ModulesController, YourSite
[{EB7DD5D5-EC44-4A06-8124-5DABE66BDC98}: 302 - List with no Filter::Controller]
True
Updated  37  Found Values
```

I would suggest maybe running the actual script with the editing commented out just to make sure you have proper matches (test it out before you _actually_ update). 
```powershell
    #$currentItem.Editing.BeginEdit()
    #$currentItem[$_.Name] = $newControllerVal
    #$currentItem.Editing.EndEdit()
```
This all goes without saying, the same code can be used to update `Actions`, `Layout` paths and *cough* Webform *cough* `Sublayout` Ascx file paths simply by swapping out the `Controller` name in the `ReturnType Field`.

I know this is nothing earth shattering, but again SPE makes the job a little easier. 

Hope this helps someone.
