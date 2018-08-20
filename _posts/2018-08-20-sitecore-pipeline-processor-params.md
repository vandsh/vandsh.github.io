---
layout: post
title: "Sitecore Pipeline Processor Params"
category: Sitecore
tags: Sitecore
---

Countless times I run across needing to define a template id, a path, or really any hard coded string within a pipeline. I also end up finding myself scouring the interwebs to find a clear and concise examples of how to use and pass `<params>` into a pipeline (I know I have ran across plenty of them before but always lose them). Anyways, if you are here I assume you came for the code so: 

### The Pipeline/Processor Config XML
```xml
<httpRequestBegin> <!-- Or whatever pipeline you are tying into -->
	<processor type="Your.Framework.CustomProcessor, Your.Framework">
		<param desc="globalSettingsLocation">/sitecore/content/global/settings</param>
		<param desc="siteRootTemplate">{3BA5DDEC-70EA-4BF6-AC36-83CF341C646A}</param>
	</processor>
</httpRequestBegin>
```

What you see above is an example of what the `XML` _might_ look like when defining a pipeline processor with parameters.

Given the above `XML`, your actual processor would look something like this: 
 

### The Processor Code
```csharp
public class CustomProcessor
{
	private string _globalSettingsLocation;
	private string _siteRootTemplate;
	public CustomProcessor(string globalSettingsLocation, string siteRootTemplate)
	{
		_globalSettingsLocation = globalSettingsLocation;
		_siteRootTemplate = siteRootTemplate;
	}
	public void Process(HttpRequestArgs args)
	{
		//Do things with your important '_settingsTemplate' and '_siteRootTemplate' variables
		//like GetItems or maybe check for an ancestor of a particular template type...
	}
}
```

If this post helped you learn something, great! If it was something you already knew but wanted yet another example, also great!

_This post only serves as an example and should not be used to assume that I do or do not endorse using global settings items or searching for anscestors of particular "site root" types in any way shape or form. Also not looking to getting into a Configuration in Code versus File debate. KThxBai._

`</end of post>`
