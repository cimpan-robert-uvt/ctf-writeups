---
layout: default
title: Day 8 - Towel On The Sunbed
---

- [./ctf-writeups](../../../../) / [TryHackMe](../../../) / [Events](../../) / [Byte Lotus Hotel](../) / [Day 7 - Do Not Disturb](../7_Do_Not_Disturb)

# Day 8 - Towel On The Sunbed

**Difficulty:** Medium  
**Category:** Web exploitation / Race condition / Business logic

**Summary:** This room focuses on exploiting a business logic flaw known as a Race Condition. The application attempts to limit users to one reward claim every 24 hours. However, by using Burp Suite to send multiple claim requests at the exact same millisecond, we can bypass the time check, multiply our rewards instantly, and unlock the restricted `Whale Vault`.

## 1. Reconnaissance and context

We began by accessing the target web application, "Ponzi — Wellness Rewards", hosted on port 3000. 

To interact with the application, we registered a new guest account with standard credentials and logged in. The dashboard revealed a "claim reward" feature that can supposedly only be used once every 24 hours. Our ultimate goal was to gain enough rewards to access the exclusive "Whale Vault".

The core issue here is how the server checks the time. It likely checks if the user has claimed the reward, and if not, it adds the reward and updates the timestamp. If we send multiple requests simultaneously, they might all pass the initial "has the user claimed the reward?" check before the database has time to update the timestamp.

## 2. Exploitation (race condition)

To execute this attack, we used **Burp Suite**. We started a temporary project, turned on the proxy, and opened the embedded Burp browser to route our traffic.

**Step 1: Intercepting the request**
We navigated back to the web application, turned "Intercept" on in Burp Suite, and clicked the "claim reward" button. Burp successfully caught the `POST` request before it reached the server.

**Step 2: Preparing the parallel attack**
Instead of letting the request pass normally, we right-clicked the intercepted `POST` request and selected **Send to Repeater**. We then dropped the original intercepted request so it would not be processed by the server yet.

In the Repeater tab, we prepared our Race Condition attack:
1. We right-clicked the request tab and selected **Add to group** (creating a new group).
2. We duplicated this request within the group 7 times.
3. Using the group action dropdown next to the send button, we selected **Send group in parallel**.

This specific feature in Burp Suite uses a Single Packet Attack technique, ensuring all 7 requests hit the target server at the exact same millisecond. 

**Step 3: The result**
Because the requests arrived simultaneously, the server processed all of them before it could lock the database row and update our last claim time. 

We turned the Burp intercept off, went back to the browser, and refreshed the page. The attack was successful: our account was credited with multiple rewards at once, granting us the necessary balance to enter the Whale Vault. 

Upon opening the vault, we found the flag.

**Flag:** 
<span class="spoiler" role="button" tabindex="0" aria-label="Hidden flag">THM{t0w3l_0n_th3_sunb3d_d0ubl3_sp3nt}</span>

## 3. Mitigation (how to fix it)

Race conditions (or TOCTOU - Time of Check to Time of Use flaws) are common in applications handling balances, rewards, or inventory. To prevent this, developers should implement:

+ **Database row locking:** When checking if a user is eligible for a reward, the application must use pessimistic locking (example `SELECT ... FOR UPDATE` in SQL) to lock the user's database record until the transaction is fully complete.
+ **Atomic transactions:** Ensure that the "check eligibility" and "update balance" steps occur within a single, indivisible database transaction.
+ **Strict rate limiting:** Implement robust rate limiting on the API endpoint to drop excessive requests originating from the same user within a tiny fraction of a second.

<br>
- [Day 9 - CryptoCabana](../9_CryptoCabana)