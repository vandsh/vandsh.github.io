---
layout: post
title: "Tag Helpers in dotnet MVC"
category: dotnet
tags: dotnet MVC ASP.Net
---

Ok, so every project you come across, especially when you get people logging in with various roles and responsibilities, you eventually get the request to have things conditionally show or hide based on those roles. Traditionally, I would have just used some good old `if` statements in the `View` (what fun is that). I poked around for a cooler solution and came across [`Tag Helpers`](https://docs.microsoft.com/en-us/aspnet/core/mvc/views/tag-helpers/intro?view=aspnetcore-3.0).  

I had seen this before: `<a class="nav-link" asp-controller="Account" asp-action="SignOut">Logout</a>` so I thought it was worth a try. What I wanted was a way to explicitly hide or show a menu item, a button, a link or any DOM element really for certain users.

Since I already had the user and role management pieces sorted out, and some `constants` for these roles, I figured I could pass them in to my `tag helper` to denote which one(s) to show or hide for.

## PermissionConstants.cs

```csharp
public static class PermissionConstants
{
    public static class Job
    {
        public const string Create = "JobCreate";
        public const string Read = "JobRead";
        public const string Update = "JobUpdate";
        public const string Deactivate = "JobDeactivate";
    }

    public static class User
    {
        public const string Create = "UserCreate";
        public const string Read = "UserRead";
        public const string Update = "UserUpdate";
        public const string Deactivate = "UserDeactivate";
        public const string ManageRoles = "UserManageRoles";
    }
}
```

The above `roles` were associated with a user and acquired by doing a `_userService.GetCurrentUserPermissionsAsync()`, of course with some proper caching so we weren't hammering a database with every instance of the tag helper.

## ShowForTagHelper.cs
```csharp
[HtmlTargetElement(Attributes="show-for-permission")]
[HtmlTargetElement(Attributes="hide-for-permission")]
public class ShowForTagHelper : TagHelper
{
    private readonly IUserService _userService;
    public ShowForTagHelper(IUserService userService)
    {
        _userService = userService;
    }

    /// <summary>
    /// Show this html element for a particular permission type
    /// </summary>
    [HtmlAttributeName("show-for-permission")]
    public string ShowForPermission { get; set; }

    /// <summary>
    /// Hide this html element for a particular permission type
    /// </summary>
    [HtmlAttributeName("hide-for-permission")]
    public string HideForPermission { get; set; }

    public override async Task ProcessAsync(TagHelperContext context, TagHelperOutput output)
    {
        base.Process(context, output);
        var currentUserPermissionsAll = await _userService.CurrentUserPermissionsAsync();
        if (currentUserPermissionsAll != null && currentUserPermissionsAll.Any())
        {
            var currentUserPermissions = currentUserPermissionsAll.Where(x => x.Value).Select(p => p.Key);
            bool showForUser = (ShowForPermission != null) ? currentUserPermissions
				.Intersect(_getPermissionsList(ShowForPermission)).Any() : true;
            bool hideForUser = (HideForPermission != null) ? currentUserPermissions
				.Intersect(_getPermissionsList(HideForPermission)).Any() : false;
            if (!showForUser || hideForUser)
            {
                output.SuppressOutput();
                //since at this point the output is suppressed, it doesn't pay to continue checking...
            }
        }
        else
        {
	    // if attribute exists, but user has no roles: hide
            output.SuppressOutput();
        }
    }

    private IEnumerable<string> _getPermissionsList(string permAttr)
    {
        return permAttr.Split(',');
    }
}
```

A quick unpacking of the above: 
 - `_userService` is injected into the constuctor
 - two distinct attributes defined 
   - `HtmlTargetElement` decorator associates the attributes to the helper
   - `HtmlAttributeName` associates the attributes to to concrete properties
 - override the `ProcessAsync` method to do your thing
   - `output.SuppressOutput()` is the magic that hides this from ever being rendered

## SomeMarkup.cshtml

```html
<!-- other markup, then an action link that only shows for people who have 
		the roles needed to view or update a Job -->

<a class="nav-link" asp-area="" asp-controller="Jobs" asp-action="Index" 
show-for-permission="@(PermissionConstants.Jobs.Read +","+PermissionConstants.Jobs.Update)">Profile</a>

<!-- more arbitrary markup, then an entire div hidden based on role -->

<div show-for-permission="@(PermissionConstants.User.Read +","+PermissionConstants.User.Update)">
    <!-- cool things only a user with User read/update perms can see or do -->
</div>
```

Hopefully the above shows you a real world case for using a custom `TagHelper`. Not only does it seem pretty clean and heavily reusable, but honestly it feels like it separates the concern of hiding or showing content based on a users permissions to a separate handler (almost AOP-esq, don't quote me on that tho!). 
