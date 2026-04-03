# AWS IAM Policy Attachment via CLI: Identity-Driven Access Management

> **Attaching a customer-managed IAM policy to an IAM user using the AWS CLI with full identity verification, ARN resolution, and attachment validation.**

---

## Table of Contents

- [Project Overview](#project-overview)
- [Problem Statement](#problem-statement)
- [Objective](#objective)
- [Tools and Technologies](#tools-and-technologies)
- [Resources Used](#resources-used)
- [Architecture and Workflow](#architecture-and-workflow)
- [Implementation Steps](#implementation-steps)
  - [Step 1: Verify AWS Identity](#step-1-verify-aws-identity)
  - [Step 2: Retrieve IAM Policy ARN](#step-2-retrieve-iampolicy-arn)
  - [Step 3: Attach Policy to IAM User](#step-3-attach-policy-to-iam-user)
  - [Step 4: Verify Policy Attachment](#step-4-verify-policy-attachment)
- [Results and Outcome](#results-and-outcome)
- [Security and Best Practices](#security-and-best-practices)
- [Operational Considerations](#operational-considerations)
- [Troubleshooting and Edge Cases](#troubleshooting-and-edge-cases)
- [Lessons Learned](#lessons-learned)

---

## Project Overview

This document details the end-to-end process of attaching an existing customer-managed AWS IAM policy to an existing IAM user using the AWS CLI. The workflow follows a structured **identity verification, policy resolution, attachment, and validation** pattern, ensuring auditability and correctness at every stage.

All operations were scoped exclusively to the **`us-east-1`** region and performed under a single authenticated IAM identity, adhering to the principle of least privilege throughout.

---

## Problem Statement

In enterprise AWS environments, IAM permissions are frequently managed through customer-managed policies rather than inline or AWS-managed policies. This allows for centralized policy governance, reuse across multiple principals, and version-controlled permission sets.

A common operational requirement is attaching a pre-defined policy to a specific IAM user, ensuring that user inherits the correct permission boundaries. Without a repeatable, CLI-driven approach, this process is error-prone when executed through the console, especially at scale or in automated pipelines.

**The challenge:** Attach a specific customer-managed IAM policy (`iampolicy_rose`) to a designated IAM user (`iamuser_rose`) with full pre-flight identity verification and post-execution confirmation, all without creating new IAM resources.

---

## Objective

- Verify the active AWS account and calling identity before making any changes
- Programmatically resolve the ARN of the target IAM policy by name
- Attach the policy to the target IAM user using the resolved ARN
- Validate successful attachment by querying the user's currently attached policies

---

## Tools and Technologies

| Tool | Purpose |
|------|---------|
| **AWS IAM** | Identity and access management service for managing users and policies |
| **AWS CLI** | Command-line interface for programmatic interaction with AWS services |
| **AWS STS** | Security Token Service, used to verify the active caller identity |

---

## Resources Used

| Resource Type | Name |
|--------------|------|
| IAM User | `iamuser_rose` |
| IAM Policy | `iampolicy_rose` |
| AWS Account ID | `723004723627` |
| AWS Region | `us-east-1` |

---

## Architecture and Workflow

The implementation follows a four-phase execution model:

```
[1] Verify Identity (STS)
        |
        v
[2] Resolve Policy ARN (IAM list-policies)
        |
        v
[3] Attach Policy to User (IAM attach-user-policy)
        |
        v
[4] Confirm Attachment (IAM list-attached-user-policies)
```

Each phase produces observable output used as input or validation for the next, creating a traceable audit trail through the CLI session.

---

## Implementation Steps

### Step 1: Verify AWS Identity

**Intent:** Before executing any IAM modifications, confirm that the active CLI session is authenticated under the correct account and identity. This guards against accidental changes in the wrong account or region.

**Command:**
```bash
aws sts get-caller-identity
```

**Expected Output:**
```json
{
    "UserId": "AIDA2QVTQSGV2I63UL6AY",
    "Account": "723004723627",
    "Arn": "arn:aws:iam::723004723627:user/kk_labs_user_926814"
}
```

**What to verify:**
- The `Account` value matches the target AWS account
- The `Arn` reflects the expected IAM identity
- The region in the CLI prompt is `us-east-1`

> **Operational Note:** Skipping identity verification is a common source of production incidents. Always confirm calling context before executing write operations against IAM.

**Screenshot: Caller identity confirmed via AWS STS**

![Step 1 - Verify AWS Identity](https://github.com/user-attachments/assets/db3e9d50-d012-4996-b5a0-f138450035dd)

*The `aws sts get-caller-identity` command confirms the active IAM identity, AWS account, and authenticated user ARN prior to any modifications.*

---

### Step 2: Retrieve IAM Policy ARN

**Intent:** Rather than hardcoding the policy ARN, programmatically resolve it by policy name. This approach is more resilient to account-level differences and eliminates manual ARN transcription errors.

**Command:**
```bash
aws iam list-policies \
  --scope Local \
  --query "Policies[?PolicyName=='iampolicy_rose'].Arn" \
  --output text
```

**Flags explained:**
- `--scope Local` restricts results to customer-managed policies, excluding AWS-managed policies
- `--query` applies a JMESPath filter to extract only the ARN of the matching policy
- `--output text` returns clean, parseable output without JSON formatting

**Expected Output:**
```
arn:aws:iam::723004723627:policy/iampolicy_rose
```

> **Best Practice:** Always resolve policy ARNs dynamically by name. Hardcoding ARNs introduces a dependency on account IDs and reduces portability of automation scripts.

> **Edge Case:** If no output is returned, the policy does not exist in the account or is an AWS-managed policy (try removing `--scope Local` to confirm). An empty result must be investigated before proceeding.

**Screenshot: Policy ARN resolved from customer-managed policies**

![Step 2 - Retrieve IAM Policy ARN](https://github.com/user-attachments/assets/31eb591b-2e09-40b2-a741-0175147ba6e9)

*The `aws iam list-policies` command with a JMESPath query filter confirms the existence of `iampolicy_rose` and returns its fully qualified ARN.*

---

### Step 3: Attach Policy to IAM User

**Intent:** Attach the customer-managed policy to the target IAM user using the ARN resolved in the previous step. A successful execution produces no output, which is standard AWS CLI behavior for write operations.

**Command:**
```bash
aws iam attach-user-policy \
  --user-name iamuser_rose \
  --policy-arn arn:aws:iam::723004723627:policy/iampolicy_rose
```

**Parameters:**
- `--user-name` specifies the target IAM user
- `--policy-arn` specifies the fully qualified ARN of the policy to attach

**Expected behavior:** The command returns with no output and exit code `0`, indicating success. The absence of an error message is the confirmation of a successful API call.

> **Risk:** Attaching a policy with overly broad permissions to a user violates least-privilege principles. Always review the policy document before attachment in production environments.

> **Operational Note:** A single IAM user can have a maximum of **10 managed policies** attached directly. If this limit is reached, the command will fail with a `LimitExceededException`. Consider using IAM groups for scalable policy management.

**Screenshot: Policy successfully attached to IAM user**

![Step 3 - Attach Policy to IAM User](https://github.com/user-attachments/assets/de834476-2d35-48c5-b547-2a30dbfe5d16)

*The `aws iam attach-user-policy` command executes silently with no error output, confirming that the policy was successfully attached to `iamuser_rose`.*

---

### Step 4: Verify Policy Attachment

**Intent:** Confirm that the policy attachment was successful by querying the list of policies attached to the target user. This is a critical validation step that closes the loop on the operation.

**Command:**
```bash
aws iam list-attached-user-policies \
  --user-name iamuser_rose
```

**Expected Output:**
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "iampolicy_rose",
            "PolicyArn": "arn:aws:iam::723004723627:policy/iampolicy_rose"
        }
    ]
}
```

**What to verify:**
- `PolicyName` matches `iampolicy_rose`
- `PolicyArn` matches the ARN resolved in Step 2
- The `AttachedPolicies` array is not empty

> **Best Practice:** Always run a read-after-write verification following any IAM mutation. IAM changes are eventually consistent across regions, but within a single region the change is typically reflected immediately.

**Screenshot: Policy attachment confirmed for IAM user**

![Step 4 - Verify Policy Attachment](https://github.com/user-attachments/assets/3a8d52ae-1a06-4ad8-bd00-86af7127f838)

*The `aws iam list-attached-user-policies` command returns `iampolicy_rose` in the `AttachedPolicies` list, confirming the successful end-to-end attachment of the policy to `iamuser_rose`.*

---

## Results and Outcome

| Objective | Status |
|-----------|--------|
| Active IAM identity verified via AWS STS | Completed |
| Customer-managed policy ARN resolved dynamically | Completed |
| Policy attached to target IAM user | Completed |
| Attachment confirmed via read-after-write validation | Completed |

The IAM user `iamuser_rose` now inherits all permissions defined in the `iampolicy_rose` customer-managed policy. The operation was completed exclusively through the AWS CLI with no console interaction, making it fully automatable and auditable.

---

## Security and Best Practices

- **Least Privilege Enforced:** No new IAM users or policies were created. Only a pre-existing policy was attached to a pre-existing user, minimizing the blast radius of the operation.
- **Dynamic ARN Resolution:** The policy ARN was resolved programmatically rather than hardcoded, reducing the risk of ARN transcription errors.
- **Pre-flight Identity Check:** Caller identity was verified before any write operations, preventing accidental changes to unintended accounts.
- **Read-after-Write Validation:** Every mutation was followed by a read verification, ensuring the expected state was achieved.
- **Region-Scoped Execution:** All commands were executed within `us-east-1`, avoiding unintended cross-region actions.
- **No Inline Policies Used:** Customer-managed policies are preferred over inline policies as they are reusable, versionable, and auditable.

---

## Operational Considerations

- **IAM Propagation:** While IAM changes are typically immediate within a region, allow a short propagation window before expecting policies to take effect in downstream services.
- **Policy Versioning:** Customer-managed policies support versioning. Before attaching, confirm the correct version is set as the default using `aws iam get-policy`.
- **Automation Suitability:** This workflow is fully scriptable. The pattern of resolving an ARN by name and passing it to an attach command is a common pattern in infrastructure automation and CI/CD pipelines.
- **IAM Groups as an Alternative:** For managing permissions across multiple users, attaching policies to IAM groups (rather than individual users) is the recommended approach for scalability and auditability.

---

## Troubleshooting and Edge Cases

| Scenario | Likely Cause | Resolution |
|----------|-------------|------------|
| `list-policies` returns empty output | Policy does not exist or is AWS-managed | Remove `--scope Local` flag to confirm; verify the policy name spelling |
| `attach-user-policy` returns `NoSuchEntityException` | User or policy ARN does not exist | Verify user with `aws iam get-user` and policy ARN with `list-policies` |
| `attach-user-policy` returns `LimitExceededException` | User has reached 10 attached policy limit | Detach unused policies or migrate user to an IAM group |
| `list-attached-user-policies` returns empty `AttachedPolicies` | Attachment did not succeed or wrong user name queried | Re-run the attach command; confirm the user name is correct |
| `AccessDeniedException` on any command | Calling identity lacks required IAM permissions | Verify the calling user has `iam:AttachUserPolicy`, `iam:ListPolicies`, and `iam:ListAttachedUserPolicies` |

---

## Lessons Learned

- **Always verify identity first.** A simple `get-caller-identity` call at the beginning of any IAM session prevents costly mistakes caused by operating in the wrong account or with the wrong credentials.
- **Resolve ARNs dynamically.** Using JMESPath queries with `--query` to extract ARNs by name makes CLI workflows portable and less brittle than hardcoded identifiers.
- **Silent success is valid.** AWS CLI write operations that return no output and exit with code `0` have succeeded. Do not interpret the absence of output as an error.
- **Validate every mutation.** A read-after-write check is a lightweight but essential safeguard that catches silent failures and confirms expected system state.
- **Scope queries appropriately.** Using `--scope Local` in `list-policies` reduces noise by filtering out the hundreds of AWS-managed policies, making output directly actionable.






















# AWS IAM User Policy Attachment (AWS CLI)

## 📌 Project Overview

- This project demonstrates how to securely attach an existing AWS IAM policy to an existing IAM user using the AWS CLI.
- The task validates identity, locates the policy, attaches it to the user, and verifies successful attachment, following IAM best practices.

- All actions were executed exclusively in the `us-east-1` region.

## 🎯 Objective

- Verify active AWS identity

- Locate an existing IAM policy

- Attach the policy to an IAM user

- Confirm successful policy attachment

## 🛠️ Tools & Technologies

`AWS IAM`

`AWS CLI`

`AWS STS`

## 📂 Resources Used

| Resource Type | Name |
|--------------|------|
| IAM User     | `iamuser_rose` |
| IAM Policy   | `iampolicy_rose` |
| AWS Region   | `us-east-1` |

##  Implementation Steps

### Step 1: Verify AWS Identity

- Ensured the correct AWS account and IAM identity were active before making changes.

- `aws sts get-caller-identity`

📸 Screenshot:
<img width="1027" height="563" alt="image" src="https://github.com/user-attachments/assets/db3e9d50-d012-4996-b5a0-f138450035dd" />


### Step 2: Retrieve IAM Policy ARN

- Queried IAM to confirm the policy exists and retrieved its ARN.

- `aws iam list-policies \`
  -  `--scope Local \`
  -  `--query "Policies[?PolicyName=='iampolicy_rose'].Arn" \`
  -  `--output text`

📸 Screenshot:
<img width="1031" height="724" alt="image" src="https://github.com/user-attachments/assets/31eb591b-2e09-40b2-a741-0175147ba6e9" />


### Step 3: Attach Policy to IAM User

- Attached the IAM policy to the target user.

- `aws iam attach-user-policy \`
  -  `--user-name iamuser_rose \`
  -  `--policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/iampolicy_rose`

📸 Screenshot:
<img width="1035" height="711" alt="image" src="https://github.com/user-attachments/assets/de834476-2d35-48c5-b547-2a30dbfe5d16" />


### Step 4: Verify Policy Attachment

- Confirmed the policy was successfully attached to the IAM user.

- `aws iam list-attached-user-policies \`
  -  `--user-name iamuser_rose`

📸 Screenshot:
<img width="1035" height="633" alt="image" src="https://github.com/user-attachments/assets/3a8d52ae-1a06-4ad8-bd00-86af7127f838" />

## ✅ Result

- IAM user identity verified

- Policy successfully located

- Policy attached to user

- Attachment confirmed via AWS CLI

- The IAM user now inherits all permissions defined in the attached policy.

## 🔐 Security & Best Practices

- Least-privilege access model followed

- No new IAM users or policies created

- Changes verified immediately after execution

- Region-restricted execution
