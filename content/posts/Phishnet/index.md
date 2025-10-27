
---
title: "Sherlock: PhishNet (HTB Challenge)"
summary: "An in-depth forensic investigation of a phishing email from the HTB Sherlocks series. We analyze email headers, SPF validation, and a disguised malicious attachment used in a spearphishing attack."
categories: ["HackTheBox", "Sherlocks", "Email Analysis", "Blog"]
tags: ["HTB", "Sherlock", "Phishing", "SOC"]
date: 2025-10-02
draft: false

---

# **Challenge Description**

- **Name:** PhishNet
- **Category:** SOC
- **Challenge Scenario:**  
    An accounting team receives an urgent payment request from a known vendor. The email appears legitimate but contains a suspicious link and a .zip attachment hiding malware. Your task is to analyze the email headers, and uncover the attacker's scheme.

---
# Sherlock Investigation: PhishNet

## Scenario

An accounting team received an urgent payment request from a familiar vendor. The email looked legitimate but contained a **suspicious link** and a **ZIP attachment** that was later identified as **malicious**.  
As a threat analyst, your task was to **analyze the email headers**, **examine the attachment**, and **uncover the attacker’s tactics**.

---

## Artifacts

We were provided with one artifact:

- `email.eml` — the raw phishing email containing headers, HTML body, and an attachment.
    

---

## Task 1 – Originating IP Address

The sender’s IP address is typically found in the header field `X-Sender-IP`.

```sh
❯ cat email.eml | grep X-Sender-IP 
X-Sender-IP: 45.67.89.10
```

**Answer:** `45.67.89.10`

This is the originating IP address of the sender.

---

## Task 2 – Mail Server That Relayed the Email

The “Received” headers reveal the route the email took from sender to recipient.

```sh
❯ cat email.eml | grep Received 
Received-SPF: Pass (protection.outlook.com: domain of business-finance.com designates 45.67.89.10 as permitted sender) 
Received: from mail.business-finance.com ([203.0.113.25]) 	by mail.target.com (Postfix) with ESMTP id ABC123; 	Mon, 26 Feb 2025 10:15:00 +0000 (UTC) 
Received: from relay.business-finance.com ([198.51.100.45]) 	by mail.business-finance.com with ESMTP id DEF456; 	Mon, 26 Feb 2025 10:10:00 +0000 (UTC) 
Received: from finance@business-finance.com ([198.51.100.75]) 	by relay.business-finance.com with ESMTP id GHI789; 	Mon, 26 Feb 2025 10:05:00 +0000 (UTC)
```

The **last relay** before reaching the victim was:

**Answer:** `mail.business-finance.com (203.0.113.25)`

---

## Task 3 – Sender’s Email Address

```sh
❯ cat email.eml | grep From 
X-Envelope-From: finance@business-finance.com 
From: "Finance Dept" <finance@business-finance.com>
```

**Answer:** `finance@business-finance.com`

---

## Task 4 – Reply-To Address

```sh
❯ cat email.eml | grep Reply-To 
Reply-To: <support@business-finance.com>
```

**Answer:** `support@business-finance.com`

This can indicate **email spoofing** — attackers often set a separate “Reply-To” to receive responses on a different mailbox.

---

## Task 5 – SPF (Sender Policy Framework) Result

```sh
❯ cat email.eml | grep SPF 
Received-SPF: Pass (protection.outlook.com: domain of business-finance.com designates 45.67.89.10 as permitted sender)
```

**Answer:** `Pass`

This means the sending IP (`45.67.89.10`) is authorized to send mail on behalf of `business-finance.com`.  
However, **a passing SPF check does not confirm legitimacy**, as the domain itself could have been compromised.

---

## Task 6 – Domain in the Phishing URL

The HTML content of the email reveals the phishing link:

```sh
<a href="https://secure.business-finance.com/invoice/details/view/INV2025-0987/payment">Download Invoice</a>
```

**Answer:** `secure.business-finance.com`

Attackers often use **subdomains** that appear trustworthy to trick recipients.

---

## Task 7 – Fake Company Name

From the email footer:

> Best regards,  
> Finance Department  
> **Business Finance Ltd.**

**Answer:** `Business Finance Ltd.`

---

## Task 8 – Attachment Name

From the MIME section of the email:

```sh
Content-Type: application/zip; name="Invoice_2025_Payment.zip" Content-Disposition: attachment; filename="Invoice_2025_Payment.zip"
```

**Answer:** `Invoice_2025_Payment.zip`

---

## Task 9 – SHA-256 Hash of the Attachment

After extracting the attachment using `munpack`:

```sh
❯ munpack email.eml Invoice_2025_Payment.zip (application/zip) ❯ sha256sum Invoice_2025_Payment.zip 8379c41239e9af845b2ab6c27a7509ae8804d7d73e455c800a551b22ba25bb4a  Invoice_2025_Payment.zip
```
**Answer:**  
`8379c41239e9af845b2ab6c27a7509ae8804d7d73e455c800a551b22ba25bb4a`

---

## Task 10 – Malicious File Contained Within the ZIP

`unzip` and `zipinfo` failed due to an invalid archive header, but `7z` managed to read partial data:

```sh
❯ 7z l Invoice_2025_Payment.zip

7-Zip 23.01 (x64) : Copyright (c) 1999-2023 Igor Pavlov : 2023-06-20
 64-bit locale=en_US.UTF-8 Threads:4 OPEN_MAX:1024

Scanning the drive for archives:
1 file, 75 bytes (1 KiB)

Listing archive: Invoice_2025_Payment.zip

--
Path = Invoice_2025_Payment.zip
Type = zip
ERRORS:
Unexpected end of archive
Physical Size = 75
Characteristics = Local

   Date      Time    Attr         Size   Compressed  Name
------------------- ----- ------------ ------------  ------------------------
2025-02-26 15:56:48 .....      1690811      1249907  invoice_document.pdf.bat
------------------- ----- ------------ ------------  ------------------------
2025-02-26 15:56:48            1690811      1249907  1 files

Errors: 1

```

**Answer:** `invoice_document.pdf.bat`

This “double extension” technique (`.pdf.bat`) is a **common obfuscation** used to trick users into thinking the file is a safe PDF when it’s actually an **executable batch script**.

---

## Task 11 – Associated MITRE ATT&CK Technique

This campaign uses a **malicious attachment delivered via phishing email**.

**Answer:**  
Technique: Phishing: Spearphishing Attachment  
**MITRE ID:** [T1566.001](https://attack.mitre.org/techniques/T1566/001/)

## Analysis Summary

|Category|Detail|
|---|---|
|**Sender Email**|finance@business-finance.com|
|**Originating IP**|45.67.89.10|
|**Relay Server**|mail.business-finance.com (203.0.113.25)|
|**Reply-To**|[support@business-finance.com](mailto:support@business-finance.com)|
|**SPF Result**|Pass|
|**Phishing Domain**|secure.business-finance.com|
|**Fake Company**|Business Finance Ltd.|
|**Attachment Name**|Invoice_2025_Payment.zip|
|**Contained File**|invoice_document.pdf.bat|
|**SHA-256 Hash**|8379c41239e9af845b2ab6c27a7509ae8804d7d73e455c800a551b22ba25bb4a|
|**MITRE Technique**|T1566.001 – Phishing: Attachment|
