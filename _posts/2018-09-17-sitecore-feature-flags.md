---
layout: post
title: "Sitecore Feature Flags Module"
category: Sitecore
tags: Sitecore Feature-Flags
---

I would like to introduce a module that I have been working on for a while: [Sitecore Feature Flags](https://github.com/vandsh/sitecore-feature-flags)!

### What is it? ###
I should have probably named it `Sitecore Feature Rules` or `Sitecore Dynamic Features`, but `Sitecore Feature Flags` is an interpretation of `Feature Flag` development built within `Sitecore`. Instead of an item with a master list of all renderings/modules and a `toggle` or `flag` that enables or disables specific components, this module  utilizes `Rules` to enable or disable modules at the point of Content Entry. Why? Because rather than executing logic on _every_ request to determine if a rendering/module should be displayed to an end user based on some curated list, let's focus on "guiding" the content editors away from modules that are works in progress but deployed. Given this is more of a `conditinal rendering` using `Rules` approach than it is a `flag|toggle` approach, it actually gives a tremendous amount of power to Content Editors than _just_ enabling and disabling modules; it allows controls to be allowed based on _any_ rules in Sitecore (specific users, specific tree loctions, even a macro to determine what `Site` you may currently be under).  

To illustrate what this means, let's assume you have a Sitecore instance with 2 sites, one being a global site and the second being a region specific sub-site. Let's also assume that you are creating a new module and your creative team has only given you designs for the global instance. Instead of managing the controls by creating a seperate `page template` and `placeholder setting` for the sub-site, you can levarage one shared `placeholder setting` but only allow the global site to use that control using `Sitecore Feature Flags` module, defining a rule to only allow the control in the placeholder if it's current site is `Global`.

### What does it do? ###
- Under the hood 2 new processors are added via  `Sitecore.FeatureFlags.config` to the following `pipelines`:
  - `getPlaceholderRenderings` to execute the logic to add/prune allowed controls in `Experience Editor`.
  - `getLookupSourceItems` to execute the logic to add/prune allowed options when a `template` inherits from `_Module Options` base template.
 
### What do I need for it? ###

- Sitecore and the [installation package](https://github.com/vandsh/sitecore-feature-flags/tree/master/Sitecore%20Packages) at a minimum.  Has been tested on 8.2, and 9.0

### What do I need if for? ###

- If you want to be able to deploy your site changes without having a module/component completely buttoned up
- If you want to have more dynamic control of your modules
- If you want to minimize your `placeholder setting` definitions

### How do I use it ###
Should be as easy as:

1. Install the package in [Sitecore Packages](https://github.com/vandsh/sitecore-feature-flags/tree/master/Sitecore%20Packages)

1. When creating a new/editing an existing `placeholder setting`, look for the `Allowed Modules Rule` field. Define your rules here and look for the `Feature Flag` tagged rules and actions. These will allow you to filter modules by `Site` and use the `allow|block control on this item`

1. When creating a folder that stores arbitrary data items (ie: tags, background colors, styles etc) use the `Module Options Folder` or use the `_Module Option` base template to give you the ability to use the `Feature Flag` rules on said folder.  This will give you the ability to exclude _child_ times from a list when this folder is used as a datasource (ie: exclude the `Grey Background` style from the Global site). This allows you to keep one source of truth for your datasource rather than having to have separate folders per site.

1. When creating data items that may sit under a page type (think `About Us/_content/Hero`) you can leverage the `SiteInfo` macro in `Feature Flags` to block that item from being inserted using the standard `Insert Options` rule for specific sites if the module is not ready to be used.

### Video ### 

Video of usage (creating a rule on the `placeholder setting` to remote the `Sample Rendering 2` component from all sites except for `Global`):

<iframe width="760" height="465" src="https://www.youtube.com/embed/8R5mU7lOWIs" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

Video of usage (creating a `Modules Options` folder with child items) that is referenced in the `Sample Item` template. Created rule excluding `Green Background` from everywhere _other_ than `Global`.

<iframe width="760" height="465" src="https://www.youtube.com/embed/fPhw1trBxMI" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

### Screenshots ###

![alt text](https://github.com/vandsh/sitecore-feature-flags/raw/master/moduleOptions.png "Module Options")

![alt text](https://github.com/vandsh/sitecore-feature-flags/raw/master/placeholderSettings.png "Placeholder Settings")

Look for a more in-depth write up soon on [Geekhive's website](https://www.geekhive.com) and hopefully on [Marketplace](https://marketplace.sitecore.net/) as well.

Hope this helps!
