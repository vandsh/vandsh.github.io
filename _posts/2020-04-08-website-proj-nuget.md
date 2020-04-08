---
layout: post
title: "NuGet for Website Projects"
category: dotnet
tags: dotnet NuGet CI-CD Powershell
---

### The Problem
Unlike proper `Web Application` projects, `Website` projects in Visual Studio just take everything under a folder and assume that is the project. And 9 times out of 10 (based on the handful of times I have dealt with them) these projects, when in source control, have _all_ binaries checked in as well. 

References can be sorta made using `NuGet` on the `Bin` folder, but in my experience this only sort-of works: `.refresh` files are added to the folder as a kind of redirect to the original file located elsewhere (like the `../packages` directory. These refreshes are triggered at certain times like when the project is opened or switching between projects. It also does not care about any folders nested underneath like the `Bin/roslyn` folder.

That kinda stinks.

### The Solution
A single Powershell file to help facilitate the restore and frankly, copying of dlls locally so we can work without having to shove binary files in source.
In addition to the script, I also included a custom `nuget.config` to allow for other `NuGet` feeds (like `Telerik`).

Folder Structure:
 - Project/ (root)
   - .nuget/nuget.config
   - lib/
   - packages/
   - src/
   - restore.ps1

## .nuget/nuget.config

Make sure we have all the dependencies we need, in our case we needed `Telerik` as well, which happens to be authenticated.

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="Telerik" value="https://nuget.telerik.com/nuget" />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
  </packageSources>
  <activePackageSource>
    <add key="All" value="(Aggregate source)" />
  </activePackageSource>
  <packageSourceCredentials>
    <Telerik>
      <add key="Username" value="bob@pizza.co" />
      <add key="ClearTextPassword" value="NoToppings!" />
    </Telerik>
  </packageSourceCredentials>
</configuration>
```

## restore.ps1

Handle what the package manager in a website project can't:

```powershell
set-location $PSScriptRoot
Write-Host "Executing NuGet self update and restore" -ForegroundColor Green
nuget update -self
nuget restore .\src\packages.config  -PackagesDirectory packages

set-location $PSScriptRoot\src
Write-Host "Copying files for .refresh" -ForegroundColor Green
Get-ChildItem -Path ./ -Filter *.refresh -Recurse -File -Name | ForEach-Object {
    $referenceFile = Get-Content $_
    $destinationFile = $_.Replace(".refresh", "")
    Write-Host $referenceFile " -> "   $destinationFile
    Copy-Item $referenceFile -Destination $destinationFile
}

```

Executing this script does the following: 

 - `NuGet` self update
 - `NuGet` restore using the `packages.config` maintained by the project and puts them one layer up in the `../packages` directory
 - Iterates thru all of the `.refresh` files in the project and copies over the original `*.dll` file to the location of the `.refresh` file.

This helped a lot for local development, particularly for the initial environment setups.  Queue the next step...

## CI/CD

Along with this effort, we also helped set up `CI/CD` within `Azure Devops`. We used a very similar principle in resolving the dependencies during the Build pipeline:

 - Use NuGet 5.5.0 task
 - NuGet restore task
   - Path to NuGet.config `$/Project/.nuget/nuget.config`
   - Destination Directory `$(UserProfile)/.nuget/packages`
 - PowerShell Script task (inline) 
   - ```powershell
     Write-Host $PSScriptRoot
Write-Host "Copying files for .refresh" -ForegroundColor Green
Get-ChildItem -Path ./ -Filter *.refresh -Recurse -File -Name | ForEach-Object {
    $referenceFile = (Get-Content $_).Replace("..\packages","$(UserProfile)\.nuget\packages")
    $destinationFile = $_.Replace(".refresh", "")
    Write-Host $referenceFile " -> "   $destinationFile
    Copy-Item $referenceFile -Destination $destinationFile
}

      ```
 - Build Solution task
  - Solution: `$/Project/src/website.publishproj`
  - MSBuild Arguments 
    - `/p:deployOnBuild=true` 
    - `/p:publishProfile=Project` 
    - `/p:publishUrl="$(build.artifactstagingdirectory)\\"`
 - Publish Artifacts task

The result of the above pipeline and really the entire effort is a solution that doesn't involve checking in binaries, uses NuGet and overall feels just slightly closer to normal Web Application development.

Hope this helps!

