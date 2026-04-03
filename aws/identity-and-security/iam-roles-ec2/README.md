# AWS IAM Role for EC2 Workloads: Security-First Implementation via CLI

> **Production-grade IAM role provisioning using AWS CLI with explicit trust boundaries, least-privilege design, and full auditability.**

---

## Table of Contents

- [Overview](#overview)
- [Architecture Summary](#architecture-summary)
- [Environment and Scope](#environment-and-scope)
- [Prerequisites](#prerequisites)
- [Step 1: Identity Verification (Pre-flight Check)](#step-1-identity-verification-pre-flight-check)
- [Step 2: Define Explicit Trust Policy (EC2 Only)](#step-2-define-explicit-trust-policy-ec2-only)
- [Step 3: Create the IAM Role](#step-3-create-the-iam-role)
- [Step 4: Discover Policy ARN Dynamically](#step-4-discover-policy-arn-dynamically)
- [Step 5: Attach Policy to Role](#step-5-attach-policy-to-role)
- [Step 6: Validate Role State](#step-6-validate-role-state)
- [Security and DevOps Principles Demonstrated](#security-and-devops-principles-demonstrated)
- [Operational Considerations](#operational-considerations)
- [Troubleshooting Reference](#troubleshooting-reference)
- [Outcome](#outcome)

---

## Overview

### Problem

EC2 instances that access AWS services (S3, SSM, Secrets Manager, etc.) require a secure method of obtaining credentials. Embedding static IAM user access keys in instance configuration is a critical security anti-pattern: keys can be leaked, are difficult to rotate, and violate the principle of least privilege. In regulated and enterprise environments, this approach fails compliance requirements outright.

### Solution

IAM Roles for EC2 provide temporary, automatically rotated credentials via the instance metadata service (IMDS). The role is assumed by the EC2 service on behalf of the instance, eliminating the need for any long-lived credentials. This implementation demonstrates the full provisioning lifecycle using the AWS CLI with a security-first approach.

### What This Implementation Covers

- Pre-flight identity verification to prevent cross-account misconfigurations
- Explicit, service-scoped trust policy creation (no wildcard principals)
- IAM role provisioning with a customer-managed policy attachment
- Dynamic policy ARN resolution for automation and CI/CD portability
- Final state validation to confirm correct role configuration

---

## Architecture Summary

```
EC2 Instance
    |
    | assumes role via STS
    v
IAM Role: iamrole_kareem
    |-- Trust Policy: ec2.amazonaws.com (AssumeRole only)
    |-- Attached Policy: iampolicy_kareem (customer-managed)
         |-- Scoped permissions to required AWS services
```

Credential flow: EC2 calls `sts:AssumeRole` implicitly via the instance profile. AWS STS returns temporary credentials (Access Key, Secret Key, Session Token) with a default TTL of 1 hour, auto-rotated by the platform.

---

## Environment and Scope

| Parameter | Value |
|---|---|
| AWS Account | `472012609282` |
| Region | `us-east-1` |
| IAM Role Name | `iamrole_kareem` |
| Trusted Principal | `ec2.amazonaws.com` |
| Policy Attached | `iampolicy_kareem` (customer-managed) |
| Credential Model | Temporary STS credentials |
| CLI Profile | Default (us-east-1) |

---

## Prerequisites

- AWS CLI v2 installed and configured
- IAM user with permissions to create roles and attach policies (`iam:CreateRole`, `iam:AttachRolePolicy`, `iam:ListPolicies`)
- Customer-managed policy `iampolicy_kareem` already exists in the target account
- Shell environment with `bash` or `zsh`

---

## Step 1: Identity Verification (Pre-flight Check)

### Intent

Before any IAM resource is created, confirm the active AWS identity. This is a non-negotiable first step in production and enterprise workflows. It prevents accidental deployments into the wrong AWS account, confirms the CLI is operating as an auditable non-root IAM user, and provides a record of who initiated the change, which is required in change-management and incident-response contexts.

### Command

```bash
aws sts get-caller-identity
```

### Expected Output

```json
{
    "UserId": "AIDAW3ZRE24BOGOYWGBGV",
    "Account": "472012609282",
    "Arn": "arn:aws:iam::472012609282:user/kk_labs_user_476778"
}
```

**Screenshot: Active AWS identity confirmed via STS before any IAM operations are performed.**

![Identity Verification](https://github.com/user-attachments/assets/69fcb71f-5cb3-4aa0-9d04-9a5e35e6486f)

### Why This Matters

- **Cross-account protection:** Confirms you are operating in the correct account before making changes
- **Non-root enforcement:** Output verifies a named IAM user is active, not the root account
- **Audit trail:** The ARN provides an immutable record of who executed the deployment
- **Enterprise compliance:** Required as a pre-step in ITIL, SOC 2, and ISO 27001 change workflows

---

## Step 2: Define Explicit Trust Policy (EC2 Only)

### Intent

The trust policy is the permission boundary that defines which entities are allowed to assume the IAM role. Restricting it explicitly to `ec2.amazonaws.com` ensures no other service, user, or account can assume this role without an explicit policy change, which would itself generate an audit event.

### Command

```bash
cat <<EOF > trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

**Screenshot: Trust policy JSON written to disk, scoped exclusively to the EC2 service principal.**

![Trust Policy Creation](https://github.com/user-attachments/assets/a84053b7-ee50-411e-9352-b40fac8d2a03)

### Security Design Decisions

- **No wildcard principals (`*`):** Prevents unintended cross-service or cross-account assumption
- **No cross-account trust:** The trust boundary is limited to the current account's EC2 service
- **Single permitted action:** Only `sts:AssumeRole` is allowed; no other STS actions are granted
- **Explicit version pinning:** `2012-10-17` is the current and required policy language version

### Edge Cases and Risks

- If you need multiple trusted services (e.g., EC2 and Lambda), add them as additional entries in the `Statement` array rather than using wildcards
- Trust policies are evaluated independently of permission policies; a misconfigured trust policy will silently prevent role assumption without an obvious error message
- Verify the file was written correctly with `cat trust-policy.json` before proceeding

---

## Step 3: Create the IAM Role

### Intent

The IAM role is the identity that EC2 instances will assume. Using a role instead of an IAM user with static credentials eliminates long-lived secrets from the environment entirely. AWS automatically rotates the temporary credentials issued via the instance profile, removing the operational burden of manual key rotation.

### Command

```bash
aws iam create-role \
    --role-name iamrole_kareem \
    --assume-role-policy-document file://trust-policy.json
```

### Expected Output (abbreviated)

```json
{
    "Role": {
        "Path": "/",
        "RoleName": "iamrole_kareem",
        "RoleId": "AROAW3ZRE24BKWJGK4FV4",
        "Arn": "arn:aws:iam::472012609282:role/iamrole_kareem",
        "CreateDate": "2026-02-21T00:42:09Z",
        "AssumeRolePolicyDocument": { ... }
    }
}
```

**Screenshot: IAM role created successfully. The response confirms the role ARN, role ID, and the embedded trust policy document.**

![IAM Role Creation](https://github.com/user-attachments/assets/63ae9ea4-a359-40d0-82e1-87c73f613bd6)

### Why Roles Over IAM Users

| Concern | IAM User (Static Keys) | IAM Role (Temporary Credentials) |
|---|---|---|
| Credential lifetime | Indefinite until manually rotated | 1 hour (auto-rotated) |
| Rotation burden | Manual, error-prone | Automatic via STS |
| Blast radius on leak | Full access until key deleted | Expires automatically |
| Compliance posture | Fails most CIS Benchmarks | Meets CIS, SOC 2, PCI-DSS |
| Use with EC2/ECS/EKS | Requires embedding keys | Native integration via IMDS |

### Operational Notes

- Record the `RoleId` and `Arn` from the output; these are needed for instance profile creation (next step in a full EC2 deployment)
- The `CreateDate` provides an immutable timestamp for change-management records
- If the role already exists, this command will return an `EntityAlreadyExists` error; use `get-role` to inspect the existing role before deciding to update or proceed

---

## Step 4: Discover Policy ARN Dynamically

### Intent

Rather than hard-coding the policy ARN (which varies by account ID and environment), the ARN is resolved dynamically using a JMESPath query against the IAM policy list. This pattern makes the workflow portable across accounts and environments and is the correct approach for CI/CD pipelines, Infrastructure as Code, and shared runbooks.

### Commands

```bash
POLICY_ARN=$(aws iam list-policies \
    --query "Policies[?PolicyName=='iampolicy_kareem'].Arn" \
    --output text)

echo $POLICY_ARN
```

### Expected Output

```
arn:aws:iam::472012609282:policy/iampolicy_kareem
```

**Screenshot: Policy ARN dynamically resolved and stored in a shell variable, confirming the policy exists in the target account.**

![Dynamic Policy ARN Resolution](https://github.com/user-attachments/assets/7697860a-21b4-4ac4-bf5f-6afb910bac68)

### Engineering Rationale

- **Automation-ready:** Shell variable resolution works identically in interactive sessions, bash scripts, and CI/CD runners (GitHub Actions, Jenkins, GitLab CI)
- **Environment-agnostic:** No account ID hard-coded; the query resolves correctly across dev, staging, and production accounts
- **Fail-safe by design:** If the policy does not exist, `$POLICY_ARN` is empty. Adding a guard (`if [ -z "$POLICY_ARN" ]; then echo "Policy not found"; exit 1; fi`) prevents attaching a null ARN in automated pipelines
- **Eliminates human error:** Removes the risk of typos in a 12-digit account ID embedded in a 70-character ARN

### Best Practice Enhancement

For production scripts, add a guard clause immediately after the `echo`:

```bash
if [ -z "$POLICY_ARN" ]; then
  echo "ERROR: Policy 'iampolicy_kareem' not found. Aborting."
  exit 1
fi
```

---

## Step 5: Attach Policy to Role

### Intent

Attaching the customer-managed policy to the role grants the EC2 instance the specific permissions it needs to operate. Customer-managed policies are preferred over AWS-managed policies in production because they provide full version control, can be scoped to the minimum required actions and resources, and can be updated without affecting other roles that might use the same policy.

### Command

```bash
aws iam attach-role-policy \
    --role-name iamrole_kareem \
    --policy-arn $POLICY_ARN
```

**Screenshot: Policy attachment executed. No output on success is expected behavior from the AWS CLI for this operation.**

![Policy Attachment](https://github.com/user-attachments/assets/c4619b7b-bd8b-417f-a690-8abf6447a00b)

### Operational Considerations

- **Silent success:** `attach-role-policy` returns no output on success. This is expected AWS CLI behavior. Immediately proceed to Step 6 to validate the attachment was applied correctly.
- **Policy version limits:** AWS enforces a limit of 5 non-default versions per managed policy. In active development environments, monitor version counts to avoid `LimitExceeded` errors.
- **Multiple policy attachments:** A role can have up to 10 managed policies attached. For complex permission sets, consolidate into a single customer-managed policy rather than stacking multiple policies, which makes auditing and debugging more difficult.
- **Idempotency:** Re-running this command on an already-attached policy is safe; AWS will return success without creating a duplicate attachment.

---

## Step 6: Validate Role State

### Intent

The final validation step confirms that the full provisioning workflow completed correctly. Explicitly verifying the attached policy prevents silent failures from propagating to production workloads, ensures the correct policy version is active, and provides documentation evidence for change-management sign-off.

### Command

```bash
aws iam list-attached-role-policies --role-name iamrole_kareem
```

### Expected Output

```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "iampolicy_kareem",
            "PolicyArn": "arn:aws:iam::472012609282:policy/iampolicy_kareem"
        }
    ]
}
```

**Screenshot: Final validation confirms `iampolicy_kareem` is correctly attached to `iamrole_kareem`, completing the provisioning workflow.**

![Final Validation](https://github.com/user-attachments/assets/1d091c3a-05ff-4dee-acb0-ec347678e240)

### Validation Checklist

- `AttachedPolicies` array contains exactly one entry: `iampolicy_kareem`
- `PolicyArn` matches the ARN confirmed in Step 4
- No unexpected policies are attached (indicates a clean, minimal role)
- Role name matches the intended target (`iamrole_kareem`)

### Additional Verification (Optional)

For a complete role audit, run the following commands to confirm full role integrity:

```bash
# Confirm trust policy is correctly set
aws iam get-role --role-name iamrole_kareem \
    --query "Role.AssumeRolePolicyDocument"

# Confirm no inline policies exist (should return empty list in a clean role)
aws iam list-role-policies --role-name iamrole_kareem
```

---

## Security and DevOps Principles Demonstrated

| Principle | Implementation |
|---|---|
| **Least-Privilege Access** | Customer-managed policy with scoped permissions; no AWS managed wildcard policies |
| **Explicit Trust Boundaries** | Trust policy restricted to `ec2.amazonaws.com` only; no wildcard principals |
| **Temporary Credential Model** | Role-based STS credentials eliminate long-lived static keys entirely |
| **Audit-Friendly Operations** | All provisioning steps executed via CLI with structured JSON output |
| **Dynamic Configuration** | Policy ARN resolved at runtime; no account IDs hard-coded |
| **Pre-flight Verification** | Identity confirmed before any mutations; deployment scoped to us-east-1 |
| **Validation-Driven Workflow** | Final state verified explicitly before close of change |

---

## Operational Considerations

### Instance Profile Requirement

An IAM role alone cannot be directly attached to an EC2 instance. For full EC2 deployment, create an instance profile as the next step:

```bash
# Create instance profile
aws iam create-instance-profile --instance-profile-name iamrole_kareem-profile

# Add role to instance profile
aws iam add-role-to-instance-profile \
    --instance-profile-name iamrole_kareem-profile \
    --role-name iamrole_kareem

# Associate with EC2 instance
aws ec2 associate-iam-instance-profile \
    --instance-id i-xxxxxxxxxxxxxxxxx \
    --iam-instance-profile Name=iamrole_kareem-profile
```

### Region Scoping

This workflow was executed in `us-east-1`. IAM is a global service, but instance profiles and role associations are regional in practice. Ensure the correct `--region` flag or `AWS_DEFAULT_REGION` environment variable is set in multi-region environments.

### Cleanup (When Decommissioning)

```bash
# Detach policy before deleting role
aws iam detach-role-policy \
    --role-name iamrole_kareem \
    --policy-arn $POLICY_ARN

# Delete the role
aws iam delete-role --role-name iamrole_kareem
```

---

## Troubleshooting Reference

| Symptom | Probable Cause | Resolution |
|---|---|---|
| `NoSuchEntity` on role creation | Trust policy file path incorrect | Confirm `trust-policy.json` exists with `ls -la` |
| `$POLICY_ARN` is empty | Policy name typo or policy does not exist | Verify with `aws iam list-policies --scope Local` |
| `AccessDenied` on any step | IAM user lacks required permissions | Confirm `iam:CreateRole`, `iam:AttachRolePolicy` are granted |
| EC2 cannot assume role | Instance profile not created or not attached | See Instance Profile Requirement section above |
| Role exists but policy is missing | Step 5 ran against wrong role name | Re-run `attach-role-policy` and validate with Step 6 |

---

## Outcome

- IAM role `iamrole_kareem` provisioned successfully in account `472012609282`
- Trust policy correctly restricts assumption to `ec2.amazonaws.com` only
- Customer-managed policy `iampolicy_kareem` attached and verified
- No static credentials created or stored at any point in the workflow
- Deployment fully auditable via AWS CloudTrail
- Workflow is portable and automation-ready for CI/CD integration
