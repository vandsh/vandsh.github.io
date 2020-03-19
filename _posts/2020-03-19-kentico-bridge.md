---
layout: post
title: "Bridge: YAML CI for Kentico"
category: kentico
tags: Kentico Deploy CI-CD
---

Having worked in Kentico for a while, something that keeps coming up in the back of my mind is: why hasn't someone developed a friendly tool to manage the database items? I was looking for something using YAML so it's easy to merge and use in source control, completely free and open-source, config driven and (last but not least) able to be leveraged in a dev-ops workflow.

Ultimately I wanted something like [SitecoreUnicorn](https://github.com/SitecoreUnicorn/Unicorn) but for Kentico. In my quest to find this tool, I didn't find many that checked all these req's so I decided to just start throwing things at a wall to see what stuck. After some dev effort and show and tell with colleagues, I have what I consider not even a `beta` solution yet. 

Let me introduce to you `Bridge` for Kentico!

# -|--|- Bridge

![alt text](/assets/bridgescreenshot.png "Bridge Screenshot")

## The what? 
 - A Yaml based service for handling the promotion of Kentico CMS based items thru source control.

## The why? 
 - Because XML is terrible to read and even harder to merge. The reason I created this was to allow us to not only break down features modularly thru some basic configs, but I wanted to make sure it was easy to adopt and that means making it easy to use with source control.

## The when? 
 - Use this either along side an existing Kentico build using the [nuget](https://www.nuget.org/packages/Bridge.Kentico/) or standalone by cloning this repo and updating the `connectionString` parameters to match your Kentico db.

## The how?
 - Installation:
     - *Standalone*: clone, compile and run the app. Fire up a browser and go to `localhost/bridge/index`
     - *Alongside*: install via [nuget](https://www.nuget.org/packages/Bridge.Kentico/) and navigate to `yourKenticoUrl/Admin/BridgeUI/index`
 - The configs:
     - *CoreConfig*: one or more core config can be specified. They contain a list of `classtypes` to care about and a list of `ignoreFields` on the item to simply ignore when serializing/syncing.
     - *ContentConfig*: one or more content config can be specified. Contains a list of `pagetypes` to care about and a list of `ignoreFields` on the items to not serialize/sync. This also will handle all custom fields. Also, a `query` attribute allows you to pick which content within the tree to focus on. This should support a basic Kentico Query.
 - Using the GUI:
     - Core Configs:
        - Core Configs *Serialize All*: iterates thru all of the defined `coreConfig` nodes and fetches all the corresponding classes, and stuffs them into the `serialization/core/{coreConfigName}` folder.
        - Core Configs *Sync All*: this is the inverse of the `Serialize All` task, it takes what's in the file system and pushes it to the Kentico DB.
        - "Named Core Config" *Serialize*: pulls _just_ the specified config down to the file system
        - "Named Core Config" *Sync*: pushes _just_ the specified config up to the database
        - "Named Core Config" *Diff*: does a temp serialize of the current Kentico DB and compares it to what's in the filesystem and spits out a diff.
    - Content Configs:
        - Content Configs *Serialize All*: iterates thru all of the defined `contentConfig` nodes and fetches all the corresponding classes, and stuffs them into the `serialization/content/{contentConfigName}` folder.
        - Content Configs *Sync All*: this is the inverse of the `Serialize All` task, it takes what's in the file system and pushes it to the Kentico DB.
        - "Named Content Config" *Serialize*: pulls _just_ the specified config down to the file system
        - "Named Content Config" *Sync*: pushes _just_ the specified config up to the database
        - "Named Content Config" *Diff*: does a temp serialize of the current Kentico DB and compares it to what's in the filesystem and spits out a diff.
 - The Endpoints:
     - Within a dev-ops cycle (ie a release to a dev or test environment) make a call to the Bridge endpoints using an authenticated call (using something like `curl`, `wget` or `Invoke-WebRequest`).
     - The response itself should be a stream so it will respond as things occur rather than processing everything and responding.
     - The endpoints are as follows:
         - `yourUrl/bridge/diffcore?/{configName}`: returns the comparison between the serialized DB and what is in the file system for the optional core configName
         - `yourUrl/bridge/diffcontent?/{configName}`: returns the comparison between the serialized DB and what is in the file system for the optional content configName
         - `yourUrl/bridge/serializecore?/{configName}`: serializes the specified core config (or all if none specified) to the filesystem
         - `yourUrl/bridge/serializecontent?/{configName}`: serializes the specified content config (or all if none specified) to the filesystem
         - `yourUrl/bridge/synccore?/{configName}`: syncs the specified core config (or all if none specified) to the database
         - `yourUrl/bridge/synccontent?/{configName}`: syncs the specified content config (or all if none specified) to the database
     - *My ultimate goal is to leverage the endpoints in order to promote a continuous integration and deployment methodology within the Kentico community*

## The dev questions?
 - What kind of Class Types do the `CoreConfigs` include?
     - Any custom class you create in Kentico should be covered: page types, custom tables etc.
 - What are the `ignoreFields`?
     - Fields that we are just going to ignore, either they are tied to users, sites, timestamps or other content.
     - We strip them so we can insert the content cleanly without reference to the originating source site.
 - What fields do `ContentConfigs` include?
     - Anything not in the `ignoreFields` should be written to YAML.
     - This includes custom fields.
 - Security?
     - Bridge actually ties into the Kentico membership provider.
     - Standalone: Using the NancyFx Basic Auth provider, it forwards the basic auth creds into the Kentico Provider to validate you are a user.
     - Alongside: it picks up the `HttpContext.CurrentUser.IsAuthenticated` from your logged in Kentico session.
     - Security wise, this is really just checking to make sure you are _an authenticated user, nothing more_, but we could in the future check for roles or claims.
     - If installing this in (eventually) a staging or production environment, you may want to consider using it only during the release process and then (programmatically) deleting the `/Bridge` folder and/or removing the `NancyHttpRequestHandler` entry in the `web.config`
 - Why [Nancy](https://github.com/NancyFx/Nancy)?
     - Low-ceremony dependency injection: so I could see about making this tool more universal
     - Lightweight and self-contained: I don't know how people want to use this tool: standalone, alongside kentico?
     - Simple request control: I didn't want to have to worry about the routes someone was using in their Kentico site.
 - Why YAML?
     - Easy to read, easy to merge. I wanted to defer as much of this process to standard tooling like merging item conflicts using Kdiff/WinMerge etc.
     - Smaller file footprint: I didn't need all of the wrapper XML nodes.
 - Future plans?
     - Please feel free to hit me up if you like the tool but are having issues or would like to see any new features
     - One thing I have already kinda planned for is allowing for multi-site configs, allowing you to handle different sites all within one GUI.
     - I have also considered abstracting out the Kentico libraries and making this into a tool that could handle a raw database as well, time will tell.

## Disclaimer

This is in alpha for a reason: it's incredibly green and raw. I am hoping over time it ages like a fine wine. 
With that said, use at your own risk until you are comfortable with the changes you are making.

Anyways, thanks for reading!
