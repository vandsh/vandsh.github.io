---
layout: post
title: "Powershell WebClient with Timeout"
category: Powershell
tags: Sitecore CI-CD Powershell
---

Every once and a while, in CI/CD you need to just simply invoke a webpage. Jeremy Davis reminded me of [this struggle](https://jermdavis.wordpress.com/2018/01/08/issues-with-invoke-webrequest-and-ie-on-servers/) when sometimes things are locked down pretty tight, or things are just not cooperating when trying to use the `Invoke-WebRequest` commandlet even after trying the `-UseBasicParsing` flag.

[Curl](https://curl.haxx.se/download.html) (the binary, not the `Invoke-WebRequest` alias) is always a very solid option, fairly system agnositc and quite optimistic when you want it to be (ie: `-k, --insecure` if you are hitting a localhost url on an IIS site with a domain specific SSL).

But like Jeremy pointed out, `System.Net.WebClient` is a great alternative to `Invoke-WebRequest`, especially if you don't feel like adding the `curl.exe` to your solution.  The only issue I have ran into with `WebClient` is if you need to define a timeout other than the default, there is no easy way.  Below is a snippet I have used a couple times in the CI/CD process (but to be honest have always ended up using `curl`); it uses inline C# code to create a class that inherits `WebClient` but overrides the `GetWebRequest` method and forces the timeout: 


```powershell
$timeoutWebclientCode = @"
using System.Net;

public class TimeoutWebClient : WebClient
{
    public int TimeoutSeconds;

    protected override WebRequest GetWebRequest(System.Uri address)
    {
        WebRequest request = base.GetWebRequest(address);
        if (request != null)
        {
           request.Timeout = TimeoutSeconds * 1000;
        }
        return request;
    }

    public TimeoutWebClient()
    {
        TimeoutSeconds = 300; // Timeout value by default
    }
}
"@;

Add-Type -TypeDefinition $timeoutWebclientCode -Language CSharp
$webClient = New-Object TimeoutWebClient;
$webClient.TimeoutSeconds = 900;
#to add credentials
#$webclient.Credentials = New-Object System.Net.NetworkCredential($username,$password)
$downloaded = $webClient.downloadString('http://via.placeholder.com/350x150')
Add-Content -path test.jpg -value $downloaded
```

Also, for fun I added in a line to show username/password usage just in case whatever page you are trying to invoke is behind a auth prompt.

Hope this helps!
