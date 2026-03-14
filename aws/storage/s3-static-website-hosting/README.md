# AWS S3 Static Website Hosting

## Hosting a Public Static Website on Amazon S3 Using the AWS CLI

![AWS S3](https://img.shields.io/badge/AWS-S3-orange?style=flat-square&logo=amazonaws)
![CLI](https://img.shields.io/badge/AWS_CLI-v1.40.19-blue?style=flat-square)
![Region](https://img.shields.io/badge/Region-us--east--1-green?style=flat-square)
![Status](https://img.shields.io/badge/Status-Production_Ready-brightgreen?style=flat-square)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Pre-Flight Checks](#pre-flight-checks)
  - [Phase 1 - Create the S3 Bucket](#phase-1---create-the-s3-bucket)
  - [Phase 2 - Disable Block Public Access](#phase-2---disable-block-public-access)
  - [Phase 3 - Configure Static Website Hosting](#phase-3---configure-static-website-hosting)
  - [Phase 4 - Attach a Public Bucket Policy](#phase-4---attach-a-public-bucket-policy)
  - [Phase 5 - Upload the Index Document](#phase-5---upload-the-index-document)
  - [Phase 6 - Verify Public Accessibility](#phase-6---verify-public-accessibility)
- [Outcome](#outcome)
- [Troubleshooting Reference](#troubleshooting-reference)
- [Security Considerations](#security-considerations)
- [Repository Placement](#repository-placement)

---

## Overview

This document provides a complete, production-grade implementation guide for hosting a static website on **Amazon S3** using the **AWS CLI**. It covers bucket provisioning, public access configuration, static website hosting enablement, bucket policy attachment, file upload, and end-to-end verification.

This guide is written for DevOps engineers, cloud practitioners, and platform teams operating in AWS environments. Every command is accompanied by its expected output and verification step to eliminate ambiguity and prevent configuration drift.

---

## Problem Statement

The DevOps team requires a publicly accessible internal information portal hosted on AWS without the overhead of managing web servers or compute infrastructure. The solution must:

- Serve static HTML content directly from an S3 bucket
- Be accessible to external users via a public S3 website endpoint URL
- Require zero ongoing server management
- Be fully reproducible via the AWS CLI

---

## Architecture

```
External User
     |
     | HTTP Request
     v
S3 Website Endpoint
(http://[bucket].s3-website-[region].amazonaws.com)
     |
     v
S3 Bucket (Static Website Hosting Enabled)
     |
     +-- index.html  (Index Document)
     |
Bucket Policy: s3:GetObject --> Principal: * (Public Read)
```

> **Note:** The native S3 website endpoint serves traffic over **HTTP only**. For HTTPS support, place an Amazon CloudFront distribution in front of the S3 bucket. This is outside the scope of this guide.

---

## Prerequisites

| Requirement | Detail |
|---|---|
| AWS CLI | v1.x or v2.x installed and configured |
| IAM Permissions | `s3:CreateBucket`, `s3:PutBucketWebsite`, `s3:PutBucketPolicy`, `s3:PutPublicAccessBlock`, `s3:PutObject` |
| AWS Region | `us-east-1` |
| Index File | `/root/index.html` present on the host machine |
| Network Access | Outbound HTTPS to `*.amazonaws.com` |

---

## Implementation

### Pre-Flight Checks

Before executing any AWS commands, validate the environment state. Do not skip these steps.

**Verify AWS CLI installation:**

```bash
aws --version
```

*Expected output:*

```
aws-cli/1.40.19 Python/3.10.17 Linux/6.8.0-90-generic botocore/1.38.20
```

---

**Verify IAM identity and account context:**

```bash
aws sts get-caller-identity
```

*Expected output:*

```json
{
    "UserId": "AIDAWVQYIOCKBQ3TTGWME",
    "Account": "458538643604",
    "Arn": "arn:aws:iam::458538643604:user/kk_labs_user_375792"
}
```

---

**Confirm the active AWS region:**

```bash
aws configure get region
```

*Expected output:*

```
us-east-1
```

If the region is not set, configure it:

```bash
aws configure set region us-east-1
```

---

**Confirm the index.html file exists on the host:**

```bash
ls -la /root/index.html
```

*Expected output:*

```
-rw-r--r-- 1 root root 20 Mar 14 00:24 /root/index.html
```

> **Stop here** if this file is missing. The upload step in Phase 5 will fail without it.

---

**SCREENSHOT**
<img width="1028" height="529" alt="image" src="https://github.com/user-attachments/assets/8eebf5dc-f596-48b2-95c3-2653ab6e3abb" />

> *Screenshot of terminal showing all four pre-flight commands executing successfully with expected outputs.*

---

### Phase 1 - Create the S3 Bucket

**Step 1.1 - Create the bucket:**

```bash
aws s3api create-bucket \
    --bucket xfusion-web-27751 \
    --region us-east-1
```

*Expected output:*

```json
{
    "Location": "/xfusion-web-27751"
}
```

> **Important:** `us-east-1` is the only AWS region that does **not** require the `--create-bucket-configuration LocationConstraint` parameter. All other regions require it.

---

**Step 1.2 - Confirm the bucket is accessible:**

```bash
aws s3api head-bucket --bucket xfusion-web-27751
```

*Expected output:*

```json
{
    "BucketRegion": "us-east-1",
    "AccessPointAlias": false
}
```

---

**Step 1.3 - Confirm the exit code is zero:**

```bash
echo $?
```

*Expected output:*

```
0
```

Any non-zero exit code indicates a failure. Do not proceed until this returns `0`.

---

**[SCREENSHOT PLACEHOLDER]**
> *Insert screenshot showing `create-bucket` returning the Location field and `head-bucket` confirming `BucketRegion: us-east-1` with exit code `0`.*

---

### Phase 2 - Disable Block Public Access

By default, AWS blocks all public access on newly created S3 buckets. This safeguard must be explicitly disabled before a public bucket policy can take effect.

**Step 2.1 - Disable all block public access settings:**

```bash
aws s3api put-public-access-block \
    --bucket xfusion-web-27751 \
    --public-access-block-configuration BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false
```

*Expected output:* No output. A silent return indicates success.

---

**Step 2.2 - Verify all four settings are disabled:**

```bash
aws s3api get-public-access-block --bucket xfusion-web-27751
```

*Expected output:*

```json
{
    "PublicAccessBlockConfiguration": {
        "BlockPublicAcls": false,
        "IgnorePublicAcls": false,
        "BlockPublicPolicy": false,
        "RestrictPublicBuckets": false
    }
}
```

> **Do not proceed to Phase 3 until all four values confirm `false`.** Any `true` value will cause bucket policy application to fail with an `AccessDenied` error.

---

**[SCREENSHOT PLACEHOLDER]**
> *Insert screenshot showing `get-public-access-block` output with all four configuration values set to `false`.*

---

### Phase 3 - Configure Static Website Hosting

**Step 3.1 - Enable static website hosting with index.html as the index document:**

```bash
aws s3api put-bucket-website \
    --bucket xfusion-web-27751 \
    --website-configuration '{
        "IndexDocument": {
            "Suffix": "index.html"
        },
        "ErrorDocument": {
            "Key": "index.html"
        }
    }'
```

*Expected output:* No output. A silent return indicates success.

> The `ErrorDocument` is set to `index.html` as a safe fallback since no dedicated error page exists. This prevents broken 404 responses.

---

**Step 3.2 - Verify the website hosting configuration is active:**

```bash
aws s3api get-bucket-website --bucket xfusion-web-27751
```

*Expected output:*

```json
{
    "IndexDocument": {
        "Suffix": "index.html"
    },
    "ErrorDocument": {
        "Key": "index.html"
    }
}
```

> A `NoSuchWebsiteConfiguration` error means Step 3.1 did not apply correctly. Re-run it verbatim.

---

**[SCREENSHOT PLACEHOLDER]**
> *Insert screenshot of `get-bucket-website` returning the `IndexDocument` and `ErrorDocument` configuration.*

---

### Phase 4 - Attach a Public Bucket Policy

**Step 4.1 - Write the bucket policy to a temporary file:**

```bash
cat > /tmp/bucket-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::xfusion-web-27751/*"
        }
    ]
}
EOF
```

---

**Step 4.2 - Verify the policy file contents before applying:**

```bash
cat /tmp/bucket-policy.json
```

*Expected output:*

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::xfusion-web-27751/*"
        }
    ]
}
```

> **Visually confirm the bucket name in the `Resource` ARN is exactly `xfusion-web-27751` before proceeding.** A typo here will silently apply an ineffective policy.

---

**Step 4.3 - Apply the policy to the bucket:**

```bash
aws s3api put-bucket-policy \
    --bucket xfusion-web-27751 \
    --policy file:///tmp/bucket-policy.json
```

*Expected output:* No output. A silent return indicates success.

---

**Step 4.4 - Verify the policy is live on the bucket:**

```bash
aws s3api get-bucket-policy --bucket xfusion-web-27751
```

*Expected output:*

```json
{
    "Policy": "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Sid\":\"PublicReadGetObject\",\"Effect\":\"Allow\",\"Principal\":\"*\",\"Action\":\"s3:GetObject\",\"Resource\":\"arn:aws:s3:::xfusion-web-27751/*\"}]}"
}
```

---

**[SCREENSHOT PLACEHOLDER]**
> *Insert screenshot showing `get-bucket-policy` returning the full policy JSON confirming `Effect: Allow`, `Principal: *`, and the correct `Resource` ARN.*

---

### Phase 5 - Upload the Index Document

**Step 5.1 - Upload index.html with the correct content type:**

```bash
aws s3 cp /root/index.html s3://xfusion-web-27751/index.html \
    --content-type "text/html"
```

*Expected output:*

```
upload: ./index.html to s3://xfusion-web-27751/index.html
```

> The `--content-type "text/html"` flag is required. Without it, S3 may assign `binary/octet-stream`, causing browsers to download the file rather than render it.

---

**Step 5.2 - Confirm the file is present in the bucket:**

```bash
aws s3 ls s3://xfusion-web-27751/
```

*Expected output:*

```
2026-03-14 00:34:41         20 index.html
```

---

**Step 5.3 - Verify the object metadata and content type:**

```bash
aws s3api head-object \
    --bucket xfusion-web-27751 \
    --key index.html
```

*Expected output:*

```json
{
    "AcceptRanges": "bytes",
    "LastModified": "Sat, 14 Mar 2026 00:34:41 GMT",
    "ContentLength": 20,
    "ETag": "\"bddd508bfd1fab1d7eabc0ad6d8db1ea\"",
    "ContentType": "text/html",
    "ServerSideEncryption": "AES256",
    "Metadata": {}
}
```

> Confirm `ContentType` is `text/html`. If it shows anything else, re-run Step 5.1.

---

**[SCREENSHOT PLACEHOLDER]**
> *Insert screenshot showing the `s3 cp` upload confirmation, `s3 ls` listing `index.html`, and `head-object` returning `ContentType: text/html`.*

---

### Phase 6 - Verify Public Accessibility

**Step 6.1 - Validate the HTTP response code:**

```bash
curl -I http://xfusion-web-27751.s3-website-us-east-1.amazonaws.com
```

*Expected output:*

```
HTTP/1.1 200 OK
x-amz-id-2: IPe3cVix+42BJQK5iOU5udQj2zbYf+H7AjAw37rl5PWRrL92iA1INkFY7nv4qlwTZLqJdppW6Jg=
x-amz-request-id: 8B6S80HB5SZ2966E
Date: Sat, 14 Mar 2026 00:36:04 GMT
Last-Modified: Sat, 14 Mar 2026 00:34:41 GMT
ETag: "bddd508bfd1fab1d7eabc0ad6d8db1ea"
Content-Type: text/html
Content-Length: 20
Server: AmazonS3
```

---

**Step 6.2 - Retrieve the full page content to confirm HTML is being served:**

```bash
curl http://xfusion-web-27751.s3-website-us-east-1.amazonaws.com
```

*Expected output:*

```
Welcome to KKE labs!
```

---

**[SCREENSHOT PLACEHOLDER]**
> *Insert screenshot showing `curl -I` returning `HTTP/1.1 200 OK` and `curl` returning the rendered HTML content `Welcome to KKE labs!`.*

---

## Outcome

The static website is live, publicly accessible, and correctly serving content via the S3 website endpoint.

| Metric | Value |
|---|---|
| **Bucket Name** | `xfusion-web-27751` |
| **AWS Region** | `us-east-1` |
| **Website URL** | `http://xfusion-web-27751.s3-website-us-east-1.amazonaws.com` |
| **HTTP Status** | `200 OK` |
| **Content Served** | `Welcome to KKE labs!` |
| **Content Type** | `text/html` |
| **Server Side Encryption** | `AES256` |

### Full Phase Completion Summary

| Phase | Action | Status |
|---|---|---|
| Pre-flight | CLI, identity, region, and file verified | Complete |
| Phase 1 | Bucket `xfusion-web-27751` created in `us-east-1` | Complete |
| Phase 2 | All block public access settings disabled | Complete |
| Phase 3 | Static website hosting enabled with `index.html` | Complete |
| Phase 4 | Public `s3:GetObject` bucket policy applied | Complete |
| Phase 5 | `index.html` uploaded with `text/html` content type | Complete |
| Phase 6 | Website publicly accessible, serving correct content | Complete |

---

## Troubleshooting Reference

| Symptom | Root Cause | Resolution |
|---|---|---|
| `403 Forbidden` | Block public access still enabled or bucket policy missing | Repeat Phase 2 and Phase 4 |
| `404 Not Found` | `index.html` not uploaded or index document misconfigured | Repeat Phase 3 and Phase 5 |
| File downloads instead of rendering | `ContentType` set to `binary/octet-stream` | Re-run Phase 5 Step 5.1 with `--content-type "text/html"` |
| `BucketAlreadyExists` | Bucket name is taken globally across all AWS accounts | Verify the exact bucket name and retry |
| `IllegalLocationConstraintException` | Wrong region flag used during creation | Confirm region is `us-east-1` and retry Phase 1 |
| `MalformedPolicy` | JSON syntax error in policy file | Re-run Phase 4 Step 4.1 and verify with `cat` before applying |
| `AccessDenied` on `put-bucket-policy` | Block public access still active | Confirm all four values are `false` in Phase 2 before applying the policy |

---

## Security Considerations

This configuration grants **unrestricted public read access** (`Principal: *`) to all objects in the bucket. This is intentional for a public static website but requires the following awareness:

- **Only `s3:GetObject` is granted.** Write, delete, and list operations remain restricted to authorized IAM identities.
- **Do not store sensitive files in this bucket.** Any object uploaded to `xfusion-web-27751` will be publicly readable.
- **Server-side encryption (`AES256`) is active** on all uploaded objects, providing encryption at rest even for publicly accessible content.
- **For HTTPS support**, front the bucket with an Amazon CloudFront distribution using an ACM SSL certificate. The native S3 website endpoint does not support HTTPS.
- **Review public access settings regularly** using AWS Config or AWS Security Hub to detect unintended configuration drift.

---



<img width="1027" height="694" alt="image" src="https://github.com/user-attachments/assets/7be07ffa-332f-4339-9364-4b98f1ef5421" />
<img width="1030" height="755" alt="image" src="https://github.com/user-attachments/assets/c2f0d0f1-af8d-45b9-abb6-b924df96afc0" />
<img width="1038" height="792" alt="image" src="https://github.com/user-attachments/assets/2339fefb-dcea-4858-9af2-b493a5a639f9" />
<img width="1031" height="739" alt="image" src="https://github.com/user-attachments/assets/4e7812b7-4610-4c1a-a070-486b7219e0ea" />
<img width="1036" height="598" alt="image" src="https://github.com/user-attachments/assets/7a73bb77-98eb-4bf3-a7b5-d856d880cbd7" />
<img width="1033" height="783" alt="image" src="https://github.com/user-attachments/assets/cf365cd0-1ae8-46e9-83aa-2a8bf6b857ab" />
<img width="1036" height="613" alt="image" src="https://github.com/user-attachments/assets/0d8be373-fed5-4a85-a0dc-a39fc2f75fa1" />
<img width="1035" height="812" alt="image" src="https://github.com/user-attachments/assets/20800c9b-f9eb-4ca4-971a-7d291e3df0b0" />
<img width="1031" height="316" alt="image" src="https://github.com/user-attachments/assets/22616771-06af-4c10-b0eb-4d600bf7432f" />
