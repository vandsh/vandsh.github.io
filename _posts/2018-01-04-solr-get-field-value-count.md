---
layout: post
title: "Solr Get Values and Count for Field"
category: Sitecore
tags: Sitecore Solr Indexing
---

Found this super helpful when trying to see if a field is even getting indexed at all, and if so what the values are so you can test your query logic with _actual_ values in the index.

The same thing can be achieved in the Solr Admin by setting `rows` = 0, check the `facet` box, 
and put the field you want to know the values of in the `facet.field` textbox.

```
yoursolrurl:8983/solr/your_index/select?q=*%3A*&rows=0&wt=json&indent=true&facet=true&facet.field=field_you_want_to_know
```

_Example below will return all template names and associated counts of index items with that value:_
```
http://yoursolrurl:8983/solr/sitecore_master_index/select?q=*%3A*&rows=0&wt=json&indent=true&facet=true&facet.field=_templatename
```


