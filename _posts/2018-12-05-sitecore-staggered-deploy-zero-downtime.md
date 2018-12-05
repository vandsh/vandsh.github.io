---
layout: post
title: "Sitecore Staggered Deploys: Zero Downtime"
category: Sitecore
tags: Sitecore CI-CD PowerShell
published: true
---

I want to live in a world where the concept of a production deploy is simply clicking one button and waiting for a green checkmark.

### Prelude
After reading [Jeff Darchuks's post on Zero-Downtime PaaS Deployments](https://jeffdarchuk.com/2018/12/04/easy-sitecore-9-azure-paas-no-downtime-deployments/) I was reminded of a few recent projects we were involved in where we focused on `Zero-Downtime` and I wanted to get it on paper in any event it may help others. Unlike Jeff's scenario, we were not in Paas but in IaaS and On-Prem.  The two projects that come to mind had some big similarities out of the gate:

- 2 content delivery nodes behind load balancers
- 1 content authoring server
- deploying to everything all at once
- Bi-weekly production releases (small feature sets being released often)

So after having the discussion with both clients about how we could achieve "zero-downtime" with the only noticable impact would be an existing user session changing delivery nodes and that by achieving zero-downtime we could deploy during business hours with all hands on deck, we were given the green light. Our approach was pretty simple; assuming we have a solid build, and it has been put thru the paces on dev/qa/staging and we are ready for a production release:

### The Approach
- Release code and content to Content Management as normal
- Pause for CM verification (quick smoke test)
- Release to node 1 (of 2) 
  - Take node out of load-balancers
  - Stop IIS site
  - Deploy Code
  - Start IIS site
  - Pause for verification (quick smoke test)
  - Put node back into load-balancer
- Release to node 2 (of 2)
  - Take node out of load-balancers
  - Stop IIS site
  - Deploy Code
  - Start IIS site
  - Pause for verification (quick smoke test)
  - Put node back into load-balancer
- Profit

### Some Considerations
Before I continue, I would like to call out some points that aren't covered in the above that may be areas of concern for _others_ looking towards this same approach:
- For a more seemless fail-over from one node to the next sessions would need to be shared
- Content would need to freeze for the entirety of the Content Management release
- Content in release (templates/global items) would be deployed to Master, which is shared by both nodes (at least was in our situation) and published once both nodes are back and updated
  - This could have been an issue if we created code that did not fail gracefully when content didn't exist (not in `web` yet only `master`)
  - If this became more of a concern in our case, we were looking at spinning up a secondary set of databases for all and keeping them in-sync using replication, but we didn't need to cross this bridge
- The outward facing capacity of the site would effectively be cut in half since we are taking half of the available servers out.
  - This required us to simply focus on releasing on less "peak" times but still fine during the business day (start of business kick-offs etc).
- Depending on what your release platform was, isolating which server you had context to became tricky
  - Octopus handled the above scenario like a champ (since we could tell tasks which _exact_ servers to run on using roles/names)
  - Azure DevOps was tricky to start because we had been using Deployment Groups. You can check the "deploy sequentially" box rather than in parallel, but our end result was to have each server have its own Agent.
  - Creating separate agents allowed us to know _exactly which_ server we were and what we were doing, plus it allowed the CM server to handle the nodes being pulled out and put in the load-balancer within a single release pipeline (tasks broken up into 3 tracks with different agents specified: cm - pull out, cd - deploy, cm - put back in).

### PowerShell Goodies
So the above really covers the the "high level" approach, but what would a blog post be without some code (and yes this code assumes you are using Azure IaaS and is centrally located [here](https://github.com/vandsh/handy-spe-scripts/tree/master/Azure%20LoadBalancer)):

For the sake of the below let's assume we are in a working director of `C:\agent\A1\Azure LoadBalancer\` (sure it might look different in Ocotpus, but it shouldn't matter all that much, just need it to be in a place it can be executed from the agent/tentacle).

Before we get into Azure based stuff, we can start with the IIS site. To start and stop the IIS sites, executing the following `PowerShell` scripts accomplished that. This was explicitly done because a handful of times we got into a situation where a release would fail because the worker process was _still_ locking a dll:

## StopSite.ps1
```powershell
Param(
  [Parameter(Mandatory=$True,Position=1)]
  [string]$variablesFilePath
)
	$azureVariables = Get-Content -Raw -Path $variablesFilePath | ConvertFrom-Json
	$siteName = $azureVariables.iisSiteName
	Stop-WebSite -Name "$siteName"
```

## StartSite.ps1
```powershell
Param(
  [Parameter(Mandatory=$True,Position=1)]
  [string]$variablesFilePath
)
	$azureVariables = Get-Content -Raw -Path $variablesFilePath | ConvertFrom-Json
	$siteName = $azureVariables.iisSiteName
	Start-WebSite -Name "$siteName"
```
_Side node: the above does need you to have the Powershell module `WebAdministration` installed_

Okay, now we can get into some more Azure-y stuff. Let's create a `azureVariables.json` and update the values in that file to that of your own subscription and resources varying based on which server you are on:

## AzureVariables.json
```json
{
	"subscriptionId": "000000-0000-0000-0000-867j53e09n",
	"profilePath": "C:\\agent\\A1\\Azure LoadBalancer\\AzureProfile.json",
	"nicName": "ProdWinCD-01-nic",
	"resourceGroupName": "eastus-RSG-yoursite-PRD",
	"lbName": "LB-ProdCD",
	"poolName": "BackendPool-P-CD",
	"iisSiteName": "SCPRODCD"
}
```
From there, let's create and execute the `LoginAndStoreContext` which will basically create a token/context file for us to execute AzureRM functions from the server:

## LoginAndStoreContext.ps1
`usage: .\LoginAndStoreContext.ps1 "./azureVariables.json"`
```powershell
Param(
  [Parameter(Mandatory=$True,Position=1)]
  [string]$variablesFilePath
)
	$azureVariables = Get-Content -Raw -Path $variablesFilePath | ConvertFrom-Json

	Login-AzureRMAccount
	Select-AzureRmSubscription -Subscriptionid $azureVariables.subscriptionId
	New-Item -ItemType File -Force -Path $azureVariables.profilePath
	Save-AzureRmContext -Path $azureVariables.profilePath -Force
```
_re: Saving Context to file; to be honest I do not know if there is an expiration on saving context to file, I have to assume there is, but have had trouble tracking down anything documented_

Now that we have our Azure Context, we should be all clear to execute calls using it to our Resources. The first one we can knock out is the `RemoveNICFromBackendPool` script:

## RemoveNICFromBackendPool.ps1
`usage: .\RemoveNICFromBackendPool.ps1 "./azureVariables.json"`
```powershell
Param(
  [Parameter(Mandatory=$True,Position=1)]
  [string]$variablesFilePath
)
	$azureVariables = Get-Content -Raw -Path $variablesFilePath | ConvertFrom-Json
	
	Import-AzureRmContext -Path $azureVariables.profilePath
	$nic = Get-AzureRmNetworkInterface -name $azureVariables.nicName -resourcegroupname $azureVariables.resourceGroupName
	$nic.IpConfigurations[0].LoadBalancerBackendAddressPools=$null
	Set-AzureRmNetworkInterface -NetworkInterface $nic
```

The above script will pull your `azureVariables.nicName` out of the backend ip pool specified by `azureVariables.poolName`.  The following script will do the _exact opposite_ of the above and add it back in:

## AddNICToBackendPool.ps1
`usage: .\AddNICToBackendPool.ps1 "./azureVariables.json"`
```powershell
Param(
  [Parameter(Mandatory=$True,Position=1)]
  [string]$variablesFilePath
)
	$azureVariables = Get-Content -Raw -Path $variablesFilePath | ConvertFrom-Json
	
	Import-AzureRmContext -Path $azureVariables.profilePath
	$lb = Get-AzureRmLoadBalancer -name $azureVariables.lbName -resourcegroupname $azureVariables.resourceGroupName
	$backend = Get-AzureRmLoadBalancerBackendAddressPoolConfig -name $azureVariables.poolName -LoadBalancer $lb
	$nic = Get-AzureRmNetworkInterface -name $azureVariables.nicName -resourcegroupname $azureVariables.resourceGroupName
	$nic.IpConfigurations[0].LoadBalancerBackendAddressPools=$backend
	Set-AzureRmNetworkInterface -NetworkInterface $nic
```

In the [repository](https://github.com/vandsh/handy-spe-scripts/tree/master/Azure%20LoadBalancer) there are some `AppGateway` scripts as well, but I unfortunately did not have the time to test these (but feel free to poach at your own risk).

### Conclusion
So I covered a couple things here; a high level approach to Staggered deploys resulting in zero-downtime deployments and also included some (hopefully) helpful scripts to accomplish this approach when you are using Azure IaaS. I _know_ there is room for improvement in all of the above so feel free to ping me on the Slack or the Twitters with any comments/questions/grievances/etc.

Anyways, hope this helps!

</message>
