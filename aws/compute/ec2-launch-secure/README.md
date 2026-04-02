# [EC2 Key Pair Provisioning via AWS CLI](https://github.com/arinze-edeh/cloud-infrastructure-devops-labs)

> **Secure SSH credential provisioning for EC2 access using AWS CLI with RSA key pairs and Linux permission hardening**

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Objectives](#objectives)
- [Tools and Services Used](#tools-and-services-used)
- [Environment Details](#environment-details)
- [Architecture and Access Flow](#architecture-and-access-flow)
- [Implementation](#implementation)
  - [Step 1: Confirm AWS CLI Region Context](#step-1-confirm-aws-cli-region-context)
  - [Step 2: Create EC2 Key Pair via AWS CLI](#step-2-create-ec2-key-pair-via-aws-cli)
  - [Step 3: Harden Private Key Permissions](#step-3-harden-private-key-permissions)
  - [Step 4: Verify Key Pair Registration in AWS](#step-4-verify-key-pair-registration-in-aws)
- [Validation Summary](#validation-summary)
- [Key Decisions](#key-decisions)
- [Errors and Resolutions](#errors-and-resolutions)
- [Security Best Practices](#security-best-practices)
- [Lessons Learned](#lessons-learned)
- [Real-World Relevance](#real-world-relevance)
- [Skills Demonstrated](#skills-demonstrated)

---

## Overview

This lab demonstrates the creation and secure management of an Amazon EC2 key pair using the AWS CLI. Key pairs form the foundational authentication mechanism for SSH access to EC2 instances. Provisioning them correctly via CLI reflects real-world DevOps automation workflows where interactive console actions are replaced with repeatable, scriptable operations suitable for pipelines and infrastructure-as-code environments.

---

## Problem Statement

EC2 instances launched without a pre-existing key pair cannot be accessed via SSH after deployment. In automated and production workflows, engineers must provision key pairs programmatically before instance launch, store the private key securely on the controlling host, and confirm the public key is registered in the target AWS region. Failure to apply correct file permissions on the `.pem` file results in SSH rejecting the credential entirely.

---

## Objectives

- Create an RSA-based EC2 key pair using the AWS CLI
- Extract and store the private key material locally as a `.pem` file
- Apply strict Linux file permissions (`chmod 400`) to protect the private key
- Verify local file permissions and confirm AWS-side key pair registration
- Establish credentials ready for use in EC2 instance provisioning

---

## Tools and Services Used

| Tool / Service | Purpose |
|---|---|
| **AWS EC2** | Key pair creation and registration |
| **AWS CLI** | CLI-driven provisioning and verification |
| **Linux Shell** | File permissions management and verification |
| **IAM** | Pre-configured lab credentials for authenticated API calls |

---

## Environment Details

| Parameter | Value |
|---|---|
| **AWS Region** | `us-east-1` |
| **Key Pair Name** | `xfusion-kp` |
| **Key Type** | RSA |
| **Output Format** | `.pem` (PEM-encoded private key) |
| **Local File Permissions** | `400` (owner read-only) |

---

## Architecture and Access Flow

```
Engineer (CLI)
     |
     | aws ec2 create-key-pair
     v
AWS EC2 Key Pairs API (us-east-1)
     |
     |-- Public Key --> Stored in AWS (KeyPairId: key-0e75af6c25d206d03)
     |-- Private Key --> Returned once, extracted to xfusion-kp.pem
     v
Local Filesystem
     |
     | chmod 400 xfusion-kp.pem
     v
Secured .pem file (owner read-only)
     |
     | ssh -i xfusion-kp.pem ec2-user@<instance-ip>
     v
EC2 Instance (SSH Access)
```

---

## Implementation

### Step 1: Confirm AWS CLI Region Context

Before provisioning any resource, confirm the active AWS CLI profile and region context. This prevents accidental resource creation in unintended regions, which is a common source of access failures when engineers later attempt to launch instances in a different region than where the key pair was registered.

The shell prompt confirms the active region as `us-east-1`, which is the target deployment region for this lab.

> **Screenshot 1: AWS CLI region context confirmed before key pair creation**

![Step 1 - Region Context](https://github.com/user-attachments/assets/f0f4b053-6032-4924-9152-d2b329c1f20d)

---

### Step 2: Create EC2 Key Pair via AWS CLI

Create an RSA key pair named `xfusion-kp` and redirect the raw private key material directly into a local `.pem` file using a single CLI command.

```bash
aws ec2 create-key-pair \
  --key-name xfusion-kp \
  --key-type rsa \
  --query 'KeyMaterial' \
  --output text > xfusion-kp.pem
```

**Command breakdown:**

- `--key-name xfusion-kp` sets the key pair name registered in AWS EC2
- `--key-type rsa` specifies RSA as the encryption algorithm (2048-bit by default)
- `--query 'KeyMaterial'` extracts only the private key string from the full JSON response
- `--output text` strips JSON wrapping so the raw PEM content is written cleanly to disk
- `> xfusion-kp.pem` redirects the private key into a local file

**Critical note:** AWS returns the private key material **only once** at creation time. If the file is lost or overwritten before permissions are secured, the key pair must be deleted and recreated. There is no mechanism to retrieve private key material after the initial API response.

> **Screenshot 2: Key pair creation command executed successfully**

![Step 2 - Key Pair Creation](https://github.com/user-attachments/assets/650da4fc-6985-47d3-aa1f-5526b7674019)

---

### Step 3: Harden Private Key Permissions

Apply `chmod 400` to restrict the `.pem` file to owner-only read access. SSH clients enforce this requirement and will explicitly reject private keys that are accessible to group or other users, returning an `Unprotected private key file` error.

```bash
chmod 400 xfusion-kp.pem
```

**Why `400` and not `600`:**

`400` (read-only for owner) is the production standard for SSH private keys. `600` (read and write for owner) is sometimes used but grants unnecessary write access. In automation environments where keys are injected into containers or CI runners, `400` prevents accidental key modification by other processes running as the same user.

Verify the permission change was applied correctly:

```bash
ls -l xfusion-kp.pem
```

**Expected output:**

```
-r-------- 1 root root 1679 Jan 30 03:29 xfusion-kp.pem
```

The `-r--------` permission string confirms that only the file owner can read the key. The `1679` byte size is consistent with a valid RSA 2048-bit PEM private key.

> **Screenshot 3: chmod 400 applied and permissions verified with ls -l**

![Step 3 - Permission Hardening and Verification](https://github.com/user-attachments/assets/4dd0aabd-ebcf-45c8-a0c5-440fe9b40481)

---

### Step 4: Verify Key Pair Registration in AWS

Confirm that the key pair is registered in the AWS EC2 service for the target region. This step validates that the public key was successfully stored AWS-side and that the key pair is available for use when launching new EC2 instances.

```bash
aws ec2 describe-key-pairs --key-names xfusion-kp
```

**Expected output:**

```json
{
    "KeyPairs": [
        {
            "KeyPairId": "key-0e75af6c25d206d03",
            "KeyType": "rsa",
            "Tags": [],
            "CreateTime": "2026-01-30T03:29:05.314Z",
            "KeyName": "xfusion-kp",
            "KeyFingerprint": "55:6d:5c:31:de:f9:16:e5:ae:e7:e7:8f:cb:58:03:a8:e9:e8:81:53"
        }
    ]
}
```

**Output fields to verify:**

- `KeyName` confirms the correct key is registered
- `KeyType: "rsa"` confirms the algorithm
- `KeyPairId` is the unique AWS identifier for the key pair, useful for referencing in CloudFormation, Terraform, or IAM policies
- `KeyFingerprint` allows cross-verification if needed between the stored public key and the local private key

> **Screenshot 4: Key pair confirmed registered in AWS EC2 with metadata**

<img width="973" height="743" alt="image" src="https://github.com/user-attachments/assets/61205384-a513-4955-a4c1-de3ab3fdd1d2" />


---

## Validation Summary

| Validation Check | Expected Result | Status |
|---|---|---|
| Key pair created via CLI | No error returned, `.pem` file written | Confirmed |
| Private key file size | ~1679 bytes (RSA 2048-bit PEM) | Confirmed |
| File permissions | `-r--------` (400) | Confirmed |
| AWS key pair registration | `KeyName: xfusion-kp` returned by `describe-key-pairs` | Confirmed |
| Key type | `rsa` | Confirmed |

---

## Key Decisions

**RSA over ED25519:** RSA was selected for compatibility. While ED25519 offers stronger security properties and smaller key sizes, RSA 2048-bit remains the default for AWS-managed key pairs and is universally supported across EC2 instance types, jump hosts, and SSH client versions in mixed environments.

**`--query 'KeyMaterial' --output text` pattern:** Using `--query` to extract only the private key material and `--output text` to strip JSON formatting ensures the `.pem` file contains clean PEM content without extra characters that would corrupt the key and cause SSH authentication failures.

**`chmod 400` over `chmod 600`:** Read-only permissions follow the principle of least privilege. Private keys should never be modified after creation. `400` enforces this at the filesystem level.

**Immediate permission hardening post-creation:** Applying `chmod 400` immediately after file creation, before any other operations, ensures there is no window during which the key file exists with permissive default permissions.

---

## Errors and Resolutions

**SSH rejects key with "Unprotected private key file" error**

This occurs if `chmod 400` was not applied or the permissions were later relaxed. SSH clients will refuse to use private keys that are readable by group or other users.

Resolution: Re-apply `chmod 400 xfusion-kp.pem` before attempting SSH connections.

---

**`aws ec2 create-key-pair` returns empty `.pem` file**

This can occur if `--output text` was omitted and the raw JSON was written to the file, or if `--query 'KeyMaterial'` was not included, resulting in the full JSON object being stored instead of the private key string.

Resolution: Delete the corrupted file, delete the key pair from AWS with `aws ec2 delete-key-pairs --key-name xfusion-kp`, and rerun the creation command with the correct flags.

---

**Key pair not found in `describe-key-pairs` after creation**

Most commonly caused by a region mismatch: the key pair was created in one region but the verification command was run against a different default region.

Resolution: Always pass `--region us-east-1` explicitly in both commands, or confirm the active profile region with `aws configure get region` before executing any commands.

---

## Security Best Practices

- **Never commit `.pem` files to version control.** Add `*.pem` to `.gitignore` at the repository root as a baseline control in all infrastructure repositories.
- **Store private keys in a secrets manager for team environments.** AWS Secrets Manager or HashiCorp Vault should be used when keys need to be shared or rotated across teams.
- **Restrict key access to the owner only.** `chmod 400` is the minimum required by SSH and reflects the principle of least privilege at the filesystem level.
- **Use separate key pairs per environment.** A single key pair shared across development, staging, and production creates lateral movement risk if the key is compromised.
- **Rotate keys periodically.** In production environments, establish a key rotation schedule. Decommissioned key pairs should be deleted from AWS EC2 to reduce the attack surface.
- **Audit key pair usage with CloudTrail.** `ec2:CreateKeyPair` and `ec2:DescribeKeyPairs` API calls are logged in CloudTrail. Enable CloudTrail in all regions and set alerts for unexpected key pair creation events.
- **Prefer EC2 Instance Connect or AWS Systems Manager Session Manager** for ephemeral access in environments where managing long-lived SSH keys adds operational overhead.

---

## Lessons Learned

- **Private key material is returned exactly once.** The AWS API does not store the private key. The moment the `create-key-pair` response is returned is the only opportunity to capture it. Treating this step as a critical handoff point in any provisioning workflow is essential.
- **Permission hardening must be immediate.** On shared systems, a newly created file with default permissions is briefly accessible to other processes or users running as the same account. Applying `chmod 400` immediately after creation eliminates this window.
- **`ls -l` verification is not optional in production documentation.** Visually confirming `-r--------` in the output makes intent auditable and provides a checkpoint for peer review or compliance requirements.
- **Region context must always be validated before provisioning.** A key pair created in `us-east-1` cannot be used to access an instance launched in `eu-west-1`. Confirming the active region at the start of every session prevents this class of error.

---

## Real-World Relevance

This workflow directly reflects how Cloud and DevOps Engineers provision SSH access credentials in production environments. In automated pipelines, the same CLI commands are embedded in provisioning scripts, Terraform `null_resource` blocks, or Ansible playbooks that execute key pair creation as a prerequisite step before instance launch. The pattern of extracting only the `KeyMaterial` field, writing it to a secured file, and confirming AWS-side registration is the standard approach used in enterprise AWS environments where console access is restricted in favor of CLI and IaC-driven operations.

---

## Skills Demonstrated

- AWS EC2 key pair lifecycle management
- AWS CLI resource provisioning and verification
- Linux file permissions and security hardening (`chmod`, `ls -l`)
- SSH credential management best practices
- Cloud security fundamentals (least privilege, key rotation, secrets handling)
- Infrastructure documentation to production and onboarding standards


































# Secure EC2 Access – Key Pair Creation (AWS CLI)

## Overview
This lab demonstrates how to create and securely manage an Amazon EC2 key pair using the AWS CLI.  
Key pairs are required for secure SSH access to EC2 instances and are a foundational security control in AWS environments.

---

## Objectives
- Create an EC2 key pair using RSA encryption
- Secure the private key with proper Linux permissions
- Verify key pair creation using AWS CLI
- Prepare credentials for secure EC2 access

---

## Tools & Services Used
- AWS EC2
- AWS CLI
- Linux Shell
- IAM (Preconfigured lab credentials)

---

## Environment Details
- AWS Region: `us-east-1`
- Authentication Method: SSH Key Pair
- Key Type: RSA

---

## Step 1: Confirm AWS Region

<img width="1041" height="880" alt="image" src="https://github.com/user-attachments/assets/f0f4b053-6032-4924-9152-d2b329c1f20d" />

## Step 2: Create EC2 Key Pair

- Created an RSA-based EC2 key pair named xfusion-kp using the AWS CLI

aws ec2 create-key-pair \
  --key-name xfusion-kp \
  --key-type rsa \
  --query 'KeyMaterial' \
  --output text > xfusion-kp.pem
<img width="1109" height="948" alt="image" src="https://github.com/user-attachments/assets/650da4fc-6985-47d3-aa1f-5526b7674019" />


## Step 3: Secure the Private Key

- Applied strict file permissions to protect the private key as required by SSH.

chmod 400 xfusion-kp.pem
<img width="1041" height="880" alt="image" src="https://github.com/user-attachments/assets/f0f4b053-6032-4924-9152-d2b329c1f20d" />

- Verified permissions

ls -l xfusion-kp.pem

<img width="1099" height="902" alt="image" src="https://github.com/user-attachments/assets/4dd0aabd-ebcf-45c8-a0c5-440fe9b40481" />

## Step 4: Verify Key Pair Creation in AWS

- Confirmed the key pair exists in AWS.

aws ec2 describe-key-pairs --key-names xfusion-kp
<img width="1068" height="911" alt="image" src="https://github.com/user-attachments/assets/edd63c96-09dd-4acb-b4fd-d3be90c7ec9b" />


## Result

- EC2 key pair successfully created

- Private key securely stored

- Key pair verified and ready for EC2 instance launches

## Security Best Practices

- Never commit private keys to GitHub

- Restrict key permissions to the owner only

- Rotate keys periodically in production environments

## Real-World Relevance

- This workflow reflects how Cloud and DevOps Engineers securely provision access to EC2 instances in real production environments using CLI-based automation.

## Skills Demonstrated

- AWS EC2 access management

- AWS CLI usage

- Linux file permission management

- Cloud security fundamentals
