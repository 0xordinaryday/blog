---
layout: post
title:  "Defender, you're starting to sh%t me"
date:   2021-03-10 20:00:00 +1000
category: meta
---

## WHAT THE ACTUAL F*** WINDOWS?
I wrote up my post for Optimum, and Defender suddenly decides to nuke a post about Cute from September last year out of nowhere. The *Windows Security* app won't restore it. I try with an administrator command prompt:

{% highlight shell %}
C:\Program Files\Windows Defender>MpCmdRun.exe -restore -listall
The following items are quarantined:

ThreatName = Backdoor:Win32/Dirtelti!ml
      file:C:\Users\David\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu18.04onWindows_79rhkp1fndgsc\LocalState\rootfs\opt\website\minima\hacking\_posts\2020-09-25-cute.md quarantined at 3/10/2021 9:59:40 AM (UTC)

C:\Program Files\Windows Defender>MpCmdRun.exe -restore -all
Restoring the following quarantined items:

ThreatName = Backdoor:Win32/Dirtelti!ml
   ERROR - file:C:\Users\David\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu18.04onWindows_79rhkp1fndgsc\LocalState\rootfs\opt\website\minima\hacking\_posts\2020-09-25-cute.md quarantined at 3/10/2021 9:59:40 AM (UTC) cannot be restored. Error code = 0x80508014
CmdTool: Failed with hr = 0x80508014. Check C:\Users\David\AppData\Local\Temp\MpCmdRun.log for more information
{% endhighlight %}

So not only is this post suddenly a problem after **6 F&&&ING MONTHS**, it **cannot be restored**. All over the world exchange servers are getting popped like nobodies business and I can't write a blog post. GET F&&&ED. 
