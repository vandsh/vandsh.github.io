---
layout: post
title: "SecureClientSecret one time use"
category: dotnet
tags: dotnet Graph AD
---

Hooking up `Graph` for a recent project where I wanted to obtain a little more information from `Azure Active Directory` from the user immediately after login, I started running into a few snags. 

First, the [code I was using as a guide](https://github.com/microsoft/Partner-Center-Storefront/blob/35abe9e988ab2fd6f612146c74c71db8d3f0b57e/src/Storefront/App_Start/Startup.Auth.cs#L77) was using the `authorizationCode` form the original sign in to obtain an access token for graph. 
That is just fine and acceptable, until you need to make another call. 
Simply put, sometime in Sept of 2018, [AAD stopped letting Auth codes be reused](https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/whats-new-archive#change-notice-authorization-codes-will-no-longer-be-available-for-reuse).

Frustrated but not defeated, I found a [super helpful post](https://www.linkedin.com/pulse/handling-error-aadsts54005-nicholas-mccollum/) on working thru this change. 

### Startup.Auth.cs
_Close but not quite..._
```csharp
// Within OpenIdConnectAuthenticationNotifications()
AuthorizationCodeReceived = (context) =>
{
    // Get these for later
    var redirectUri = new Uri(HttpContext.Current.Request.Url.GetLeftPart(UriPartial.Path));
    string signedInUserObjectId = context.AuthenticationTicket.Identity.FindFirst("http://schemas.microsoft.com/identity/claims/objectidentifier").Value;
    string userTenantId = context.AuthenticationTicket.Identity.FindFirst("http://schemas.microsoft.com/identity/claims/tenantid").Value;

    // Like a good kid, we are going to store our secrets in Azure KeyVault
    var redirectUri = new Uri(HttpContext.Current.Request.Url.GetLeftPart(UriPartial.Path));
    KeyVaultService keyVaultService = new KeyVaultService();
    var secureString = await keyVaultService.GetAsync(ApplicationConfig.ActiveDirectoryPortalKey);
    SecureClientSecret clientSecret = new SecureClientSecret(secureString);

    // Obtain an access token for the current application using the authorization code (modified from that Linkedin Post)
    ClientCredential credential = new ClientCredential(ApplicationConfig.ActiveDirectoryClientID, clientSecret);
    AuthenticationContext authContext = new AuthenticationContext($"{ApplicationConfig.ActiveDirectoryEndPoint}{userTenantId}");
    AuthenticationResult result = authContext.AcquireTokenByAuthorizationCodeAsync(context.ProtocolMessage.Code, redirectUri, credential, context.Options.ClientId).Result;

    // Obtain and cache access tokens for additional resources using the access token
    // from the application as an assertion (modified from Linkedin Post)
    UserAssertion userAssertion = new UserAssertion(result.AccessToken);
    AuthenticationResult graphResult = authContext.AcquireTokenAsync("https://graph.microsoft.com", credential, userAssertion).Result;
    
    // Actual calls to Graph, blows up before even getting here...
    IGraphClient graphClient = new GraphClient(userTenantId, graphResult);
    List<RoleModel> roles = await graphClient.GetDirectoryRolesAsync(signedInUserObjectId).ConfigureAwait(false);
...
```

It got me about 90% of the way there, but as we all know, it is that last 10% that is the hardest. 
The part that was failing hard was the `graphResult = authContext.AquireTokenAsync` call. 
I kept getting a terribly vague `Object reference not set to an instance of an object.` error from a proper MS class, which only added to the frustration.
Finally after hours of debugging, searching and all those things devs do when they can't figure something out, I found a trail leading `SecureClientSecret` being one time use...


### Startup.Auth.cs
_Much betta!_
```csharp
AuthorizationCodeReceived = async (context) =>
{
    // Same as above
    var redirectUri = new Uri(HttpContext.Current.Request.Url.GetLeftPart(UriPartial.Path));
    string signedInUserObjectId = context.AuthenticationTicket.Identity.FindFirst("http://schemas.microsoft.com/identity/claims/objectidentifier").Value;
    string userTenantId = context.AuthenticationTicket.Identity.FindFirst("http://schemas.microsoft.com/identity/claims/tenantid").Value;

    // Same as above
    KeyVaultService keyVaultService = new KeyVaultService();
    var secureString = await keyVaultService.GetAsync(ApplicationConfig.ActiveDirectoryPortalKey);
    SecureClientSecret clientSecret = new SecureClientSecret(secureString);

    // Same as above
    ClientCredential credential = new ClientCredential(ApplicationConfig.ActiveDirectoryClientID, clientSecret);
    AuthenticationContext authContext = new AuthenticationContext($"{ApplicationConfig.ActiveDirectoryEndPoint}{userTenantId}");
    AuthenticationResult result = authContext.AcquireTokenByAuthorizationCodeAsync(context.ProtocolMessage.Code, redirectUri, credential, context.Options.ClientId).Result;

    // Create a SECOND SecureClientSecret, just for the Graph calls...
    SecureClientSecret graphClientSecret = new SecureClientSecret(secureString);
    ClientCredential graphClientCredential = new ClientCredential(ApplicationConfig.ActiveDirectoryClientID, graphClientSecret);
    UserAssertion userAssertion = new UserAssertion(result.AccessToken);
    AuthenticationResult graphResult = await authContext.AcquireTokenAsync("https://graph.microsoft.com", graphClientCredential, userAssertion);
    
    // No explosions, works as expected...
    IGraphClient graphClient = new GraphClient(userTenantId, graphResult);
    List<RoleModel> roles = await graphClient.GetDirectoryRolesAsync(signedInUserObjectId).ConfigureAwait(false);
...
```

Not going to lie, I feel a bit miffed at all of this. 
There is some weak documentation supporting this was done "intentionally" for security purposes. 
Having it documented in the class metadata itself or make it disposable if you are _that_ concerned, anything along those lines would have saved me several hours of hair pulling.

Don't worry MS, I won't send you the bill.




