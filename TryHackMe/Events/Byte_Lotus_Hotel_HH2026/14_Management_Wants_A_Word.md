---
layout: default
title: Day 14 - Management Wants A Word
---

- [./ctf-writeups](../../../../) / [TryHackMe](../../../) / [Events](../../) / [Byte Lotus Hotel](../) / [Day 13 - The Guestbook](../13_The_Guestbook)

# Day 14 - Management Wants A Word      <a href="https://tryhackme.com/room/hh-managementwantsaword-6bf3cc41" target="_blank" rel="noopener"><img src="{{ '/assets/thm-logo.png' | relative_url }}" alt="TryHackMe" width="50"></a>

**Difficulty:** Hard  
**Category:** Digital forensics / Cryptography (DPAPI) / Reverse engineering

**Summary:** This challenge is by far the most interesting in this series with an advanced digital forensics and cryptography scenario (a massive "Russian doll" of decryption). WIt involves advanced Windows forensics on a KAPE triage dump. We extracted and cracked local credentials to manually decrypt the system's DPAPI Master Key. We then used Python to decrypt Chrome's AES key, revealing a saved browser password that successfully unlocked a hidden VeraCrypt container and the final flag.

## 1. Forensics and artifact extraction

We started by downloading the challenge archive and searching through the provided KAPE forensic dump. 

In the user's documents (`/KAPE/C/Users/vera/Documents/`), we found a suspicious file named `backup` with no extension, categorized simply as "data".

Next, we investigated the Google Chrome browser artifacts. Opening the `History` SQLite database in a DB browser, we found a login URL: `http://bytelotus.thm:8080/login`. 

Moving to the `Login Data` SQLite database, we queried the `logins` table and discovered a saved entry for the username `VeraSecretVault`. The password field contained an encrypted BLOB starting with the string `v10`. In Chrome forensics, the `v10` prefix indicates that the password is encrypted using AES-GCM and the decryption key is protected by the Windows Data Protection API (DPAPI).

## 2. Credential dumping and cracking

To decrypt DPAPI data, we first needed the user's Windows login password. We navigated to the registry backups in the KAPE dump (`/KAPE/C/Windows/System32/config/`) and located the `SAM` and `SYSTEM` hives.

We used Impacket's `secretsdump.py` to extract the local NTLM hashes:

```
secretsdump.py -sam SAM -system SYSTEM LOCAL
```

This yielded Vera's NTLM hash: `1241186a4aac4f34f4bf7ace71b396a8`. We took this hash to CrackStation, which successfully cracked it, revealing the plaintext password: `minivera`.

## 3. DPAPI and Chrome decryption

With Vera's Windows password, we could now unravel the DPAPI encryption chain.

**Step 1: Decrypt the DPAPI Master Key**
We located Vera's Master Key in `/AppData/Roaming/Microsoft/Protect/.../c90719ef-5b98-474e-b934-136d606a702a`. Using a DPAPI extraction tool, we decrypted it:

```
dpapi masterkey -sid S-1-5-21-2529683458-431225740-1723070931-1000 -password minivera -file ./c90719ef-5b98-474e-b934-136d606a702a
```

This gave us the decrypted Master Key: `0x5e5715ec9b6df5a86e97902692a66d28e691f05d5bc1e04d0159cfe960e94c978c07e5004a0179d3a96df2468885a28175b0b02cc064445f116a752d2b3e9d40`.

**Step 2: Extract the Chrome AES Key**
Chrome stores its encrypted AES key in the `Local State` JSON file. We created a quick Python script (`get_chrome_aes_key.py`) to parse this file, decode the base64 string, strip the first 5 bytes (the "DPAPI" header), and save the raw encrypted key to a file (`encrypted_key.bin`):

```python
import json, base64, sys
data = json.load(open(sys.argv[1], encoding="utf-8"))
enc_key = base64.b64decode(data["os_crypt"]["encrypted_key"])
blob = enc_key[5:]

with open("encrypted_key.bin", "wb") as f:
    f.write(blob)
```

We then used the decrypted Master Key to unprotect this AES key:

```
dpapi unprotect -key 0x5e5715ec9b6df5a86e97902692a66d28e691f05d5bc1e04d0159cfe960e94c978c07e5004a0179d3a96df2468885a28175b0b02cc064445f116a752d2b3e9d40 -file ./encrypted_key.bin
```

This successfully output the plaintext AES key: `206A39A0971327EA9487E4AEA9844F5D3670162456982276939A712646DA0B02`.

**Step 3: Decrypt the saved password**
We wrote a final Python script (`decrypt_chrome_password.py`) using the `Crypto.Cipher.AES` library. We supplied the encrypted BLOB from the `Login Data` database and the plaintext AES key. The script extracted the nonce (bytes 3 to 15), the ciphertext (bytes 15 to -16) and the authentication tag (last 16 bytes) to perform the AES-GCM decryption:

```python
from Crypto.Cipher import AES

blob = bytes.fromhex("763130c88a72a64f35f63e883ea0a7f64a6870e46b0bbb469a756eda88b7e324c3e1c51015aa6fd8d65ac48961e1ea324ce1707807feb3d7")
key = bytes.fromhex("206A39A0971327EA9487E4AEA9844F5D3670162456982276939A712646DA0B02")

nonce = blob[3:15]
ciphertext = blob[15:-16]
tag = blob[-16:]

cipher = AES.new(key, AES.MODE_GCM, nonce=nonce)
print(cipher.decrypt_and_verify(ciphertext, tag).decode())
```

Running this script revealed the hidden vault password: `Wh4t1sV3raD0inG0nTh1sH0st`.

## 4. Container decryption

The room's hint referenced a version number, `1.26.29`, which correlates to the open-source encryption software **VeraCrypt**. 

Assuming the 100MB `backup` file was a VeraCrypt container, we downloaded the software and used the command-line interface to mount the file into a temporary directory:

```
mkdir temp_veracrypt
veracrypt -t /KAPE/C/Users/vera/Documents/backup ./temp_veracrypt
```

When prompted, we entered the decrypted Chrome password (`Wh4t1sV3raD0inG0nTh1sH0st`) and left the PIM, keyfile, and hidden volume options blank to use the defaults. 

The container mounted successfully. Navigating into `temp_veracrypt/secret_financial_documents/`, we found a file named `important_invoice_byte_lotus.pdf` containing the final flag.

**Flag:**
<span class="spoiler" role="button" tabindex="0" aria-label="Hidden flag">THM{1t_w4s_V3r4_A11_Al0ng?!}</span>

## 5. Mitigation (how to fix it)

To protect sensitive environments against endpoint credential dumping and offline decryption, organizations should implement the following:

+ **Enforce strong passwords:** Prevent offline NTLM hash cracking by enforcing long, complex passphrases that do not exist in public dictionaries (like CrackStation's wordlists).
+ **Use specialized password managers:** Browsers' built-in password managers rely heavily on the local OS login for security. Use enterprise-grade, dedicated password managers with independent Master Passwords and Multi-Factor Authentication (MFA).
+ **Protect sensitive registry hives:** Ensure strict access controls on the SAM and SYSTEM registry hives to prevent unauthorized extraction of local credentials by attackers or malicious insiders.

<br>
- [Certificate Of Completion](https://tryhackme-certificates.s3-eu-west-1.amazonaws.com/THM-MZKGBEO5VX.pdf)