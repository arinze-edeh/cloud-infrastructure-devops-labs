# identity-and-security

> **AWS Identity, Access Management, and Data Protection** | IAM | KMS | EC2 | S3 | CLI-Driven | us-east-1

---

## Overview

This directory covers AWS identity and access control implementations across six production-aligned labs. The work spans IAM primitives (users, groups, policies, roles), instance profile-based authentication for EC2 workloads, and KMS-based symmetric encryption with cryptographic integrity verification.

All tasks were executed via AWS CLI against live AWS accounts using temporary credentials, mirroring least-privilege, audit-friendly patterns expected in regulated production environments.

---

## Directory Structure

```
identity-and-security/
├── ec2-s3-iam-integration/          # EC2 instance profile with scoped S3 access
├── iam-group-creation/              # IAM group provisioning and verification
├── iam-readonly-ec2-policy/         # Custom read-only EC2 IAM policy via CLI
├── iam-roles-ec2/                   # IAM role with EC2 trust relationship
├── iam-user-creation/               # IAM user provisioning
├── iam-user-policy-attachment/      # Policy attachment to existing IAM user
└── kms-data-encryption-workflow/    # KMS CMK creation, encryption, and decryption
```

---

## Project Summaries

---

### [EC2 to S3 Integration via IAM Role and Instance Profile](./ec2-s3-iam-integration/)

**Quick Summary:** Configured a running EC2 instance to access a private S3 bucket using IAM instance profile authentication, eliminating static credentials entirely from the instance.

**Purpose:** Replace hardcoded AWS credentials on a production EC2 instance with short-lived, automatically rotated credentials sourced from the instance metadata service (IMDS).

**Approach:** Provisioned a private S3 bucket with all four public access block flags enforced. Authored a least-privilege IAM policy scoped to `PutObject`, `GetObject`, and `ListBucket` on the specific bucket ARN. Created an IAM role with an EC2 trust relationship, bound the policy, and associated the role to the running instance via an instance profile. SSH access was bootstrapped using EC2 Instance Connect to avoid pre-existing key material dependencies.

**Outcome:** End-to-end credential-free S3 access verified from inside the EC2 instance. File upload and bucket listing confirmed via `aws s3 cp` and `aws s3 ls` with zero static credentials present anywhere on disk.

**Key Decisions:**
- Used `sleep 15` before instance profile association to account for IAM eventual consistency; documented a polling-loop alternative for automated pipelines
- Both bucket-level and object-level ARN entries included in the policy to satisfy `s3:ListBucket` scope requirements
- EC2 Instance Connect used for key injection rather than manual `authorized_keys` management

---

### [AWS KMS Data Encryption and Decryption at Rest](./kms-data-encryption-workflow/)

**Quick Summary:** Implemented symmetric file encryption using a customer-managed KMS key, with a critical resolution for binary ciphertext storage that unblocked automated validator decryption.

**Purpose:** Encrypt a sensitive data file at rest using AWS KMS, store the ciphertext as a raw binary blob, and verify cryptographic round-trip integrity using MD5 checksums and `diff`.

**Approach:** Created a `SYMMETRIC_DEFAULT` CMK with a human-readable alias. Encrypted the source file using the `fileb://` prefix and resolved a critical encoding mismatch: the default `--output text` path saves base64 ASCII text, which causes `InvalidCiphertextException` when passed back through `fileb://` at decryption time. The fix pipes the JSON response through `tr -d '"' | base64 -d` before writing to disk, producing a valid 177-byte raw binary blob. Integrity verified via `diff` (empty output) and matching MD5 hashes.

**Outcome:** Full encrypt-decrypt cycle completed. Automated validator passed. Binary file type confirmed via `xxd` hex dump showing the KMS ciphertext header (`0102 0200`).

**Key Decisions:**
- Alias creation decouples scripts from raw key UUIDs, enabling key rotation without downstream changes
- MD5 used exclusively for data integrity verification, not security hashing
- Envelope encryption documented as the correct approach for files exceeding the 4 KB KMS direct-encryption limit

---

### [IAM Group Creation](./iam-group-creation/)

**Quick Summary:** Created and verified an IAM group using AWS CLI with temporary credentials, establishing a repeatable group provisioning pattern.

**Purpose:** Demonstrate programmatic IAM group management as a foundation for team-based access control.

**Approach:** Authenticated via temporary credentials using `showcreds`, created the group with `aws iam create-group`, and verified existence and zero-membership state with `aws iam get-group`.

**Outcome:** Group `iamgroup_ammar` created and confirmed. Zero users assigned, validating a clean provisioning baseline ready for role and policy attachment.

---

### [IAM Read-Only EC2 Policy](./iam-readonly-ec2-policy/)

**Quick Summary:** Authored and deployed a custom least-privilege IAM policy granting read-only EC2 visibility via CLI, scoped to `DescribeInstances`, `DescribeImages`, and `DescribeSnapshots`.

**Purpose:** Provide operational read-only EC2 access without granting write, modify, or delete permissions.

**Approach:** Defined the policy document as a local JSON file using a heredoc, created the policy with `aws iam create-policy`, and verified its ARN using a filtered `list-policies` query.

**Outcome:** Policy `iampolicy_kirsty` created and confirmed via ARN lookup. Demonstrates least-privilege scoping at the action level rather than relying on AWS-managed policies.

---

### [IAM Role for EC2](./iam-roles-ec2/)

**Quick Summary:** Created an IAM role with an explicit EC2-only trust relationship and attached a customer-managed policy, following production-grade IAM role design patterns.

**Purpose:** Enable EC2 workloads to assume an IAM role for temporary, automatically rotated credentials, replacing long-lived access keys.

**Approach:** Verified active identity with `aws sts get-caller-identity` before making changes. Defined a trust policy scoped to `ec2.amazonaws.com` only (no wildcard principals, no cross-account trust). Retrieved the policy ARN dynamically via a `list-policies` query to avoid hard-coding. Attached the policy and validated with `list-attached-role-policies`.

**Outcome:** Role `iamrole_kareem` created, policy attached, and attachment confirmed. Dynamic ARN resolution pattern is CI/CD-portable and eliminates environment-specific coupling.

---

### [IAM User Creation](./iam-user-creation/)

**Quick Summary:** Provisioned an IAM user via AWS CLI and confirmed creation through a filtered `list-users` query.

**Purpose:** Establish a baseline IAM user as a building block for downstream policy and group attachment workflows.

**Approach:** Confirmed active region (`us-east-1`), created the user with `aws iam create-user`, and validated presence in `aws iam list-users` output.

**Outcome:** User `iamuser_jim` created and verified. No access keys or policies attached, maintaining a minimal provisioning footprint.

---

### [IAM User Policy Attachment](./iam-user-policy-attachment/)

**Quick Summary:** Attached an existing customer-managed IAM policy to an existing IAM user, with identity verification and post-attachment confirmation at each step.

**Purpose:** Grant an IAM user a defined permission set by attaching a pre-existing policy, following a four-phase verify-locate-attach-confirm workflow.

**Approach:** Verified caller identity, retrieved the policy ARN dynamically with a scoped `list-policies` query, attached via `attach-user-policy`, and confirmed with `list-attached-user-policies`.

**Outcome:** Policy `iampolicy_rose` successfully attached to `iamuser_rose`. All four validation steps passed with no errors.

---

## Technologies and Tools

| Category | Tools |
|---|---|
| Identity and Access | AWS IAM (users, groups, roles, policies, instance profiles) |
| Encryption | AWS KMS (CMK, aliases, symmetric encryption, `fileb://` binary handling) |
| Compute and Storage | Amazon EC2, Amazon S3 |
| CLI and Shell | AWS CLI v2, Bash, `base64`, `xxd`, `diff`, `md5sum`, `ssh-keygen` |
| Authentication | EC2 Instance Connect, AWS STS, temporary IAM credentials |
| Region | `us-east-1` throughout |

---

## Key Outcomes and Skills Demonstrated

- **Credential-free EC2 authentication** via IAM instance profiles and IMDS, eliminating the static credentials anti-pattern
- **Least-privilege IAM design** at the action, resource, and principal level across policies, roles, and trust relationships
- **Binary-safe KMS encryption workflows** with correct ciphertext storage and cryptographic integrity verification
- **Dynamic ARN resolution** using CLI queries to decouple automation scripts from hardcoded identifiers
- **IAM resource lifecycle management**: users, groups, roles, and policies provisioned and verified programmatically
- **Production-aligned patterns**: identity pre-flight checks, IAM eventual consistency handling, alias-based key management, and audit-friendly CLI-only operations throughout

---

## How to Navigate

Each subdirectory contains a `README.md` with full implementation details including CLI commands, expected outputs, screenshot references, error resolutions, and security considerations.

Start with [`ec2-s3-iam-integration`](./ec2-s3-iam-integration/) for the most complete end-to-end identity and access workflow, or [`kms-data-encryption-workflow`](./kms-data-encryption-workflow/) for the encryption implementation with its critical binary encoding resolution.

The IAM primitives labs (`iam-user-creation`, `iam-group-creation`, `iam-readonly-ec2-policy`, `iam-roles-ec2`, `iam-user-policy-attachment`) are self-contained and can be read independently or as a sequence covering the full IAM resource hierarchy.

---
