---
layout: post
title: "Azure API Management: Webhook Signature Validation"
category: Azure
tags: Azure APIM Webhook
---

Recently working on a handful of [Azure API Management](https://azure.microsoft.com/en-us/services/api-management/) efforts, we came across an interesting one: an operation being triggered from a [GitHub](https://developer.github.com/webhooks/creating/) event, triggered as a webhook. My first instinct was to look and see if anything similar to the [`validate-jwt`](https://docs.microsoft.com/en-us/azure/api-management/api-management-access-restriction-policies#ValidateJWT) token was available as a policy, but alas, it was not. So, instead I feel back to what I know best: code.

Ultimately in order to validate this webhook, we needed to know the following:
 - The header (or query parameter) that the signature was being stuffed in
 - The secret that was being used to create the signature

If we know that, we can 
 - create the signature, given the payload of the body 
 - can compare the signatures 
 - pass or fail the request based on comparison

## Webhook Signature Verification Policy 

```xml
<policies>
    <inbound>
        <base />
        <!-- Grab the signature from the header and stuff it in a variable -->
        <set-variable name="requestSignature" value="@{
            return context.Request.Headers["X-Hub-Signature"][0].Replace("sha1=","");
            }" />
        <!-- Take the payload and the Secret (which should be in a KeyVault or at least a secret Named value) 
            and generate a hash to compare to the original.-->
        <set-variable name="compareHash" value="@{
            byte[] keyByte = System.Text.ASCIIEncoding.ASCII.GetBytes("NotASecret!");
            byte[] messageBytes = System.Text.ASCIIEncoding.ASCII.GetBytes(context.Request.Body.As<string>());
            var hash = new HMACSHA1(keyByte);
            var hashBytes = hash.ComputeHash(messageBytes);
            StringBuilder hex = new StringBuilder(hashBytes.Length * 2);
            foreach (byte b in hashBytes){
                hex.AppendFormat("{0:x2}", b);
            }
            return (string)hex.ToString();
        }" />
        <choose>
            <!-- Cast as strings and comapare the signature/hashes -->
            <when condition="@((string)context.Variables["requestSignature"] != (string)context.Variables["compareHash"])">
                <!-- The return response stops all further processing and in this case returns a 401 -->
                <return-response>
                    <set-status code="401" reason="Not Authorized" />
                    <!-- Uncomment if you need to debug -->
                    <!-- <set-body template="none">@{
                        return context.Variables["requestSignature"] + "::" + context.Variables["compareHash"];
                    }</set-body> -->
                </return-response>
            </when>
            <otherwise>
                <set-backend-service base-url="http://yourbackend.service/notreal/" />
            </otherwise>
        </choose>
    </inbound>
    <!-- The backend and outbound nodes, ie: the rest of the stuff -->
</policies>
```

Thru the variables and then the `choose/when` condition, we can short circuit any invalid signatures before going any further and in the event it _is valid_, the body of the original request will continue to the `backend-service` url for further processing. In this case it was a repository that handled resource orchestration on an internal network (Ansible, Puppet or Chef I don't recall which). 

This is just one of many ways that `Azure APIM` can prove it's power and flexibility!
