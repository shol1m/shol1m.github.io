
---
title: "HookFlare"
summary: "A Sherlocks DFIR Challenge."
categories: ["HackTheBox", "Sherlocks", "Email Analysis", "Blog"]
tags: ["HTB", "Sherlock", "Phishing", "DFIR"]
date: 2025-10-04
draft: false

---

## **Challenge Description**

- **Name:** HookFlare
- **Category:** DFIR
- **Challenge Description:**  
    A S1rBank client reported unauthorized transactions. The victim received an SMS urging a banking app update via a link, which installed a dormant app mimicking the bank’s official version. Once activated, it stole credentials, bypassed 2FA via SMS interception, and exfiltrated data. As a DFIR specialist, analyze the Android disk image to uncover the malware’s operation, reconstruct the attack chain, and identify critical IoCs.

---

# Hack The Box Sherlocks Writeup - HookFlare
## Task 1 Provide the UTC timestamp of the phishing SMS.

## Task 2 Provide the UTC timestamp marking the start of the malicious application download.

## Task 3 Provide the package name of the malicious application.

## Task 4 Provide the number of runtime permissions granted to the malicious application.

## Task 5 Provide the last access timestamp for the read sms permission used by the malicious application.

## Task 6 Provide the URL used by the malware for data exfiltration.

## Task 7 The malicious application checks if the server is live before sending data. Provide the HTTP method used for this check.

## Task 8 If the primary server is unavailable, the malicious application redirects data exfiltration to an alternate URL. Identify and provide the alternate URL.

## Task 9 The malicious application encrypts data before sending it to the server. Provide the encryption key used.

## Task 10 Credit card information was stolen. What was the second line in the exfiltrated payment information?