---
layout: default
title: Day 12 - After Hours
---

- [./ctf-writeups](../../../../) / [TryHackMe](../../../) / [Events](../../) / [Byte Lotus Hotel](../) / [Day 11 - Infinity Pool](../11_Infinity_Pool)

# Day 12 - After Hours

[**Try it yourself**](https://tryhackme.com/room/hh-afterhours-b090d1f0)  
**Difficulty:** Medium  
**Category:** Digital forensics / WMI persistence / Reverse engineering

**Summary:** This room focuses on advanced digital forensics and malware analysis. We investigate a Windows Management Instrumentation (WMI) repository to uncover a fileless persistence mechanism. After extracting and deobfuscating a hidden PowerShell stager, we recover a malicious .NET executable. By reverse-engineering this executable using ILSpy, we uncover a backdoor account creation command containing the hidden flag.

## 1. Forensics and WMI analysis

We began by extracting the provided evidence archive, `after-hours.7z`, and navigating to the `challenge_attachments` directory. Inside, we found files typical of a Windows WMI repository (`INDEX.BTR`, `OBJECTS.DATA`, `MAPPING1.MAP`, etc).

Attackers often use WMI Event Consumers (specifically `CommandLineEventConsumer`) to achieve fileless persistence on Windows. To hunt for this, we used the `strings` utility combined with `grep` to search the `OBJECTS.DATA` file for common malicious keywords:

```
strings OBJECTS.DATA | grep -iE "CommandLineEventConsumer|powershell|bypass|downloadstring"
```

The search successfully returned a massive, Base64-encoded PowerShell script.

We copied this encoded string and took it to CyberChef. Using the **From Base64** and **Remove null bytes** operations, we decoded the PowerShell stager.

Analyzing the decoded script, we discovered exactly how the malware was operating:
`... [WmiClass]'ROOT\cimv2:Win32_HardwareTelemetry' ...`

The PowerShell script was designed to read a secondary, compressed payload stored directly inside a custom WMI class named `Win32_HardwareTelemetry`.

## 2. Payload extraction

Knowing the exact name of the malicious WMI class, we went back to our terminal and searched the `OBJECTS.DATA` file again to grab the surrounding context:

```
strings OBJECTS.DATA | grep -C 10 -iE "HardwareTelemetry"
```

This revealed the second stage of the payload: another heavily encoded string. Based on the PowerShell stager's code, we knew this string was Base64 encoded and Deflate compressed.

We used CyberChef again with the **From Base64** and **Raw Inflate** operations. The output was a Windows executable file. We downloaded this extracted file to our local machine and named it `download.exe`.

To confirm its architecture, we ran the `file` command:

```
file download.exe
```

The output confirmed it was a `PE32 executable (GUI) Intel 80386 Mono/.Net assembly`. Because it was compiled using the .NET framework, we could easily decompile it to read the original source code.

## 3. Reverse engineering (.NET)

To analyze the .NET binary on our Linux machine, we extracted and set up **ILSpy** (a popular open-source .NET decompiler):

```
cd tools
unzip ILSpy-linux-x64-Release.zip
cd artifacts/linux-64
chmod 777 IlSpy
./ILSpy
```

We opened `download.exe` inside the ILSpy graphical interface and navigated to the main `Program` execution class. 

Reviewing the decompiled C# code, we found the exact command the malware executes to establish a backdoor on the infected system:

```csharp
processStartInfo.Arguments = "/c net user patch VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9 /add";
```

The malware attempts to silently add a new local user named `patch` with a highly suspicious, Base64-looking password. We used CyberChef to decode the Base64 password string (`VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9`), which revealed the final flag.

**Flag:**
<span class="spoiler" role="button" tabindex="0" aria-label="Hidden flag">THM{P4tch_op3ned_th3_BacKd00r}</span>

## 4. Mitigation (how to fix it)

To detect and prevent WMI-based persistence and fileless malware, security teams should implement the following:

+ **Monitor WMI activity:** Use EDR (Endpoint Detection and Response) solutions or Sysmon (event ID 19, 20, 21) to log and alert on the creation of new WMI event filters, consumers, and bindings.
+ **Enable PowerShell logging:** Enable PowerShell Script Block Logging (event ID 4104) and Module Logging. This captures the deobfuscated payload in the Windows Event Logs before it executes, neutralizing the attacker's attempts to hide in Base64.
+ **Restrict administrative privileges:** WMI persistence and the execution of `net user /add` both require local administrator privileges. Enforce the principle of least privilege to prevent the initial payload from executing successfully.

<br>
- [Day 13 - The Guestbook](../13_The_Guestbook)