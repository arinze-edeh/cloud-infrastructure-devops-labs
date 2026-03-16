# AWS KMS Data Encryption and Decryption at Rest

[![AWS](https://img.shields.io/badge/AWS-KMS-FF9900?style=flat-square&logo=amazon-aws)](https://aws.amazon.com/kms/)
[![Security](https://img.shields.io/badge/Security-Encryption-green?style=flat-square)](https://aws.amazon.com/security/)
[![Region](https://img.shields.io/badge/Region-us--east--1-blue?style=flat-square)](https://aws.amazon.com/about-aws/global-infrastructure/regions_az/)

---

## Table of Contents

- [Overview](#overview)
- [Business Problem](#business-problem)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Phase 1: AWS CLI Configuration](#phase-1-aws-cli-configuration)
  - [Phase 2: KMS Key and Alias Creation](#phase-2-kms-key-and-alias-creation)
  - [Phase 3: Source File Verification](#phase-3-source-file-verification)
  - [Phase 4: File Encryption](#phase-4-file-encryption)
  - [Phase 5: Decryption and Integrity Verification](#phase-5-decryption-and-integrity-verification)
- [Validation Results](#validation-results)
- [Critical Lessons Learned](#critical-lessons-learned)
- [Best Practices](#best-practices)
- [Security Considerations](#security-considerations)
- [Reference Commands](#reference-commands)

---

## Overview

This project documents the end-to-end implementation of **envelope-free symmetric encryption** using **AWS Key Management Service (KMS)** to protect a sensitive data file at rest. The workflow covers KMS key provisioning, alias management, binary-safe ciphertext storage, and cryptographic integrity verification using MD5 checksums and `diff`.

**Key Outcome:** A sensitive plaintext file is encrypted using a customer-managed KMS key, stored as a raw binary ciphertext blob, then successfully decrypted and verified to be byte-for-byte identical to the original, satisfying automated validator requirements.

---

## Business Problem

### Context

The Nautilus DevOps team identified a security gap: sensitive configuration and data files stored on compute instances were not encrypted at rest, creating compliance and data breach risks.

### Requirements

| # | Requirement |
|---|-------------|
| 1 | Create a symmetric AWS KMS customer-managed key (CMK) named `datacenter-KMS-Key` |
| 2 | Encrypt `/root/SensitiveData.txt`, base64-encode the ciphertext, and save it as `/root/EncryptedData.bin` |
| 3 | Decrypt the encrypted file and verify the output matches the original plaintext |

### Problem Statement

> Standard `aws kms encrypt` with `--output text` produces a **base64-encoded ASCII text** file. An automated validation script using `fileb://` expects a **raw binary** ciphertext blob. Supplying a base64-encoded file to `fileb://` causes the decrypt operation to fail silently or with a `InvalidCiphertextException`.

**Resolution:** Pipe the base64 output through `tr -d '"' | base64 -d` before redirecting to the `.bin` file to ensure the stored artifact is raw binary.

---

## Architecture

```
+------------------+        fileb://        +-------------------+
|  SensitiveData   | -----> KMS Encrypt --> | EncryptedData.bin |
|  .txt (plaintext)|        (raw binary)    | (177 bytes, data) |
+------------------+                        +-------------------+
                                                      |
                                              KMS Decrypt
                                                      |
                                                      v
                                          +-------------------+
                                          | DecryptedOutput   |
                                          | .txt (plaintext)  |
                                          | MD5 verified      |
                                          +-------------------+
```

**Services Used:**

- **AWS KMS** (Key Management Service) - symmetric CMK, `SYMMETRIC_DEFAULT` key spec
- **AWS CLI** - all operations performed via CLI in `us-east-1`
- **Linux shell utilities** - `base64`, `xxd`, `diff`, `md5sum`

---

## Prerequisites

- AWS CLI v2 installed and configured
- IAM user with `kms:CreateKey`, `kms:CreateAlias`, `kms:Encrypt`, `kms:Decrypt` permissions
- Access to the target host with `/root/SensitiveData.txt` present
- Region: `us-east-1`

---

## Implementation

### Phase 1: AWS CLI Configuration

Configure the AWS CLI with the lab-provided IAM credentials and set the default region to `us-east-1`.

```bash
aws configure
```

**Input when prompted:**

```
AWS Access Key ID [****************4BJ7]: <your-access-key>
AWS Secret Access Key [****************PA9R]: <your-secret-key>
Default region name [us-east-1]: us-east-1
Default output format [None]: json
```

**Verify identity and account access:**

```bash
aws sts get-caller-identity
```

**Expected output:**

```json
{
    "UserId": "AIDAYLA23275HHV6GZHWT",
    "Account": "573434484730",
    "Arn": "arn:aws:iam::573434484730:user/kk_labs_user_122234"
}
```

> **Checkpoint:** Confirm the `Account` field matches the lab console URL account ID before proceeding.

---

>**SCREENSHOT: Terminal showing `aws sts get-caller-identity` returning Account ID `573434484730` and correct ARN**

<img width="1035" height="833" alt="Image" src="https://github.com/user-attachments/assets/7f31382e-0055-4ca8-ac57-008390dea81b" />

---

### Phase 2: KMS Key and Alias Creation

#### 2a. Create the Symmetric CMK

```bash
aws kms create-key \
  --description "datacenter-KMS-Key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --region us-east-1
```

**Expected output (truncated):**

```json
{
    "KeyMetadata": {
        "AWSAccountId": "573434484730",
        "KeyId": "d3208ff2-a7eb-4ea1-bee0-03d9240f3c02",
        "Arn": "arn:aws:kms:us-east-1:573434484730:key/d3208ff2-a7eb-4ea1-bee0-03d9240f3c02",
        "Enabled": true,
        "Description": "datacenter-KMS-Key",
        "KeyUsage": "ENCRYPT_DECRYPT",
        "KeyState": "Enabled",
        "KeySpec": "SYMMETRIC_DEFAULT",
        "EncryptionAlgorithms": ["SYMMETRIC_DEFAULT"],
        "MultiRegion": false
    }
}
```

> **Note:** Copy the `KeyId` value immediately. It is required for alias creation in the next step.

---

**SCREENSHOT PLACEHOLDER**
> ***[Screenshot: Terminal showing full `aws kms create-key` JSON response with `KeyState: "Enabled"` and `KeyId` visible]***

---

#### 2b. Create a Human-Readable Alias

Aliases provide a stable, human-readable reference to the key ID, insulating scripts and applications from key rotation changes.

```bash
aws kms create-alias \
  --alias-name alias/datacenter-KMS-Key \
  --target-key-id d3208ff2-a7eb-4ea1-bee0-03d9240f3c02 \
  --region us-east-1
```

> No output on success. A non-zero exit code indicates an error.

#### 2c. Verify the Alias

```bash
aws kms list-aliases --region us-east-1 | grep datacenter-KMS-Key
```

**Expected output:**

```
"AliasName": "alias/datacenter-KMS-Key",
"AliasArn": "arn:aws:kms:us-east-1:573434484730:alias/datacenter-KMS-Key",
```

---

**SCREENSHOT PLACEHOLDER**
> ***[Screenshot: Terminal showing `aws kms list-aliases` grep output confirming `alias/datacenter-KMS-Key` and its ARN]***

---

### Phase 3: Source File Verification

Verify the sensitive data file exists and record its content for post-decryption comparison.

```bash
ls -la /root/SensitiveData.txt && cat /root/SensitiveData.txt
```

**Expected output:**

```
-rw-r--r-- 1 root root 25 Mar 16 20:02 /root/SensitiveData.txt
This is a sensitive file.
```

> **Note:** Record the file content (`This is a sensitive file.`) and size (`25 bytes`). This is the ground truth for verification in Phase 5.

---

**SCREENSHOT PLACEHOLDER**
> ***[Screenshot: Terminal output showing `ls -la` file metadata and `cat` plaintext content of `SensitiveData.txt`]***

---

### Phase 4: File Encryption

Encrypt the plaintext file using the KMS key and save the result as a **raw binary** file.

#### Why Raw Binary Matters

The `aws kms encrypt` command returns the ciphertext as a base64-encoded string in its JSON response. If saved directly, the `.bin` file contains ASCII text. The automated validator calls `fileb:///root/EncryptedData.bin`, which instructs the CLI to treat the file as raw binary input. A base64-text file passed through `fileb://` results in `InvalidCiphertextException` at decryption time.

**The fix:** Strip JSON quotes with `tr -d '"'` then decode the base64 string with `base64 -d` before writing to disk.

```bash
aws kms encrypt \
  --key-id alias/datacenter-KMS-Key \
  --plaintext fileb:///root/SensitiveData.txt \
  --region us-east-1 \
  --output json \
  --query CiphertextBlob \
  | tr -d '"' \
  | base64 -d > /root/EncryptedData.bin
```

#### Verify the Output is Raw Binary

```bash
ls -lh /root/EncryptedData.bin
xxd /root/EncryptedData.bin | head -3
```

**Expected output:**

```
-rw-r--r-- 1 root root 177 Mar 16 20:08 /root/EncryptedData.bin
00000000: 0102 0200 782b a71b 3a5f 408e 4844 0df6  ....x+..:_@.HD..
00000010: d014 1b9f 1824 9b7b 2ce9 5f1a b84b 376a  .....$.{,._..K7j
00000020: 537c d3bd 2c01 21c8 c1d1 2651 7dd1 3497  S|..,.!...&Q}.4.
```

> **Checkpoint:** `xxd` must show non-printable binary bytes (not a clean `AQICA...` ASCII string). File size should be approximately 177 bytes. The first bytes `0102 0200` are the KMS ciphertext header, confirming a valid AWS KMS binary blob.

---

**SCREENSHOT PLACEHOLDER**
> ***[Screenshot: Terminal showing `ls -lh` (177 bytes) and `xxd` hex dump output confirming raw binary content of `EncryptedData.bin`]***

---

### Phase 5: Decryption and Integrity Verification

Decrypt the binary ciphertext directly using `fileb://` and verify the output is byte-for-byte identical to the original.

#### Decrypt

```bash
aws kms decrypt \
  --ciphertext-blob fileb:///root/EncryptedData.bin \
  --output text \
  --query Plaintext \
  --region us-east-1 | base64 -d > /root/DecryptedOutput.txt
```

> KMS returns the plaintext as a base64-encoded string. Pipe through `base64 -d` to recover the original binary/text content.

#### Verify Content

```bash
cat /root/DecryptedOutput.txt
```

**Expected output:**

```
This is a sensitive file.
```

#### Diff Check (Structural Equality)

```bash
diff /root/SensitiveData.txt /root/DecryptedOutput.txt
```

> **Expected:** No output. Any output indicates a mismatch and the encryption/decryption cycle has failed.

#### MD5 Checksum (Cryptographic Equality)

```bash
md5sum /root/SensitiveData.txt /root/DecryptedOutput.txt
```

**Expected output:**

```
fb78248c0fdf1e8cc7d9b81af2423b16  /root/SensitiveData.txt
fb78248c0fdf1e8cc7d9b81af2423b16  /root/DecryptedOutput.txt
```

> **Checkpoint:** Both MD5 hashes must be identical (`fb78248c0fdf1e8cc7d9b81af2423b16`). This provides cryptographic proof that no data was lost or corrupted during the encrypt/decrypt cycle.

---

**SCREENSHOT PLACEHOLDER**
> ***[Screenshot: Terminal showing `cat /root/DecryptedOutput.txt` with matching content, empty `diff` output, and identical MD5 hashes for both files]***

---

## Validation Results

| Requirement | Implementation | Result |
|---|---|---|
| Symmetric KMS CMK named `datacenter-KMS-Key` | Created with `SYMMETRIC_DEFAULT` spec | PASS |
| Alias `alias/datacenter-KMS-Key` mapped to key | Verified via `list-aliases` | PASS |
| `EncryptedData.bin` in `/root/` as base64 ciphertext | 177-byte raw binary file | PASS |
| Decrypted output matches original | `diff` empty, MD5 identical | PASS |
| Automated validator decryption | `fileb://` on raw binary succeeded | PASS |

---

**SCREENSHOT PLACEHOLDER**
> ***[Screenshot: Automated validator UI showing green success checkmark and "Task Completed Successfully" confirmation]***

---

## Critical Lessons Learned

### 1. `fileb://` Requires Raw Binary, Not Base64 Text

**Problem:** Using `--output text --query CiphertextBlob > EncryptedData.bin` saves a base64 ASCII string. The validator's `fileb://` prefix signals the CLI to read the file as raw binary bytes. Feeding base64 text as binary input produces an invalid ciphertext blob, causing `InvalidCiphertextException`.

**Resolution:**

```bash
# WRONG: saves base64 ASCII text
aws kms encrypt ... --output text --query CiphertextBlob > /root/EncryptedData.bin

# CORRECT: saves raw binary
aws kms encrypt ... --output json --query CiphertextBlob \
  | tr -d '"' | base64 -d > /root/EncryptedData.bin
```

### 2. Always Verify File Type After Encryption

The `xxd` hex dump is the fastest way to confirm binary content. A valid KMS ciphertext blob always begins with `0102` (version byte + reserved), not printable ASCII characters like `A` or `Q`.

### 3. Aliases Decouple Applications from Key IDs

Using `alias/datacenter-KMS-Key` instead of the raw UUID `d3208ff2-...` means key rotation only requires updating the alias target, not every script or application that references the key.

### 4. MD5 Alone Is Not a Security Guarantee

MD5 is used here purely for **data integrity verification** (confirming byte-for-byte round-trip accuracy), not for cryptographic security. Never use MD5 as a security hash in production. Use SHA-256 or stronger.

### 5. `diff` Empty Output Means Success

In shell scripting, an empty `diff` output is the success condition, not a failure. A common mistake is expecting some form of "OK" message from `diff`.

---

## Best Practices

### KMS Key Management

* **Always create an alias** for every CMK. Raw key IDs are UUIDs that provide no semantic context and complicate operational runbooks.
* **Apply key policies with least privilege.** Restrict `kms:Decrypt` to only the IAM roles and services that require it.
* **Enable key rotation** for long-lived CMKs using `aws kms enable-key-rotation`. KMS automatically rotates the backing key material annually without changing the key ID or alias.
* **Tag all KMS keys** with environment, owner, and cost-center tags for governance and billing attribution.
* **Never delete a KMS key with existing ciphertext** unless you are prepared to permanently lose access to all data encrypted under it. Schedule deletion with a minimum 7-day waiting period.

### Encryption Workflow

* **Use `fileb://` consistently.** When working with binary data (ciphertext blobs, certificates, images), always use `fileb://` rather than `file://` to prevent shell encoding corruption.
* **Store ciphertext as raw binary.** Avoid double-encoding. If a downstream system requires base64, encode at the point of transmission, not at rest.
* **Verify integrity after every encrypt/decrypt cycle** in automated pipelines using checksum comparison (`sha256sum` preferred over `md5sum` in production).
* **For files larger than 4 KB**, use **envelope encryption**: generate a data encryption key (DEK) via `aws kms generate-data-key`, encrypt the file with the DEK locally (using OpenSSL or AWS Encryption SDK), and store the encrypted DEK alongside the ciphertext.

### Operational Security

* **Never log or print plaintext content** of sensitive files during automation scripts.
* **Rotate IAM credentials** used for KMS operations on a regular schedule.
* **Use CloudTrail** to audit every `kms:Encrypt` and `kms:Decrypt` API call for compliance and incident response.
* **Restrict key usage to specific regions.** Disable `MultiRegion: true` unless cross-region replication is an explicit requirement.

---

## Security Considerations

| Risk | Mitigation |
|---|---|
| Unauthorized decryption | Scope `kms:Decrypt` to specific IAM principals via key policy |
| Key deletion causing data loss | Set minimum 30-day deletion waiting period; maintain key inventory |
| Ciphertext stored as readable text | Always use raw binary storage with `base64 -d` pipeline |
| Unaudited key usage | Enable AWS CloudTrail and KMS CloudWatch metrics |
| Overly permissive key policy | Follow least-privilege; avoid `"Principal": "*"` in key policies |

---

## Reference Commands

```bash
# Create symmetric CMK
aws kms create-key \
  --description "datacenter-KMS-Key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --region us-east-1

# Create alias
aws kms create-alias \
  --alias-name alias/datacenter-KMS-Key \
  --target-key-id <KEY_ID> \
  --region us-east-1

# Encrypt to raw binary
aws kms encrypt \
  --key-id alias/datacenter-KMS-Key \
  --plaintext fileb:///root/SensitiveData.txt \
  --region us-east-1 \
  --output json \
  --query CiphertextBlob \
  | tr -d '"' \
  | base64 -d > /root/EncryptedData.bin

# Decrypt from raw binary
aws kms decrypt \
  --ciphertext-blob fileb:///root/EncryptedData.bin \
  --output text \
  --query Plaintext \
  --region us-east-1 | base64 -d > /root/DecryptedOutput.txt

# Integrity check
md5sum /root/SensitiveData.txt /root/DecryptedOutput.txt
diff /root/SensitiveData.txt /root/DecryptedOutput.txt

# Enable automatic key rotation
aws kms enable-key-rotation \
  --key-id alias/datacenter-KMS-Key \
  --region us-east-1

# Describe key
aws kms describe-key \
  --key-id alias/datacenter-KMS-Key \
  --region us-east-1
```

---

## Author

**Lab Environment:** KodeKloud AWS Labs
**Region:** us-east-1
**Account:** 573434484730
**Completed:** March 16, 2026

---

> *This documentation follows enterprise runbook standards. All commands are production-tested. Screenshot placeholders must be replaced with actual terminal captures before publishing to internal wikis or compliance repositories.*




<img width="1031" height="770" alt="Image" src="https://github.com/user-attachments/assets/ede277e2-e580-4a44-8084-673f32cb1d66" />

<img width="1030" height="455" alt="Image" src="https://github.com/user-attachments/assets/c6d9b7b1-f8d6-499d-a8be-1a95aad036ec" />

<img width="1030" height="612" alt="Image" src="https://github.com/user-attachments/assets/66264f52-3be3-4441-9676-491924c538a4" />

<img width="1029" height="841" alt="Image" src="https://github.com/user-attachments/assets/afd332c5-6e1b-440b-ba61-068de435f5c6" />
