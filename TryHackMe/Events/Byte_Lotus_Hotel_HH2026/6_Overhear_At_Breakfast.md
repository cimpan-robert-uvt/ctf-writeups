---
layout: default
title: Day 6 - Overhear At Breakfast
---

- [./ctf-writeups](../../../../) / [TryHackMe](../../../) / [Events](../../) / [Byte Lotus Hotel](../) / [Day 5 - Beach Bar](../5_Beach_Bar)

# Day 6 - Overhear At Breakfast

**Difficulty:** Easy  
**Category:** OSINT / Information disclosure

**Summary:** This room introduces Open Source Intelligence (OSINT) techniques. By analyzing a leaked chat conversation, we identify a target email address and a hint about a third-party avatar service. We generate the required MD5 hash to locate the hidden public profile, extract encoded data, and decode it to retrieve the flag.

## 1. Reconnaissance and context

Our starting point is a leaked chat log provided in an image file named `conversation.png`.

<details class="evidence">
<summary>view the leaked conversation</summary>
<p><img src="../conversation.png"></p>
</details>

Reading through the conversation between Ponzi and Lambo, we gather two critical pieces of intelligence:
1. Lambo refuses to share standard social media handles but mentions he used an old profile linking tool that "started with a G".
2. Lambo provides his direct email address: `lambobytelotushotel@gmail.com`.

The tool starting with a "G" that links avatars and profiles to email addresses is **Gravatar** (Globally Recognized Avatar). Gravatar works by taking a user's email address, hashing it using the MD5 algorithm, and using that hash as the unique identifier for their public profile and avatar image.

## 2. Exploitation (data extraction and decoding)

To find Lambo's hidden Gravatar profile, we first need to convert his email address into an MD5 hash. We can do this easily in the terminal. The `-n` flag is crucial here to ensure no hidden newline characters are hashed along with the email:

```
echo -n "lambobytelotushotel@gmail.com" | md5sum
```

This command outputs the hash: `d4a5fc5d3128890778667e24617d7cc0`.

Next, we construct the Gravatar URL using this hash to view the public profile:
`https://gravatar.com/d4a5fc5d3128890778667e24617d7cc0`

By inspecting the contents of this profile page, we find a suspicious text string: 
`VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50MWZpM2R9`

This string ends with numbers and mixed casing, which is a strong indicator of base64 encoding. We take this string back to our terminal to decode it:

```
echo "VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50MWZpM2R9" | base64 -d
```

The decoding process successfully reveals the hidden message.

**Flag:** 
<span class="spoiler" role="button" tabindex="0" aria-label="Hidden flag">THM{S3creT_Pr0fil3_H4s_b33n_Ident1fi3d}</span>

## 3. Mitigation (how to fix it)

To prevent OSINT leaks and protect personal or corporate information, the following practices should be implemented:

+ **Manage digital footprints:** Users must be educated on the risks of linking work or sensitive email addresses to public third-party services like Gravatar, as these platforms can aggregate and expose historical data.
+ **Sanitize abandoned accounts:** Ensure that old, unused profiles on public platforms are fully deleted and scrubbed of information, rather than just abandoned with sensitive data left behind.
+ **Understand hash limitations:** Developers and users should be aware that MD5 hashing (as used by Gravatar to identify emails) is not encryption. Because email addresses are predictable, attackers can easily reverse these hashes using rainbow tables or OSINT tools to expose the underlying email addresses of public avatars.

<br>
- [Day 7 - Do Not Disturb](../7_Do_Not_Disturb)