---
layout: post
title: "Sitecore Gated Search"
category: Sitecore
tags: Sitecore Search Solr
---

During a recent project, we had the request to implement a "gated" search; when people were logged in, we wanted to show them things in search that they had _permission_ to see, where as if they were an anonymous user viewing search they would only see content marked as read by `Everyone`. On top of this, the Sitecore instance was multi-site (approximately 120 different sites each with their own `Security Domain`). 

We fell back to what we knew best: a `Computed` search field. As you can see below, we are taking the item that is currently being indexed and looking up which roles `CanRead` from it:

### The Computed Field ###
```csharp
public class ReadAccessComputedField : IComputedIndexField
{
    public string FieldName { get; set; }
    public string ReturnType { get; set; }
    public object ComputeFieldValue(IIndexable indexable)
    {
        var sitecoreIndexable = indexable as SitecoreIndexableItem;
        if (sitecoreIndexable == null) return null;
        var item = (Item)sitecoreIndexable;
        var rolesForRead = new List<string>();

        // only indexing content security roles and anonymous at the moment
        var roles = new List<string>();
        roles.Add("Everyone");
        roles.Add("default\\Anonymous");
        roles.Add(string.Format("{0}\\Anonymous", Sitecore.Context.Domain));
        var contentSecurityRoles = _getContentSecurityRolesMappings().Select(x => x.Role).ToList();
        roles.AddRange(contentSecurityRoles);

        var targetSite = Sitecore.Links.LinkManager.ResolveTargetSite(item);
        using (new SiteContextSwitcher(Factory.GetSite(targetSite.Name)))
        {
            foreach (var role in roles)
            {
                var currentRole = Role.FromName(role);
                using (new SecurityEnabler())
                {
                    var canRead = item.Security.CanRead(currentRole);
                    if (canRead)
                    {
                        rolesForRead.Add(currentRole.Name);
                    }
                }
            }
            return rolesForRead;
        }
    }

    /// <summary>
    /// Gets an explicit list of roles that are applied to content 
    /// to "gate" it from anonymous users rather than trying to iterate
    /// thru every role in Sitecore
    /// </summary>
    /// <returns>List&lt;ContentSecurityRole&gt;</returns>
    public static List<ContentSecurityRole> _getContentSecurityRolesMappings()
    {
        var lst = new List<ContentSecurityRole>();
    
        // Read the configuration nodes
        foreach (XmlNode node in Factory.GetConfigNodes("yourSite.Account/contentSecurityRole"))
        {
            //Create a element of this type
            var elem = new ContentSecurityRole();
            elem.Name = XmlUtil.GetAttribute("name", node);
            elem.Role = XmlUtil.GetAttribute("value", node);
            lst.Add(elem);
        }

        return lst;
    }
}
```

As you can see above, the `_getContentSecurityRolesMappings` gets a very explicit list of `Roles` that we are actually using to restrict the content. This was significantly faster than iterating thru _every_ role in Sitecore per item.  That explicit mapping took place in a separate config:

```xml
<?xml version="1.0"?>
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
  <sitecore>
    <yourSite.Account>
      <contentSecurityRole name="Pizza Group 1" value="yoursite_domain\Pizza_Role" />
      <contentSecurityRole name="Cake Group 1" value="yoursite2_domain\Cake_Role" />
    </yourSite.Account>
  </sitecore>
</configuration>
```

The above `xml` mapped into the following class within the `_getContentSecurityRolesMappings` method:

```csharp
public class CommunityMapping
{
    public string Name { get; set; }
    public string Role { get; set; }
}
```

Ok, so at this point we have a new field with all the applicable `Read` roles into it, now we need to filter our content by this role (the easy part). I personally love the `PredicateBuilder` so I will use that as an example, but you should be able to massage this into whatever approach you are using as well:

### The Filter ###
```csharp
//deep in the bowels of a search service...

var mainPredicate = PredicateBuilder.True<ContentSearchResultItem>();

// other predicates like content type filtering, path filtering here....

var accessPredicate = PredicateBuilder.False<ContentSearchResultItem>();
var currentUserRoles = Sitecore.Context.User.Roles.Select(x => x.Name);
accessPredicate = accessPredicate.Or(a => a.ReadAccess.Contains("Everyone"));
accessPredicate = accessPredicate.Or(a => a.ReadAccess.Contains("default\\Anonymous"));
accessPredicate = accessPredicate.Or(a => a.ReadAccess.Contains(string.Format("{0}\\Anonymous", Sitecore.Context.Domain)));

foreach (var currentUserRole in currentUserRoles)
{
    accessPredicate = accessPredicate.Or(a => a.ReadAccess.Contains(currentUserRole));
}

mainPredicate = predicate.And(accessPredicate);
// get results, paging, you know the drill
```

So the above will create an `Or` filter for any of the roles that will likely apply to the current user (`anonymous` or any current roles I may be logged in to) and pass that along to filter the content that applies to that set of roles.

And that my friends is what we did to satisfy the need for gated content in search within a multi-site environment.

Hope this helps!

