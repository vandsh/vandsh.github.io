---
layout: post
title: "Powershell Send Email with Attachment"
category: Powershell
tags: Powershell SMTP CI/CD
---

Whether it's to send an email as part of a build or release workflow, as part of a scheduled task (ie: a report of some kind) or you are simply just trying to test to see if your SMTP service is working (the reason I wrote this, SMTP service was refusing attachments...) sometimes its helpful to have a `Powershell` script to do it for you.  Below is a script that can call an authenticated SMTP service and send an email with an attachment:

### Powershell Send Email with Attachment
```powershell
$Username = "user@yoursmtpsrvc.com";
$Password= "whatami";

function Send-ToEmail([string]$email){

    $message = new-object Net.Mail.MailMessage;
    $message.From = $Username;
    $message.To.Add($email);
    $message.Subject = "Testing Email Server";
    $message.Body = "Please Ignore for now";

    write-host "attaching"
    $file = "C:\Users\You\tst.txt"
    $att = new-object Net.Mail.Attachment($file)
    $message.Attachments.Add($file)
	
    write-host "new smtp"
    $smtp = new-object Net.Mail.SmtpClient("yoursmtpsrvc.com", "25"); 
    $smtp.EnableSSL = $false;
    $smtp.Credentials = New-Object System.Net.NetworkCredential($Username, $Password);

    write-host "new send"
    $smtp.send($message);
    write-host "pre dispose"
    $att.Dispose()
    write-host "**Mail Sent" ; 
 }
 
Send-ToEmail  -email "youractual@mail.com";

exit 1
```

This is a script I have used for a while and honestly may be an aggregation of a few old posts/snippets out in the interwebs (my guess is [this one](https://stackoverflow.com/questions/36355271/how-to-send-email-with-powershell)). Anyways, it's here for your reference.

Hope this helps!
