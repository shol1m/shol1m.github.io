---
title: "linux cut command"
summary: "A comprehensive guide to the linux cut command"
categories: ["Cheat sheets","Blog",]
# tags: ["P3rf3ctr00tCTF","rev"]
#externalUrl: ""
#showSummary: true
date: 2024-12-28
draft: false
---

# Comprehensive Cheat Sheet for the `cut` Command in Linux

The cut command is a powerful utility for text processing in Linux, enabling the extraction of specific bytes, characters, or fields from lines of input data. It is particularly useful for parsing structured text files such as logs, CSVs, or configuration files. Here's an overview of its features and practical usage:

# Syntax

```bash
cut [OPTION]... [FILE]...
```

# Key Options

|**Option**|**Description**|
|---|---|
|`-b`|Extract specific bytes (e.g., `-b 1-5`).|
|`-c`|Extract specific characters (e.g., `-c 3,5-7`).|
|`-d`|Specify a delimiter (default is tab).|
|`-f`|Specify fields (columns) to extract (requires `-d`).|
|`--complement`|Invert the selection; return all except the specified bytes, characters, or fields.|
|`--output-delimiter`|Specify a custom delimiter for the output (default is the input delimiter).|
|`--help`|Display help information.|
|`--version`|Show version information.|
## Sample log 

I will be using the sample log below for examples.
```bash
203.0.113.42 - - [31/Jul/2023:12:34:56 +0000] "GET /index.php HTTP/1.1" 200 1234 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.132 Safari/537.36"
120.54.86.23 - - [31/Jul/2023:12:34:57 +0000] "GET /contact.php HTTP/1.1" 404 5678 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.138 Safari/537.36"
185.76.230.45 - - [31/Jul/2023:12:34:58 +0000] "GET /about.php HTTP/1.1" 200 9876 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.164 Safari/537.36"
201.39.104.77 - - [31/Jul/2023:12:34:59 +0000] "GET /login.php HTTP/1.1" 200 4321 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.54 Safari/537.36"
112.76.89.56 - - [31/Jul/2023:12:35:00 +0000] "GET /index.php HTTP/1.1" 200 7890 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/95.0.4638.54 Safari/537.36"
211.87.186.35 - - [31/Jul/2023:12:35:01 +0000] "GET /contact.php HTTP/1.1" 200 1234 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.81 Safari/537.36"
156.98.34.12 - - [31/Jul/2023:12:35:02 +0000] "GET /about.php HTTP/1.1" 404 5678 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.85 Safari/537.36"
202.176.73.99 - - [31/Jul/2023:12:35:03 +0000] "GET /login.php HTTP/1.1" 200 9876 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.90 Safari/537.36"
122.65.187.55 - - [31/Jul/2023:12:35:04 +0000] "GET /index.php HTTP/1.1" 200 4321 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.110 Safari/537.36"
77.188.103.244 - - [31/Jul/2023:12:35:05 +0000] "GET /contact.php HTTP/1.1" 200 7890 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.290 Safari/537.36"

```


---


# Basic Usage

## 1. Extract Specific Bytes
- Extract first 5 bytes in each line.

```bash
cut -b 1-5 apache.log
```
### Output
```sh
└─$ cut -b 1-5 apache.log
203.0
120.5
185.7
201.3
112.7
211.8
156.9
202.1
122.6
77.18

```
## 2. Extract Specific Characters
- Extract characters 1 to 13 and from 20 to 30

```bash
cut -c 1-13,20-30 apache.log
```
### Output
```sh
203.0.113.42 1/Jul/2023:
120.54.86.23 1/Jul/2023:
185.76.230.4531/Jul/2023
201.39.104.7731/Jul/2023
112.76.89.56 1/Jul/2023:
211.87.186.3531/Jul/2023
156.98.34.12 1/Jul/2023:
202.176.73.9931/Jul/2023
122.65.187.5531/Jul/2023
77.188.103.24[31/Jul/202
```
## 3. Extract Specific Fields
- Extract methods from the log using a space as the delimiter. i.e 

```bash
cut -d ',' -f 2,4 apache.log
```
### Output
```sh
200
404
200
200
200
200
404
200
200
200
```

## 4. Use Complement
- Extract all characters except for positions 1 to 13

```bash
cut -c 3-5 --complement apache.log
```

### Output
```sh
- - [31/Jul/2023:12:34:56 +0000] "GET /index.php HTTP/1.1" 200 1234 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.132 Safari/537.36"
- - [31/Jul/2023:12:34:57 +0000] "GET /contact.php HTTP/1.1" 404 5678 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.138 Safari/537.36"
 - - [31/Jul/2023:12:34:58 +0000] "GET /about.php HTTP/1.1" 200 9876 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.164 Safari/537.36"
 - - [31/Jul/2023:12:34:59 +0000] "GET /login.php HTTP/1.1" 200 4321 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.54 Safari/537.36"
- - [31/Jul/2023:12:35:00 +0000] "GET /index.php HTTP/1.1" 200 7890 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/95.0.4638.54 Safari/537.36"
 - - [31/Jul/2023:12:35:01 +0000] "GET /contact.php HTTP/1.1" 200 1234 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.81 Safari/537.36"
- - [31/Jul/2023:12:35:02 +0000] "GET /about.php HTTP/1.1" 404 5678 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.85 Safari/537.36"
 - - [31/Jul/2023:12:35:03 +0000] "GET /login.php HTTP/1.1" 200 9876 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.90 Safari/537.36"
 - - [31/Jul/2023:12:35:04 +0000] "GET /index.php HTTP/1.1" 200 4321 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.110 Safari/537.36"
4 - - [31/Jul/2023:12:35:05 +0000] "GET /contact.php HTTP/1.1" 200 7890 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.290 Safari/537.36"
 
```

## 5. Change Output Delimiter
- Extract fields 1 and 9, separating them with `|`.

```bash
cut -d ' ' -f 1,9 --output-delimiter='|' apache.log
```
### Output
```sh
203.0.113.42|200
120.54.86.23|404
185.76.230.45|200
201.39.104.77|200
112.76.89.56|200
211.87.186.35|200
156.98.34.12|404
202.176.73.99|200
122.65.187.55|200
77.188.103.244|200
```

## 6. Combine with Piped Input
- Extract the 4th field from the input strings

```bash
cat apache.log | cut -d ' ' -f 4 
```
### Output
```sh
[31/Jul/2023:12:34:56
[31/Jul/2023:12:34:57
[31/Jul/2023:12:34:58
[31/Jul/2023:12:34:59
[31/Jul/2023:12:35:00
[31/Jul/2023:12:35:01
[31/Jul/2023:12:35:02
[31/Jul/2023:12:35:03
[31/Jul/2023:12:35:04
[31/Jul/2023:12:35:05
```

---

# Advanced Examples

## 1. Extract Last Field from Log File
- Use `awk` to isolate the last field and `cut` to extract a subfield.

```bash
awk '{print $NF}' apache.log | cut -d '/' -f 2
```

### Output
```sh
537.36"
537.36"
537.36"
537.36"
537.36"
537.36"
537.36"
537.36"
537.36"
537.36"
```

## 2. Extract Fields with Multiple Delimiters
- Extract fields 2 and 4, using `:` as the delimiter.
```bash
cat apache.log  | cut -d '/' -f 2,4
```

### Output
```sh
Jul/index.php HTTP
Jul/contact.php HTTP
Jul/about.php HTTP
Jul/login.php HTTP
Jul/index.php HTTP
Jul/contact.php HTTP
Jul/about.php HTTP
Jul/login.php HTTP
Jul/index.php HTTP
Jul/contact.php HTTP

```

## 3. Filter and Extract Specific Data
- Extract the 1st, 6th and 7th fields from log lines containing `404` status code

```bash
grep '404' apache.log | cut -d ' ' -f 1,6,7 --output-delimiter=' '
```
### Output
```js
120.54.86.23 "GET /contact.php
156.98.34.12 "GET /about.php
189.76.230.44 "GET /about.php
178.53.64.100 "GET /contact.php
178.53.64.100 "GET /login.php
203.78.122.88 "GET /contact.php
99.76.122.65 "GET /login.php
203.64.78.90 "GET /about.php
128.45.76.66 "GET /contact.php
176.145.201.99 "GET /login.php
128.45.76.66 "GET /contact.php
176.145.201.99 "GET /about.php
76.89.54.221 "GET /contact.php
176.145.201.99 "GET /contact.php
76.89.54.221 "GET /index.php
203.64.78.90 "GET /login.php
128.45.76.66 "GET /about.php
203.64.78.90 "GET /about.php
128.45.76.66 "GET /contact.php
```

---

# Working with Delimiters

## Common Delimiters

| **Delimiter**   | **Usage Example**               |
| --------------- | ------------------------------- |
| `,` (comma)     | For CSV files.                  |
| `:` (colon)     | For `/etc/passwd` files.        |
| `;` (semicolon) | For semicolon-separated values. |
| `\t` (tab)      | Default for `cut`.              |
| ` ` space       | Any space separated values      |

---

# Combining with Other Tools

## 1. Combining `cut` with `sort`
- Extract the 1st field , count unique occurrences and sort them

```bash
cut -d ' ' -f 1 apache.log | uniq -c | sort -n -r
```
### Output
```bash
      2 145.76.33.201
      1 99.76.122.65
      1 77.188.103.244
      1 76.89.54.221
      1 76.89.54.221
      1 76.89.54.221
      1 76.89.54.221
      1 76.89.54.221
      1 76.89.54.221
      1 221.90.64.76
```
## 2. Combining `cut` with `awk`
- Use `awk` to isolate a field and `cut` to extract specific characters. 

```bash
awk '{print $2}' apache.log | cut -c 1-5
```
### Output
```bash
GET
GET
GET
GET
GET
GET
GET
GET
GET
GET
```
## 3. Combining `cut` with `grep`
- Filter logs for status code 200 and extract the first field.
- 
```bash
grep '200' apache.log | cut -d ' ' -f 1
```

### Output
```bash
203.0.113.42
185.76.230.45
201.39.104.77
112.76.89.56
211.87.186.35
202.176.73.99
122.65.187.55
77.188.103.244
153.47.106.221
200.89.134.22
```
## 4. Combining `cut` with `paste`
- Combine two extracted fields into one output.

```bash
cut -d ' ' -f 1 apache.log | paste - <(cut -d ' ' -f 7 apache.log)
```
### Output
```bash
203.0.113.42	/index.php
120.54.86.23	/contact.php
185.76.230.45	/about.php
201.39.104.77	/login.php
112.76.89.56	/index.php
211.87.186.35	/contact.php
156.98.34.12	/about.php
202.176.73.99	/login.php
122.65.187.55	/index.php
77.188.103.244	/contact.php
```

---

# References

1. Linux `cut` Command Manual
	Run `man cut` for the tool manual.
2. Linux hand book
	https://linuxhandbook.com/cut-command/