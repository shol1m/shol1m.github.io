---
title: "Pores"
summary: "P3rf3ctr00t CTF rev chalenge"
categories: ["P3rf3ctr00tCTF","Blog",]
tags: ["P3rf3ctr00tCTF"]
#externalUrl: ""
#showSummary: true
date: 2024-11-23
draft: false
---


# **Challenge Description**

- **Name:** Pores
- **Category:** rev
- **Points:** 350
- **Challenge Description:**  
    P3rf3ctr00t is locked, hidden in the depths of a binary, waiting for a hero to rewrite its fate.

---

# FIle inspection
First i started by determining the file type. 
The file is a linux 64-bit elf file.
```shell
file poresssss                                                                                                                          
poresssss: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=d0a124b3a218c26eb707783fa5f3dc7f0763de88, for GNU/Linux 3.2.0, not stripped
```

# Analysis using Ghidra
For static analysis, I launched `ghidra` and imported the binary.
The binary contains two functions, main() and printFlag().
![man_function](https://gist.github.com/user-attachments/assets/51636add-5dab-4cf5-8d10-0f3612734614) ![printFlag_function](https://gist.github.com/user-attachments/assets/04816230-819d-4aa9-ba12-db1f4ea26e0a)
the main function does nothing while the printFlag() take two parameters and calculates the flag.

## Analysis with pwndbg
While Ghidra revealed the high-level structure of the binary, it did not show the critical conditional jump (`jne`) preventing `printFlag` from being called. This highlights the importance of dynamic analysis tools like `pwndbg` for a deeper understanding.
I  fired up `pwndbg` with the command `pwndbg poresssss`. Then disassembled main function to check whats going on there.
![disass_main1](https://gist.github.com/user-attachments/assets/68077176-085c-4f5e-945f-a810566f06bd)
Well there is more than i could see with ghidra. There is a condition that prevents printflag from being called.
There is a comparison between `0` and `1` , if the two numbers are not equal, the program jumps to `main +41` and printFlag is not called.
The condition (`cmp DWORD PTR [rbp-0x4], 0x1`) always evaluates as `not equal` because `rbp-0x4` is explicitly set to `0`.
## Patching the program
To make sure that the program calls the printFlag() function, a patch can be done. Instead of having an operation of `jne` , a nop (no operation can be placed instead of `jne`. this ensures that the instruction between `main +21` to `main +36` are executed. 

To patch the program we have to replace `jne` with `nop`. 
First set a break point at main with the comand `break *main` then run the binary. 
Disasseble main to check the adresses at runtime
```bash
pwndbg> disass main
Dump of assembler code for function main:
=> 0x0000555555555298 <+0>:	push   rbp
   0x0000555555555299 <+1>:	mov    rbp,rsp
   0x000055555555529c <+4>:	sub    rsp,0x10
   0x00005555555552a0 <+8>:	mov    DWORD PTR [rbp-0x4],0x0
   0x00005555555552a7 <+15>:	cmp    DWORD PTR [rbp-0x4],0x1
   0x00005555555552ab <+19>:	jne    0x5555555552c1 <main+41>
   0x00005555555552ad <+21>:	mov    esi,0x8
   0x00005555555552b2 <+26>:	lea    rax,[rip+0x2d87]        # 0x555555558040 <flag>
   0x00005555555552b9 <+33>:	mov    rdi,rax
   0x00005555555552bc <+36>:	call   0x555555555159 <printFlag>
   0x00005555555552c1 <+41>:	mov    eax,0x0
   0x00005555555552c6 <+46>:	leave
   0x00005555555552c7 <+47>:	ret
End of assembler dump.
pwndbg> 
```
This can be done by changing the address `0x00005555555552ab` and `0x00005555555552ac` to `0x90` which is nop. 
Two addresses are modified since `jne` used two bytes of memory for that instruction. Both bytes need to be replaced to avoid leaving stray bytes from the original instruction, which could corrupt the execution flow and cause a crash.
```bash
set {char} 0x00005555555552ab = 0x90 
set {char} 0x00005555555552ab = 0x90
```
To verify, disasseble main again. This is necessary to ensure the patch is correctly applied before continuing
![disass_main2](https://gist.github.com/user-attachments/assets/a5f666e3-09e8-4cbc-bb62-3029f6fe889c)

The continue the program with the command `continue`
```bash
Continuing.
r00t{p4tch_th3_bin_and_h4ve_fun}
[Inferior 1 (process 12780) exited normally]
```

# Key Takeaways

1. **Dynamic Analysis Complements Static Analysis**: While tools like Ghidra provide a high-level view of a binary, dynamic debugging with `pwndbg` uncovers critical details like conditional jumps and runtime behavior, making it essential for identifying patching opportunities.

2. **Patching with NOP Simplifies Control Flow**: Replacing conditional jumps (`jne`) with `NOP` is an effective way to bypass checks, allowing the program to execute blocked code segments like `printFlag()`.