---
layout: post
title: "Sitecore Search - Getting Result Items"
category: Sitecore
tags: Sitecore Search Solr
---

A recent conversation or debate around how to get the actual values of a list of search results has lead me to write this post.

### The Argument ###
In our recent debate around how to obtain the values for resulting search items, there was ultimately two camps and their arguments:

- Stuff everything you will need to render in the index and map/display it from whatever `SearchResultItem` you get
  - Faster performance-wise
  - Easier to just have one model rather than pulling from another source and mapping
- Stuff only what you will ever be searching on in the index and get the actual data for the item from the Application
  - More separated concerns
  - Single source of truth is the application

### The Control ###
Using [Sitecore Powershell Extensions](https://marketplace.sitecore.net/en/Modules/Sitecore_PowerShell_console.aspx) I wrote a script to create 5000 Demo Items in a `bucket`. That `Demo Item` type had 15 very simple `Single-Text` fields:

```powershell
for($i = 0; $i -lt 5000; $i++ )
{
    $name = "DemoItem$i"
    $newItem = New-Item -Path "master:\content\Sites\Global\Large Folder\$name" -ItemType "User Defined/Demo Item"
    $newItem.Editing.BeginEdit()
    #Make values that are unique to field and item but predictable so we can know how many results are expected
    $newItem["Field1"] = ("Field1$name")
    $newItem["Field2"] = ("Field2$name")
    $newItem["Field3"] = ("Field3$name")
    ...
    $newItem["Field15"] = ("Field15$name")
    $newItem.Editing.EndEdit()
}
```

I then created two indexes in order to test: 
- `sknny` which holds the only 2 fields that I would ever want to search on (think of it maybe as `Title` and `Descirption` fields) and who's results would be grabbed from the Application.
- `thick` which holds all 15 fields we will most likely be rendering on the page, only searching really on 2 (Search on `Title, Description` but show `Teaser Image, Path, Link, etc`)

### The Test ###
Below you will see the script I executed in order to get some metrics on both approaches.  

Starting with the indexes themselves, I want to make sure they are in complete lockstep, so let's rebuild. This will also show us the difference in time it takes to actually execute the rebuild. 

Next, I wanted to come up with 5 random yet predictable field values between both indexes, so I grabbed 5 numbers between 1-9. Reason I picked 1 thru 9 is because it will provide the most search `hits` over the field values (ie: `Contains` Field1DemoItem3 will hit 3 and everything in 30's,300's,3000's). 

Now that we have our shared search values, let's execute the search over both indexes with a set page size and the only difference between the two will be the `Initialize-Item` call on the `skinny` index that will be essentially retrieving the proper item from the Application (in this case the database).  The script executes a search for all 5 and averages the execution time. Behold, the script below:

```powershell
$skinny = [Sitecore.ContentSearch.ContentSearchManager]::GetIndex("sitecore_skinny_index")
$thick = [Sitecore.ContentSearch.ContentSearchManager]::GetIndex("sitecore_thick_index")

#Rebuild Indexes to assure they are in sync
Write-Host "Rebuilding the $($skinny.Name) search index."
$time = Measure-Command {
    $skinny.Rebuild()
}
Write-Host "Completed rebuilding the $($skinny.Name) search index in $($time.TotalSeconds) seconds." 

Write-Host "`nRebuilding the $($thick.Name) search index."
$time2 = Measure-Command {
    $thick.Rebuild()
}
Write-Host "Completed rebuilding the $($thick.Name) search index in $($time2.TotalSeconds) seconds."

#Search

#Get 5 random but predictable field values
$seed1 = Get-Random -Minimum 1 -Maximum 9
$seed2 = Get-Random -Minimum 1 -Maximum 9
$seed3 = Get-Random -Minimum 1 -Maximum 9
$seed4 = Get-Random -Minimum 1 -Maximum 9
$seed5 = Get-Random -Minimum 1 -Maximum 9
$seeds = @($seed1, $seed2, $seed3, $seed4, $seed5)
$pageSize = 10

Write-Host "`nSearching the $($skinny.Name) search index. Index search, data from proper Item."
$time3Total = 0;
$foundCount = 0
$seeds | Foreach{
    $currentSeed = $_
    $time3 = Measure-Command {
        $foundItems = Find-Item -Index $skinny.Name -Criteria @{Filter = "Contains"; Field = "field1_t_en"; Value = "Field1DemoItem$currentSeed"  } -First $pageSize | Initialize-Item
        $foundCount += $foundItems.Count
    }
    $time3Total += $time3.TotalMilliseconds
}
$time3Average = $time3Total / $seeds.Length
Write-Host "Completed searching, found $foundCount items in search index in $time3Average ms."

Write-Host "`nSearching the $($thick.Name) search index. Index data only."
$time4Total = 0;
$foundCount2 = 0
$seeds | Foreach{
    $currentSeed = $_
    $time4 = Measure-Command {
        $foundItems2 = Find-Item -Index $thick.Name -Criteria @{Filter = "Contains"; Field = "field1_t_en"; Value = "Field1DemoItem$currentSeed" } -First $pageSize
        $foundCount2 += $foundItems2.Count
    }
    $time4Total += $time4.TotalMilliseconds
}
$time4Average = $time4Total / $seeds.Length
Write-Host "Completed searching, found $foundCount2 items in search index in $time4Average ms."
```

### The Results ###

The first result I want to cover is the rebuild time:

```text
Rebuilding the sitecore_skinny_index search index.
Completed rebuilding the sitecore_skinny_index search index in 4.8936487 seconds.

Rebuilding the sitecore_thick_index search index.
Completed rebuilding the sitecore_thick_index search index in 10.6213085 seconds.
```

Consistently, the `thick` index took twice as long to rebuild than the `skinny` which makes sense. And really these are both super simple indexes to begin with, only `Single-Line Text`, there is no related rendered content, no rich text, no guids, no computed fields... nothing.

The next result we will look at is the search, given a page size of 10:

```text
Searching the sitecore_skinny_index search index. Index search, data from proper Item.
Completed searching, found 50 items in search index in 61.42852 ms.

Searching the sitecore_thick_index search index. Index data only.
Completed searching, found 50 items in search index in 62.98682 ms.
```

After running this a handful of times, there was a <5 ms difference between the two both ways so to me over a page size of 10: no clear performance difference.

Bumping up the page size to 25 yielded a slightly larger yet nearly neglible performance difference (somewhere between 5 and 15 ms difference):

```text
Searching the sitecore_skinny_index search index. Index search, data from proper Item.
Completed searching, found 125 items in search index in 79.08638 ms.

Searching the sitecore_thick_index search index. Index data only.
Completed searching, found 125 items in search index in 67.20058 ms.
```

This trend obviously continues (and makes logical sense) when the page size continues to grow (seeing around 20ms-ish difference with a page size of 50). 

### The Conclusion ###

In the end, how you develop search is completely up to you the developer.  But given the findings above, I did not see the difference between the two different search result approaches I was expecting.  I was whole-heartedly expecting the approach getting proper Sitecore items to be consistently a factor of two slower than the raw index results. I still didn't even see a factor of 2 difference between the two even over a page size of 1000 (even at that, only 50% slower). Another aspect of this test, the `Initialize-Item` (to the best of my understanding) is simply doing a `.GetItem()` on the `SearchResultItem` returned from the index. Given those result items are being piped in to the `Initialize-Item` method, they will be essentially iterated over. This may not be the most efficient way to get a result set of items from Sitecore and if you are someone who also builds in some sort of in-memory caching or lazy-loading, this certainly won't be taking any of that into account either.

In my opinion, the small cost difference between the two can easily be made up with a few tweaks in the underlying item lookup mechanisisms and the long term developer efficiencies of not having to follow the `Just add another field to the Solr index for it to show up on the front-end` paradigm is well worth it.  `Keep your page sizes manageable, keep your index lean, don't use your search mechanisism as a datastore; leave that for the application`, that's an easier paradigm to follow.

And with everything, there is always a middle ground between these two cases as well.

If you see any glaring holes in this test, feel free to reach out to me on [Twitter](https://twitter.com/vandsh) or [Slack](sitecorechat.slack.com).

Hope this helps!

