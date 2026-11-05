---
layout: default
title: Day 4 - Packet Light
---

- [./ctf-writeups](../../../../) / [TryHackMe](../../../) / [Events](../../) / [Byte Lotus Hotel](../) / [Day 3 - Complimentary Wellness App](../3_Complimentary)

# Day 4 - Network traffic analysis      <a href="https://tryhackme.com/room/hh-packedlight-02e5330c" target="_blank" rel="noopener"><img src="{{ '/assets/thm-logo.png' | relative_url }}" alt="TryHackMe" width="50"></a>

**Difficulty:** Easy  
**Category:** Network forensics / Wireshark

**Summary:** This room focuses on network forensics and traffic analysis. By analyzing a packet capture file (`.pcap`) in Wireshark, we identify a suspicious python script being downloaded over HTTP. This script reveals a custom XOR encryption key and a specific cookie used to exfiltrate data. We then extract the encoded cookie values from the network traffic and decode them to retrieve the hidden flag.

## 1. Reconnaissance and traffic analysis

We started by opening the provided network traffic file in Wireshark. Because raw network captures contain a lot of noise, we filtered the traffic to show only unencrypted web requests by typing `http` into the display filter.

Looking through the HTTP requests, we spotted a highly suspicious `GET` request asking for a file named `/temp/updates.py`. 

To see exactly what was downloaded, we right-clicked the packet and selected **Follow -> TCP Stream**. This revealed the source code of the python script.

Inside the script, we found two critical pieces of information:
1. The script uses a custom XOR encryption function with a specific key constructed from two parts:
```python
def getkey():
    p1 = "H0t3lSt@ff0Nly"
    p2 = "K3epS3cr3t!"
    return p1 + p2

def xor(data: bytes, key: bytes) -> bytes:
    return bytes(b ^ key[i % len(key)] for i, b in enumerate(data))
```
2. The script hides the encrypted and base64-encoded data inside an HTTP cookie named `hotel_sess_state`.

## 2. Exploitation (data extraction and decryption)

Knowing the exact name of the cookie used for exfiltration, we applied a new filter in Wireshark to isolate only the packets carrying the stolen data:

```
http.cookie contains "hotel_sess_state"
```

We clicked on each filtered packet, expanded the "Hypertext Transfer Protocol" section in the packet details, and looked at the Cookie field. We manually extracted the base64 string from each packet. The extracted fragments looked like this:

`HA==,AA==,BQ==,Mw==,Hg==,ew==,Og==,fA==,Fw==,eQ==,Ow==,Fw==,Pw==,fA==,PQ==,Kw==,IA==,eQ==,Jg==,Lw==,Fw==,eA==,Pg==,LQ==,Gg==,Fw==,MQ==,eA==,PQ==,NQ==`

To decrypt the message, we took these values to **CyberChef** and built a recipe with two steps:
    1. **From Base64**
    2. **XOR**

For the XOR operation, we set the scheme to UTF8 and used the key we found in the python script (`H0t3lSt@ff0NlyK3epS3cr3t!`). Applying this recipe successfully decrypted the fragments and revealed the flag.

**Flag:** 
<span class="spoiler" role="button" tabindex="0" aria-label="Hidden flag">THM{V3r4_1s_w4uch1ng_0veR_y0u}</span>

## 3. Mitigation (how to fix it)

To prevent this type of data exfiltration and protect network traffic, the following measures should be implemented:

+ **Enforce HTTPS (TLS):** Never use unencrypted HTTP for transmitting sensitive data or downloading scripts. If the traffic had been encrypted with TLS, the attacker's script and the exfiltrated cookies would not have been readable in Wireshark.
+ **Avoid custom cryptography:** Do not rely on simple, custom encryption methods like hardcoded XOR keys. Always use established, industry-standard cryptographic algorithms.
+ **Implement network monitoring (IDS/IPS):** Configure Intrusion Detection Systems to monitor and flag unusual behaviors, such as downloading `.py` files from temporary web directories or sending cookies with abnormally formatted base64 payloads.

<br>
- [Day 5 - Beach Bar](../5_Beach_Bar)