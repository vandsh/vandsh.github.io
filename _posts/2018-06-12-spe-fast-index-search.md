---
layout: post
title: "SPE: Fast and Index Searches"
category: Sitecore
tags: Sitecore Powershell SPE
---

Yeah, I like [Sitecore Powershell Extensions](https://marketplace.sitecore.net/en/Modules/Sitecore_PowerShell_console.aspx) and I saw someone doing [something script-like](https://sitecore.stackexchange.com/questions/12290/improving-queries-performance/12293#12293) so I thought I would venture out to the SPE docs and see what is supported.  Low and behold both `fast` and index based queries are right outta box, sweet!:
### Fast Query
```powershell
$query = "fast:/sitecore/content//*[@HaystackField1 = 'Needle' or @HaystackField2 = 'Needle']"
Get-Item -Path "master:" -Language * -Query $query
```
I find the ability to search an index from SPE pretty powerful in many ways, one of the first being that it's easier to read than trying to use something like [Luke](http://www.getopt.org/luke/) to determine if something exists in an index: 

### Index Search
```powershell
$root = (Get-Item "master:/sitecore/content/")
Find-Item -Index sitecore_master_index `
          -Criteria @{Filter = "DescendantOf"; Field = $root },
                    @{Filter = "Contains"; Field = "haystackfield1"; Value = "Needle"},
                    @{Filter = "Contains"; Field = "haystackfield2"; Value = "Needle"} |
    Initialize-Item 
```
SPE, you and I are becoming friends. 

Hope this helps someone.
