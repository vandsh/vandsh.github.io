---
layout: post
title: "Testing Webhooks or B2C Policies using Azure Functions"
category: Azure
tags: Azure Webhooks B2C
---

This post is going to be an incredibly short and sweet post covering the ability to test things like `Webhooks` or `Azure B2C` policies. If you are familiar with either, you will already know they require a generally publicly accessible service running in order to process the request. In my partuclar case, creating an invitation  `Policy` within `Azure B2C`, I wanted to trigger a "Welcome" email from this policy, but really had no clue how the `Claims` or request headers came across. So I used what I knew: I created an `Azure Function` that would grab all the things of the request and email it to me.

In this case, I want a simple `Http Triggered` function with with `Function` auth level and supported `GET|POST` http methods. I gave it a name (`WebhookPayloadEmail`), dropped the below code in the `run.csx` and added entries for `SmtpServer`, `SmtpUserName` and `SmtpPassword` to the `Application Settings` of the `Function App` itself. 
Once done with all that, I ran it a few times locally to make sure it compiled, threw a few arbitrary values in the query, headers and request body to make sure they all got caught and once I confirmed that working, I figured we were ready to go and grabbed the `Function` url (`</> Get function URL`).

## WebhookPayloadEmail Function
```csharp
#r "Newtonsoft.Json"

using System.Net;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Primitives;
using Newtonsoft.Json;
using System.Net.Mail;

public static async Task<IActionResult> Run(HttpRequest req, ILogger log)
{
    log.LogInformation("C# HTTP trigger function processed a request.");
    var smtpHost = GetEnvironmentVariable("SmtpServer");
    var smtpPort = 587;
    var smtpEnableSsl = true;
    var smtpUser = GetEnvironmentVariable("SmtpUserName");
    var smtpPass = GetEnvironmentVariable("SmtpPassword");
    string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
    string serializedQuery = JsonConvert.SerializeObject(req.Query);
    string seralizedHeader = JsonConvert.SerializeObject(req.Headers);

    using (var client = new SmtpClient(smtpHost, smtpPort))
    {
        client.UseDefaultCredentials = false;
        client.Credentials = new System.Net.NetworkCredential(smtpUser, smtpPass);
        client.DeliveryMethod = SmtpDeliveryMethod.Network;
        client.EnableSsl = smtpEnableSsl;
        MailMessage message = new MailMessage("from@test.test", "to@test.test");
        message.Subject = "Test Submission";
        message.IsBodyHtml = false;
        message.Body = requestBody + "\n\n" + serializedQuery + "\n\n" + seralizedHeader;
        
        try
        {
            client.Send(message);
            log.LogInformation("Success.");
            

            return (ActionResult)new OkObjectResult(
                new {
                    status = true,
                    message = string.Empty
                });
        }
        catch (Exception ex)
        {
            log.LogError("Failure: " + ex.ToString());
            return new BadRequestObjectResult(new
                {
                    status = false,
                    message = "Check Azure Function Logs for more information."
                });
        }
    }
}

public static string GetEnvironmentVariable(string name)
{
    return Environment.GetEnvironmentVariable(name, EnvironmentVariableTarget.Process);
}
```

Above is the main guts of this post but to round things out, in my `Azure B2C` scenario, I created a `ClaimsProvider` (see below) to trigger on the last `OrchestrationStep` of the `Invitation` of the invitation `UserJourney`:

```xml
<ClaimsProvider>
    <DisplayName>Azure Functions</DisplayName>
    <TechnicalProfiles>
        <!-- The following technical profile sends a mail message to a user. -->
        <TechnicalProfile Id="AzureFunctions-SendMailWebHook">
            <DisplayName>Send Mail Web Hook Azure Function</DisplayName>
            <Protocol Name="Proprietary" Handler="Web.TPEngine.Providers.RestfulProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null" />
            <Metadata>
                <Item Key="ServiceUrl">https://yourtenant.azurewebsites.net/api/WebhookPayloadEmail?code=T4Isi2N0tAR3L1C0D3==</Item>
                <Item Key="AuthenticationType">None</Item>
                <Item Key="SendClaimsIn">Body</Item>
                <Item Key="AllowInsecureAuthInProduction">true</Item>
            </Metadata>
            <InputClaims>
                <InputClaim ClaimTypeReferenceId="objectId" />
                <!--<InputClaim ClaimTypeReferenceId="anotherValue" PartnerClaimType="claimNameInPayload" />-->
            </InputClaims>
            <UseTechnicalProfileForSessionManagement ReferenceId="SM-Noop" />
        </TechnicalProfile>
    </TechnicalProfiles>
</ClaimsProvider>
```

And there you have it, an `Azure Function` that could funcition as a `Webhook` recipient or a `ServiceUrl` for an `Azure B2C Policy`!

_I realize this discusses the Azure Function more than Azure B2C, but after a little bit of headache and following [this tutorial](https://docs.microsoft.com/en-us/azure/active-directory-b2c/active-directory-b2c-get-started-custom) and [this codebase](https://github.com/Azure-Samples/active-directory-b2c-advanced-policies/tree/master/wingtipgamesb2c) pretty closely, I found Azure B2C to be extremely powerful for handling authentication (just in case you stumble on this page looking for help on B2C and not Azure Functions)..._

</message>
