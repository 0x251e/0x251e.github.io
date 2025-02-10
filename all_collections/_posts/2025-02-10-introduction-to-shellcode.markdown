---
layout: post
title: "Introduction to Shellcode"
date: 2025-02-10
categories: [malware]
---

## Table of Contents
1. [What is Shellcode?](#what-is-shellcode)  
2. [Types of Shellcode](#types-of-shellcode)  
3. [Use Cases of Shellcode](#use-cases-of-shellcode)  
4. [Differences Between Shellcode and Traditional Binaries](#differences-between-shellcode-and-traditional-binaries)  

### What is Shellcode
In the context of cybersecurity, a shellcode refers to a small piece of code that is used for exploiting a vulnerability of a software. At its core, shellcode is a sequence of carefully crafted machine code instructions, typically written in assembly language, that serves a payload in software exploitation. Shellcode is called "shellcode" because it is used to provide an interactive shell session. There are several techniques based on the environment of the target which can be found at [here](http://shell-storm.org/shellcode/index.html) 

### Types of Shellcode
Shellcodes can be categorized into two main types: **local** and **remote**, depending on whether the attack has direct access to the targeted system or being used to exploit over a network 

5. **Local Shellcode** - It is used when an attacker has restricted access to a machine but can still exploit a vulnerability. A common example is a buffer overflow attack, where successful exploitation allows the attack to escalate privileges and gain full control over the system

6. **Remote Shellcode** - It involves attacking a device over a network and uses TCP/IP socket to establish control over the target machine. If the shellcode connects back to the attack's device, it is called a reverse shell. It the attack connects to a port bound by the shellcode, it is referred to as bind shell. 

7. **Download and Execute Shellcode** - Instead of opening a shell, this type of downloads malware onto the target system and executes it. It is often used in drive-by download attacks, where the payload is retrieved, saved and launched automatically

8. **Staged Shellcode** - When payload size is limited, attackers use a staged approach. A small **Stage 1** shellcode is injected and executed, which then loads a larger **Stage 2** payload for full execution.

9. **Egg-Hunt Shellcode** - This is similar to the staged shellcode, it uses a small egg-hunt routine to scan the process's memory space for the actual payload (the egg) and execute it once found

10. **Omelette Shellcode** - A variant of egg-hunt shellcode, where multiple small data blocks (eggs) are injected separately, then combined into a larger omelette before execution. This is useful when only small data chunks can be injected at a time.

### Use Cases of Shellcode 
Shellcode is a crucial part of many exploit or red teaming scenarios where it serves as the exact payload that connects the execution of arbitrary code on a target system to the exploitation of a vulnerability. 

Based on the MITRE ATT&CK technique of [T1055](https://attack.mitre.org/techniques/T1055/), shellcode is used as process injection where malware injects shellcode into legitimate processes to evade detection from AV/EDR. We can take Cobalt Strike as a practical example, which use subtechnique of [DLL injection](https://attack.mitre.org/techniques/T1055/001/) to execute shellcode directly in the memory. 

### Differences between Shellcode and Traditional Binaries
Traditional binaries and shellcode represent its own fundamental approaches to be executed and having its own behavior and constraints. Traditional binaries are structured files that conform to specific formats (like ELF or PE) which containing headers, sections and relocation information that help the operating system loader place and execute them properly. It relies heavily on the OS's loader to resolve any dependencies, establishing memory layouts and handling runtime requirements. 

In contrast, shellcode operates as a raw machine code that execute independently. Shellcode's main objective is to execute very specific action, which usually within the limited and hostile confines of a target process's memory space. Refer the table below as a comparison between shellcode and traditional binaries malware.

| Charateristics         | Traditional Binary                                                                                    | Shellcode                                                                                  |
| :--------------------- | :---------------------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------- |
| Size & Structure       | Larger footprint with additional libraries and functionalities with a defined structure with section  | Compacted size to evade detection with raw machine instruction without any formal headers  |
| Execution Environment  | Can be executed as self-contained programs under OS control                                           | Used by injecting into pre-existing process memory                                         |
| Function Calls         | Uses PLT/GOT for external function resolution                                                         | Must manually resolve function addresses or contain all needed code                        |
| Initialization         | Has standard entry point (\_start)                                                                    | Must initialize its own execution environment                                              |
| Stack Usage            | Normal stack frames with standard conventions                                                         | Avoid stack operations or uses custom frames                                               |
| System Calls           | Use C Library wrappers (glibc or Windows APIs)                                                        | Often make direct system calls                                                             |

Shellcode must operate under severe constraints, with the consideration of the target environment, execution context, and potential security mechanism. This is the fundamental reason of why shellcode development requires specialized knowledge and technique compared to developing traditional executable malware.

Next topic will be "Understanding Assembly for Shellcode Development" which emphasizing basic x86/64 assembly instruction with its calling conventions. Since we have to speak low level, knowledge of registers and stack management will be discuss in depth. 

