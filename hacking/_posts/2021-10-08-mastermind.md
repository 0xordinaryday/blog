---
layout: post
title:  "THM: Masterminds"
date:   2021-10-06 21:00:00 +1000
category: hacking
---

Long time no post. I have being doing some stuff, but nothing too enthralling or particularly worth recording. With a lack of new content on VulnHub and *interesting* new content on THM, I've been having a bit of a look at Root-Me. This however, is about the new THM room [Masterminds](https://tryhackme.com/room/mastermindsxlq).

I'm not going through the whole thing there. It's interesting (so check it out), but basically it's using [Brim](https://www.brimdata.io/) to analyze some packet captures.

It's set up with an Ubuntu 'attackbox' you're supposed to use, with Brim already installed. However, the attackbox sucks so let's not use that. We can download Brim and try to install it:

{% highlight shell %}
┌──(root💀kali)-[/tmp]
└─# wget https://github.com/brimdata/brim/releases/download/v0.26.0/Brim-0.26.0.deb
--2021-10-08 05:28:46--  https://github.com/brimdata/brim/releases/download/v0.26.0/Brim-0.26.0.deb
Resolving github.com (github.com)... 52.64.108.95
Connecting to github.com (github.com)|52.64.108.95|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://github-releases.githubusercontent.com/144212075/5a85b0a0-0554-4243-b093-4f0fb73b2684?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20211008%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20211008T092848Z&X-Amz-Expires=300&X-Amz-Signature=ecc2b8cebf1dd0c2c5bc744373e1e786b15689e38ed1bb45318a5172498729eb&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=144212075&response-content-disposition=attachment%3B%20filename%3DBrim-0.26.0.deb&response-content-type=application%2Foctet-stream [following]
--2021-10-08 05:28:46--  https://github-releases.githubusercontent.com/144212075/5a85b0a0-0554-4243-b093-4f0fb73b2684?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20211008%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20211008T092848Z&X-Amz-Expires=300&X-Amz-Signature=ecc2b8cebf1dd0c2c5bc744373e1e786b15689e38ed1bb45318a5172498729eb&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=144212075&response-content-disposition=attachment%3B%20filename%3DBrim-0.26.0.deb&response-content-type=application%2Foctet-stream
Resolving github-releases.githubusercontent.com (github-releases.githubusercontent.com)... 185.199.111.154, 185.199.110.154, 185.199.109.154, ...
Connecting to github-releases.githubusercontent.com (github-releases.githubusercontent.com)|185.199.111.154|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 190513772 (182M) [application/octet-stream]
Saving to: ‘Brim-0.26.0.deb’

Brim-0.26.0.deb                   100%[============================================================>] 181.69M  5.68MB/s    in 34s     

2021-10-08 05:29:20 (5.37 MB/s) - ‘Brim-0.26.0.deb’ saved [190513772/190513772]

                                                                                                                                       
┌──(root💀kali)-[/tmp]
└─# dpkg -i Brim-0.26.0.deb                                                        
Selecting previously unselected package brim.
(Reading database ... 348228 files and directories currently installed.)
Preparing to unpack Brim-0.26.0.deb ...
Unpacking brim (0.26.0) ...
dpkg: dependency problems prevent configuration of brim:
 brim depends on libappindicator3-1; however:
  Package libappindicator3-1 is not installed.

dpkg: error processing package brim (--install):
 dependency problems - leaving unconfigured
Processing triggers for mailcap (3.70) ...
Processing triggers for desktop-file-utils (0.26-1) ...
Processing triggers for hicolor-icon-theme (0.17-2) ...
Errors were encountered while processing:
 brim
{% endhighlight %}                             

It doesn't work, because it wants **libappindicator3-1**, and that's not in the Kali repos. We need to add these lines:

{% highlight shell %}
deb http://deb.debian.org/debian buster-updates main
deb http://deb.debian.org/debian buster main contrib
{% endhighlight %}   

to **/etc/apt/sources.list**, and then update and install:

{% highlight shell %}
┌──(root💀kali)-[/tmp]
└─# apt install libappindicator3-1                                                         
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
You might want to run 'apt --fix-broken install' to correct these.
The following packages have unmet dependencies:
 libappindicator3-1 : Depends: libindicator3-7 but it is not going to be installed
E: Unmet dependencies. Try 'apt --fix-broken install' with no packages (or specify a solution).                                                                                                                                       
┌──(root💀kali)-[/tmp]
└─# apt --fix-broken install                                                               
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Correcting dependencies... Done
The following additional packages will be installed:
  libappindicator3-1 libindicator3-7
The following NEW packages will be installed:
  libappindicator3-1 libindicator3-7
0 upgraded, 2 newly installed, 0 to remove and 11 not upgraded.
1 not fully installed or removed.
Need to get 106 kB of archives.
After this operation, 217 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 http://deb.debian.org/debian buster/main amd64 libindicator3-7 amd64 0.5.0-4 [52.9 kB]
Get:2 http://deb.debian.org/debian buster/main amd64 libappindicator3-1 amd64 0.4.92-7 [53.5 kB]
Fetched 106 kB in 0s (1,121 kB/s)        
Selecting previously unselected package libindicator3-7:amd64.
(Reading database ... 349120 files and directories currently installed.)
Preparing to unpack .../libindicator3-7_0.5.0-4_amd64.deb ...
Unpacking libindicator3-7:amd64 (0.5.0-4) ...
Selecting previously unselected package libappindicator3-1:amd64.
Preparing to unpack .../libappindicator3-1_0.4.92-7_amd64.deb ...
Unpacking libappindicator3-1:amd64 (0.4.92-7) ...
Setting up libindicator3-7:amd64 (0.5.0-4) ...
Setting up libappindicator3-1:amd64 (0.4.92-7) ...
Setting up brim (0.26.0) ...
Processing triggers for libc-bin (2.32-4) ...
needrestart is being skipped since dpkg has failed
                                                                                                                                       
┌──(root💀kali)-[/tmp]
└─# dpkg -i Brim-0.26.0.deb                     
(Reading database ... 349135 files and directories currently installed.)
Preparing to unpack Brim-0.26.0.deb ...
Unpacking brim (0.26.0) over (0.26.0) ...
Setting up brim (0.26.0) ...
Processing triggers for mailcap (3.70) ...
Processing triggers for desktop-file-utils (0.26-1) ...
Processing triggers for hicolor-icon-theme (0.17-2) ...
{% endhighlight %}  

And we're good. Launching Brim from the menu is a fail, because I'm running as root:

{% highlight shell %}
┌──(root💀kali)-[/opt/thm/mastermindsxlq]
└─# brim             
[3164:1008/053956.159960:FATAL:electron_main_delegate.cc(253)] Running as root without --no-sandbox is not supported. See https://crbug.com/638180.
zsh: trace trap  brim
{% endhighlight %} 
						
So we have to run it with --no-sandbox

{% highlight shell %}
┌──(root💀kali)-[/opt/thm/mastermindsxlq]
└─# brim --no-sandbox                                                                     
05:40:05.093 › app paths: getAppPath=/opt/Brim/resources/app.asar userData=/root/.config/Brim logs=/root/.config/Brim/logs
05:40:05.229 › zed core log /root/.config/Brim/logs/zlake.log
# etc 
{% endhighlight %} 

And we're golden. We can get the IP of the attackbox and start a python server there, and download the pcaps once we've connected via OpenVPN, e.g. 

{% highlight shell %}
┌──(root💀kali)-[/opt/thm/mastermindsxlq]
└─# wget http://10.10.240.191:8000/Infection1.pcap                                                                          
--2021-10-08 05:38:33--  http://10.10.240.191:8000/Infection1.pcap
Connecting to 10.10.240.191:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 129529 (126K) [application/vnd.tcpdump.pcap]
Saving to: ‘Infection1.pcap’

Infection1.pcap                   100%[============================================================>] 126.49K   126KB/s    in 1.0s    

2021-10-08 05:38:35 (126 KB/s) - ‘Infection1.pcap’ saved [129529/129529]
{% endhighlight %} 

From there it's a matter of importing the pcap into Brim and working through the challenge. I'd never seen Brim before but it's pleasingly inutitive to use, so well done to the creators.

Oh, this isn't a writeup of the answers.
