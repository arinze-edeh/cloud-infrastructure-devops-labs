# AWS IAM Custom Policy: Programmatic EC2 Read-Only Access via AWS CLI

> **Project Type:** AWS Identity and Access Management (IAM)
> **Service:** AWS IAM | Amazon EC2
> **Region:** `us-east-1`
> **Policy Name:** `iampolicy_kirsty`
> **Implementation Method:** AWS CLI (Command Line Interface)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Objectives](#objectives)
- [Environment Details](#environment-details)
- [Architecture and Policy Design](#architecture-and-policy-design)
- [Step-by-Step Implementation](#step-by-step-implementation)
  - [Step 1: Retrieve AWS Credentials](#step-1-retrieve-aws-credentials)
  - [Step 2: Create IAM Policy JSON File](#step-2-create-iam-policy-json-file)
  - [Step 3: Create IAM Policy Using AWS CLI](#step-3-create-iam-policy-using-aws-cli)
  - [Step 4: Verify Policy Creation](#step-4-verify-policy-creation)
- [Validation Checklist](#validation-checklist)
- [Best Practices and Operational Considerations](#best-practices-and-operational-considerations)
- [Risks, Edge Cases, and Troubleshooting](#risks-edge-cases-and-troubleshooting)
- [Lessons Learned](#lessons-learned)
- [Outcome](#outcome)

---

## Overview

This project demonstrates the programmatic creation of a custom AWS Identity and Access Management (IAM) policy using the AWS CLI. The policy enforces **least-privilege access** by granting read-only visibility into Amazon EC2 resources, specifically allowing users to describe EC2 instances, AMIs (Amazon Machine Images), and snapshots.

This approach follows enterprise-grade IAM design principles: policies are version-controlled as JSON artifacts, created via CLI for repeatability and auditability, and scoped to only the permissions operationally required. No write, modify, or delete permissions are included.

This pattern is applicable across real-world scenarios including:

- Onboarding read-only operational roles for junior engineers or auditors
- Enabling automated monitoring pipelines with scoped EC2 visibility
- Supporting compliance requirements that mandate strict access controls

---

## Problem Statement

The DevOps team requires a custom IAM policy that:

- Grants **read-only visibility** into EC2 resources (instances, AMIs, snapshots)
- Is **created programmatically** using the AWS CLI for automation and reproducibility
- Is **scoped** to the `us-east-1` region and a specific AWS account
- Adheres to the **principle of least privilege**, avoiding any permissive or unnecessary access

---

## Objectives

- Authenticate to AWS using temporary session credentials
- Define an IAM policy document in standard JSON format
- Create the policy using the AWS CLI `iam create-policy` command
- Confirm successful policy creation by inspecting returned metadata
- Verify policy availability in the IAM policy list using a targeted query

---

## Environment Details

| Parameter | Value |
|---|---|
| AWS Account | Provisioned AWS account |
| Region | `us-east-1` |
| Authentication Method | Temporary AWS session credentials |
| Tools Used | AWS CLI, Linux shell |
| Policy Scope | IAM Customer Managed Policy |

---

## Architecture and Policy Design

The policy document follows the IAM JSON policy language structure. It grants a single `Allow` statement over three read-only EC2 describe actions, scoped to all resources (`*`) since `Describe*` actions do not support resource-level permissions in IAM.

**Permitted Actions:**

- `ec2:DescribeInstances` - Lists and describes all EC2 instances in the account/region
- `ec2:DescribeImages` - Describes AMIs (Amazon Machine Images) available to the account
- `ec2:DescribeSnapshots` - Describes EBS snapshots accessible to the account

**Explicitly Excluded (by omission):**

- Any `ec2:Create*`, `ec2:Run*`, `ec2:Terminate*`, `ec2:Modify*`, or `ec2:Delete*` actions
- IAM, S3, VPC, RDS, or any other service permissions

This design ensures the policy is minimal, auditable, and safe to attach to service accounts, monitoring roles, or read-only operator roles.

---

## Step-by-Step Implementation

---

### Step 1: Retrieve AWS Credentials

**Action:** Display temporary AWS session credentials to confirm authentication context before executing any IAM operations.

**Command:**

```bash
showcreds
```

**Purpose:** The `showcreds` command is a shell alias available in provisioned AWS environments that outputs the active session credentials in a structured table. This confirms:

- The correct AWS account is targeted
- The session has not expired
- The correct region (`us-east-1`) is active

**Output Fields:**

| Field | Description |
|---|---|
| AWS Console URL | Direct sign-in URL scoped to the account and region |
| AWS User Name | IAM username associated with the active session |
| AWS Password | Session password (redacted in documentation) |
| AWS Session End Time | UTC timestamp marking session expiry |

> **Operational Note:** Always verify session expiry time before beginning multi-step CLI workflows. An expired session mid-execution can leave resources in a partially created state.

**Screenshot: `showcreds` command output confirming active session and credentials**

![showcreds output showing AWS session credentials including account URL, username, and session expiry time](https://github.com/user-attachments/assets/7606d269-008c-4f0a-9816-05412a6b5988)

---

### Step 2: Create IAM Policy JSON File

**Action:** Define the IAM policy document as a local JSON file using shell heredoc redirection.

**Command:**

```bash
cat <<EOF > policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:DescribeImages",
                "ec2:DescribeSnapshots"
            ],
            "Resource": "*"
        }
    ]
}
EOF
```

**Purpose:** This command writes the IAM policy JSON to a local file (`policy.json`) using a heredoc block. The `file://` prefix in the subsequent CLI command references this file directly.

**Key Design Decisions:**

- **`"Version": "2012-10-17"`** is the required and current IAM policy language version. Omitting this or using an outdated version string will result in evaluation errors.
- **`"Effect": "Allow"`** explicitly permits the listed actions. IAM denies all actions by default; only explicitly allowed actions are permitted.
- **`"Resource": "*"`** is required for `Describe*` EC2 actions because they do not support resource-level restrictions. Attempting to scope these actions to a specific ARN will result in an invalid policy.

> **Best Practice:** Store policy JSON files in version control (e.g., Git) alongside the infrastructure code that uses them. This enables policy change auditing and rollback capability.

**Screenshot: `policy.json` file contents written to disk via heredoc redirection**

![Terminal output showing the policy.json file contents being written using cat heredoc syntax](https://github.com/user-attachments/assets/5120b923-cb2b-4a57-85ee-4945fb56c245)

---

### Step 3: Create IAM Policy Using AWS CLI

**Action:** Submit the policy document to AWS IAM using the `create-policy` command.

**Command:**

```bash
aws iam create-policy \
    --policy-name iampolicy_kirsty \
    --policy-document file://policy.json
```

**Parameter Breakdown:**

| Parameter | Value | Description |
|---|---|---|
| `--policy-name` | `iampolicy_kirsty` | Unique name for the customer-managed policy within the account |
| `--policy-document` | `file://policy.json` | References the local JSON file using the `file://` URI scheme |

**Expected Response Fields:**

| Field | Description |
|---|---|
| `PolicyName` | Confirms the name as submitted |
| `PolicyId` | Unique AWS-generated identifier for the policy object |
| `Arn` | Full ARN: `arn:aws:iam::<account-id>:policy/iampolicy_kirsty` |
| `Path` | Defaults to `/` unless a custom path is specified |
| `DefaultVersionId` | Set to `v1` on creation; increments with future updates |
| `AttachmentCount` | `0` on creation; increments each time the policy is attached to a principal |
| `IsAttachable` | `true` confirms the policy can be attached to IAM users, groups, or roles |
| `CreateDate` / `UpdateDate` | UTC timestamps recorded at creation |

> **Operational Note:** The returned ARN is the stable, globally unique identifier for this policy. Capture and store this ARN for downstream automation, such as attaching the policy to an IAM role or user via `aws iam attach-role-policy`.

> **Edge Case:** If a policy with the same name already exists in the account, the `create-policy` call will fail with an `EntityAlreadyExists` error. Use `aws iam list-policies --scope Local` to check for existing policies before creation in automated workflows.

**Screenshot: `create-policy` command response confirming successful policy creation with ARN and metadata**

![Terminal output showing the full JSON response from aws iam create-policy including PolicyId, ARN, and timestamps](https://github.com/user-attachments/assets/8cb66d29-4ae3-46ab-81fb-b02f271fe667)

---

### Step 4: Verify Policy Creation

**Action:** Query the IAM policy list to confirm the policy exists and retrieve its ARN.

**Command:**

```bash
aws iam list-policies \
    --scope Local \
    --query 'Policies[?PolicyName==`iampolicy_kirsty`].Arn' \
    --output text
```

**Parameter Breakdown:**

| Parameter | Value | Description |
|---|---|---|
| `--scope Local` | `Local` | Filters to customer-managed policies only, excluding AWS-managed policies |
| `--query` | JMESPath filter | Targets the policy by name and returns only the ARN field |
| `--output text` | `text` | Returns plain text output suitable for shell scripting and piping |

**Expected Output:**

```
arn:aws:iam::854993966332:policy/iampolicy_kirsty
```

**Purpose:** This verification step independently confirms that the policy object is registered in IAM and discoverable by name. It validates that the creation step completed successfully and that the ARN is consistent with the one returned during creation.

> **Best Practice:** In production automation pipelines, this query can be used as an idempotency check before attempting to create a policy, preventing duplicate creation errors in re-entrant workflows.

**Screenshot: `list-policies` query output confirming policy ARN is present in IAM**

![Terminal output showing the verified ARN returned from aws iam list-policies filtered by policy name](https://github.com/user-attachments/assets/092cc3e3-20f5-49e3-be83-965eaea2cf0e)

---

## Validation Checklist

- [x] AWS session credentials retrieved and session expiry confirmed
- [x] `policy.json` file created with correct IAM policy syntax and version
- [x] IAM policy created successfully using `aws iam create-policy`
- [x] Correct policy name `iampolicy_kirsty` applied and confirmed in response
- [x] Read-only EC2 permissions (`DescribeInstances`, `DescribeImages`, `DescribeSnapshots`) verified
- [x] Policy ARN returned and matches expected account and path structure
- [x] Policy visible in IAM customer-managed policy list via `list-policies --scope Local`
- [x] Region confirmed as `us-east-1` throughout execution

---

## Best Practices and Operational Considerations

- **Least Privilege by Design:** This policy intentionally omits all write and mutate permissions. When designing IAM policies, always start with the minimum required actions and expand only when operationally justified.

- **Customer-Managed Over Inline Policies:** Creating a customer-managed policy (as done here) is preferred over inline policies because managed policies are reusable, independently versioned, and auditable without inspecting the principal they are attached to.

- **Policy Versioning:** AWS IAM supports up to five versions of a customer-managed policy. Use `aws iam create-policy-version` to update an existing policy. The active version can be set independently, enabling rollback without deletion.

- **Tagging IAM Resources:** For enterprise environments, apply resource tags to IAM policies during creation using `--tags` to support cost allocation, ownership tracking, and automated governance. Example: `--tags Key=Owner,Value=devops-team Key=Environment,Value=production`.

- **Automate with IaC:** For repeatable deployments, consider codifying this policy in AWS CloudFormation or Terraform rather than running CLI commands manually. CLI is appropriate for ad-hoc operations; infrastructure-as-code is preferred for persistent resources.

- **Audit with CloudTrail:** All IAM API calls, including `CreatePolicy` and `ListPolicies`, are logged in AWS CloudTrail. Enable CloudTrail in the account to maintain a full audit trail of policy creation, modification, and attachment events.

---

## Risks, Edge Cases, and Troubleshooting

**Risk: Session Expiry During Execution**
Temporary credentials expire at the `AWS Session End Time`. If a session expires between steps, subsequent CLI calls will return `ExpiredTokenException`. Always verify session time before starting and request a new session if expiry is imminent.

**Risk: Duplicate Policy Name**
Running `create-policy` twice with the same `--policy-name` will fail with `EntityAlreadyExistsException`. Add a pre-check using `list-policies --scope Local` and filter by name before execution in automated workflows.

**Edge Case: Resource-Level ARN Restriction**
Attempting to restrict `ec2:Describe*` actions to a specific resource ARN (e.g., a specific instance ID) will result in a policy validation error. AWS IAM does not support resource-level permissions for most EC2 `Describe*` actions. The `"Resource": "*"` is mandatory for these actions.

**Troubleshooting: `AccessDenied` on `create-policy`**
The executing IAM user or role must have `iam:CreatePolicy` permission. If this call fails with `AccessDenied`, verify the calling principal's attached policies include IAM write permissions, or request elevation from an account administrator.

**Troubleshooting: `MalformedPolicyDocument`**
Ensure the JSON in `policy.json` is valid before submission. Use `python3 -m json.tool policy.json` to lint the file locally. Common causes include trailing commas, mismatched brackets, or incorrect `Version` string values.

---

## Lessons Learned

- Using the `file://` URI prefix with `--policy-document` is required when referencing a local file. Omitting it and passing the raw JSON string directly requires careful shell escaping and is error-prone in production scripts.
- The `--scope Local` flag in `list-policies` is essential for filtering to customer-managed policies. Without it, the query returns hundreds of AWS-managed policies, significantly increasing response time and output volume.
- Storing the policy ARN output from `create-policy` in a shell variable immediately after creation enables seamless chaining into downstream commands such as `attach-user-policy` or `attach-role-policy` without requiring a separate lookup step.

---

## Outcome

The IAM customer-managed policy **`iampolicy_kirsty`** was successfully created in the `us-east-1` region using the AWS CLI. The policy grants secure, scoped, read-only access to Amazon EC2 resources, supporting operational visibility for the DevOps team while strictly enforcing the principle of least privilege.

The policy is registered in IAM, confirmed via ARN lookup, and is in an attachable state (`IsAttachable: true`) ready to be assigned to IAM users, groups, or roles as required.

This implementation demonstrates a repeatable, auditable, and production-aligned approach to programmatic IAM policy management in AWS.
