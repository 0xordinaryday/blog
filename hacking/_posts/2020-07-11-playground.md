---
layout: post
title:  "THM - Python Playground"
date:   2020-07-10 18:00:00 +1000
category: hacking
---

## Introduction
*Be creative! Jump in and grab those flags! They can all be found in the usual places.*  

This is a hard rated box, and *so far* I haven't completed it fully.

## Ports
nmap says we've got 22 (SSH) and 80 (HTTP) only, and TTL says it's Linux. A detail scan says the webserver is running on Node.js, but that's pretty much it.

## Webserver
The frontpage gives us some information:

*Introducing the new era code sandbox; python playground! Normally, code playgrounds that execute code serverside are easy ways for hackers to access a system. Not anymore! With our new, foolproof blacklist, no one can break into our servers, and we can all enjoy the convenience of running our python code on the cloud!*

There are also links to signup.html and login.html, but these pages are not working. 

We can fuzz: 
>root@kali:/opt/tryhackme/python_playground# wfuzz --hc 404 -u http://10.10.76.200/FUZZ.html -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt

And that brings us to /admin.html, which we might have guessed at from this:

*Sorry, but due to some recent security issues, only admins can use the site right now. Don't worry, the developers will fix it soon :)*

## Admin
/admin.html has a login form, and when we look at the source code there is some Javascript that does a custom hash on the input password. If it matches a hardcoded hash you will be directed to /super-secret-admin-testing-panel.html, so we can skip the messing about and go straight there.

http://10.10.76.200/super-secret-admin-testing-panel.html

## Playground/Testing Panel
This is a simple page, with an input and output box. We put Python code in the input, and read the output.

Running print with and without parentheses quickly establishes that we are using Python 3, and we can read files. e.g.:

>f = open("/etc/passwd")  
print(f.read())

We can even read /etc/shadow, but there is no root hash and apparently no extra users; this is a little odd.

Trying to do any kind of import (so we can run os.system and get a reverse shell for example) produces the warning:
**Security threat detected!**
So that presents a challenge.

However, even with simple guesswork we can get the first flag with this file reading method - it's in **/root/flag1.txt**.

## Workaround
We can get partway around the import blacklist with code like this:

{% highlight python %} 
s = '__imp'  
s += 'ort__'  
f = globals()['__builtins__'].__dict__[s]  
z = f("os")  
result=[]  
for root, dirs, files in z.walk("/"):  
  for name in files:  
    if ".txt" in name:  
      result.append(z.path.join(root, name))  
print(result)  
{% endhighlight %}

But despite quite a bit of enumeration, nothing turns up.

## Back to the future
Remember that Javascript that checked the hash? Yeah? Well, we can go to work on it. We can reverse it, and derive the password **spaghetti1245**. This turns out to be the SSH password for user 'connor'.

## Connor
So in fact there is an additional user (connor) that wouldn't show up in /etc/passwd when we read it (or a **version** of it) with Python. And he has **~/flag2.txt**. So we're now two-thirds of the way there.

## TBC....
As mentioned, I haven't actually finished this yet. I'll keep plugging away at it tomorrow.

