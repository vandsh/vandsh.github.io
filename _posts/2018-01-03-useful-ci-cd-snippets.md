---
layout: post
title: "Useful CI/CD Snippets"
category: Sitecore
tags: Sitecore CI-CD Powershell
---

Continuous Integration and Deployment is nothing new, and in the consulting space the amount of toolage we get to touch is pretty broad so we try to make things fairly universal.

### Backup Webroot Script
```powershell
$source = "$(SiteWebroot)"
$destination = "C:/agent/_work/_backups/Web"
New-Item -ItemType Directory -Force -Path $destination
$copyoptions = "/MIR"
$command = "robocopy `"$($source)`" $($destination) $copyOptions"
$output = Invoke-Expression $command
exit 0
```


In the above snippet, the `$(SiteWebroot)` is just a variable, in this case from VSTS, but could easily be swapped out for an Octopus Deploy variable.  The reason I am using RoboCopy is because it's "smarter" in the sense it will only copy over the changes: the `/MIR` option is a "mirror" option to more or less keep two directories in sync on demand.

Within a release/deploy process, I have two of these tasks defined very early on in the process, one like the above for the Sitecore `Webroot` and one for the `Datafolder`.  The first deploy/release backup tasks may be excrutiatingly long, but afterwards it should be exponentially quicker. The only other note I would make is that pruning the `temp/__UpgradeHistory` files before the backup might help save some space in the backup location:

### Prune Temp files Script
```powershell
$upgradeHistory = ($(SCWebroot) + "\temp\__UpgradeHistory"); 
Write-Host "Clearing out " $upgradeHistory 
If (Test-Path $upgradeHistory){
    Remove-Item $upgradeHistory -Recurse
}
```

The above may be too aggressive for your taste, but since it's just `Inline Powershell` tasks either in VSTS or Octopus Deploy, they are easily tailorable by release/environment/whatever.
