# Azure Blob Storage Access Hardening: Public-to-Private Container Remediation

**Domain:** Azure Storage Security | **Tooling:** Azure CLI | **Auth Model:** Storage Account Key Fallback

---

## Table of Contents

- [Project Overview](#project-overview)
- [Problem Statement](#problem-statement)
- [Architecture Context](#architecture-context)
- [Security Objective](#security-objective)
- [Tooling and Environment](#tooling-and-environment)
- [Execution Workflow](#execution-workflow)
  - [Step 1: Validate Active Azure Account](#step-1-validate-active-azure-account)
  - [Step 2: Define Environment Variables](#step-2-define-environment-variables)
  - [Step 3: Check Current Container Access Level](#step-3-check-current-container-access-level)
  - [Step 4: Attempt RBAC Authentication (Observed Limitation)](#step-4-attempt-rbac-authentication-observed-limitation)
  - [Step 5: Convert Public Container to Private](#step-5-convert-public-container-to-private)
  - [Step 6: Verify Access Level After Change](#step-6-verify-access-level-after-change)
  - [Step 7: List All Containers for Final Validation](#step-7-list-all-containers-for-final-validation)
- [Final Outcome](#final-outcome)
- [Errors and Resolutions](#errors-and-resolutions)
- [Key Decisions](#key-decisions)
- [Best Practices and Operational Considerations](#best-practices-and-operational-considerations)
- [Key Learnings](#key-learnings)
- [Why This Matters](#why-this-matters)

---

## Project Overview

This document captures a **security hardening operation** performed against an Azure Blob Storage container that was inadvertently configured with public access. The remediation converts the container from `container`-level public access to fully **private**, ensuring that unauthenticated users can no longer enumerate or read blob objects.

The operation was executed entirely via the **Azure CLI** using the storage account key fallback authentication mechanism, following a deliberate and verification-driven execution pattern aligned with production change management standards.

---

## Problem Statement

Public blob container access is one of the most common and consequential misconfigurations in cloud storage environments. When a container is set to `blob` or `container` access, any party with knowledge of the storage endpoint can retrieve objects without authentication. This exposes sensitive data to unauthorized access and places the organization at risk of data breaches, regulatory violations, and reputational damage.

In this scenario, the container `nautilus-container-11818` within the storage account `nautilusst17006` was found to have public read access enabled. The remediation goal was to disable that access while preserving all container contents and leaving the adjacent private container, `nautilus-priv-17871`, untouched.

---

## Architecture Context

| Layer | Detail |
|---|---|
| **Subscription** | Azure Free Labs |
| **Storage Account** | `nautilusst17006` |
| **Region** | `eastus` |
| **Target Container** | `nautilus-container-11818` (PUBLIC to PRIVATE) |
| **Unaffected Container** | `nautilus-priv-17871` (PRIVATE, unchanged) |
| **Authentication** | Service Principal + Storage Account Key Fallback |

---

## Security Objective

- Remove anonymous public access from a misconfigured blob container
- Prevent unauthenticated read access to blob objects
- Preserve existing private container configuration without modification
- Ensure zero data loss and no container recreation
- Confirm the remediated state through verification commands before closing the change

---

## Tooling and Environment

| Tool | Purpose |
|---|---|
| **Azure CLI** | Primary execution interface |
| **Azure Cloud Shell / Terminal** | Runtime environment |
| **Service Principal** | Identity used for subscription-level operations |
| **Storage Account Key** | Fallback credential for storage-plane commands |

> **Note:** Azure CLI storage commands do not always support `--auth-mode login`. When RBAC-based authentication is unavailable for a given subcommand, the CLI silently falls back to querying the storage account key. This behavior is documented and expected in this context.

---

## Execution Workflow

### Step 1: Validate Active Azure Account

Before performing any change, confirm the active subscription context and authenticated identity. This prevents accidental operations against an unintended subscription.

```bash
az account show
```

**Expected Output:**

- `"state": "Enabled"` confirms the subscription is active
- `"type": "servicePrincipal"` confirms the authentication method
- `"name": "Azure Free Labs"` confirms the correct subscription is selected

**Screenshot: Account context verification**

> Subscription state confirmed as `Enabled`. Service principal identity validated before proceeding with any storage operations.

<img width="1030" height="580" alt="az account show output confirming active subscription and service principal authentication" src="https://github.com/user-attachments/assets/867e0c69-c3cf-4264-8787-5e1ef4f96237" />

---

### Step 2: Define Environment Variables

Parameterize all target resource identifiers as shell environment variables. This eliminates hardcoded values from subsequent commands, reduces transcription errors, and makes the runbook portable across similar remediations.

```bash
export STORAGE_ACCOUNT="nautilusst17006"
export TARGET_CONTAINER="nautilus-container-11818"
export REGION="eastus"
```

**Operational Benefits:**

- Prevents typos in resource names across multiple commands
- Enables safe reuse of this runbook against other storage accounts
- Improves auditability when logging terminal sessions

**Screenshot: Environment variable declarations**

> Variables declared for storage account, target container, and region. All subsequent commands reference these variables exclusively.

<img width="1031" height="751" alt="Shell environment variables exported for storage account, container name, and region" src="https://github.com/user-attachments/assets/1122a816-f0af-4940-a243-b86c5fbacdff" />

---

### Step 3: Check Current Container Access Level

Before modifying any resource, establish a verified baseline of the current configuration. This confirms the problem exists and provides a reference state for change validation.

```bash
az storage container show \
  --name $TARGET_CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --query "properties.publicAccess" \
  --output tsv
```

**Observed Behavior:**

- The CLI warns that no explicit credentials were provided and will fall back to querying the storage account key automatically
- The command returns `container`, confirming the container currently allows full public read access, including blob enumeration

> **Risk Implication:** A return value of `container` means any unauthenticated HTTP client can list all blobs in this container and download their contents without a SAS token or account key.

**Screenshot: Pre-change public access confirmation**

> Output returns `container`, confirming unauthenticated public access is currently active on the target container.

<img width="1034" height="850" alt="az storage container show output returning 'container' public access level before remediation" src="https://github.com/user-attachments/assets/ca111580-4f2d-48db-abf4-b3081a07056c" />

---

### Step 4: Attempt RBAC Authentication (Observed Limitation)

A least-privilege approach was attempted first using `--auth-mode login`, which delegates authorization to Azure Active Directory and the service principal's RBAC role assignments rather than relying on the storage account master key.

```bash
az storage container set-permission \
  --name $TARGET_CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --public-access off \
  --auth-mode login
```

**Result:**

The command fails with the following error:

```
az storage container set-permission: 'login' is not a valid value for '--auth-mode'. Allowed values: key.
```

**Root Cause:**

The `az storage container set-permission` subcommand does not support `--auth-mode login`. Unlike most storage data-plane operations that accept both `key` and `login` modes, this specific command is restricted to key-based authentication only. This is a known CLI limitation, not an RBAC misconfiguration.

**Resolution:** Proceed without `--auth-mode login`. The CLI will automatically retrieve the storage account key and use it for authorization.

**Screenshot: RBAC auth mode failure**

> The `--auth-mode login` flag is rejected by this specific subcommand. Allowed authentication mode is `key` only. The command exits with a non-zero status.

<img width="1032" height="672" alt="az storage container set-permission failing with --auth-mode login not supported error" src="https://github.com/user-attachments/assets/e18a314d-1593-4b43-beb6-f45da6924d16" />

---

### Step 5: Convert Public Container to Private

Execute the remediation by removing public access from the target container. By omitting `--auth-mode`, the CLI defaults to key-based authentication and automatically retrieves the storage account key from the Azure control plane.

```bash
az storage container set-permission \
  --name $TARGET_CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --public-access off
```

**Outcome:**

- Azure CLI queries and uses the storage account key transparently
- The container's public access policy is updated from `container` to `off`
- The command returns a JSON response confirming the operation metadata including `client_request_id`, `date`, `etag`, `last_modified`, `request_id`, and `version`
- No container recreation occurs. Existing blob objects are untouched.

> **Key Signal:** A successful response includes an HTTP `etag` and `last_modified` timestamp. A non-zero exit code or an error message would indicate failure. The green prompt icon following the command confirms successful execution.

**Screenshot: Successful public access removal**

> Command completes with a JSON response containing `etag` and `last_modified` fields confirming the access policy change was applied.

<img width="1032" height="510" alt="az storage container set-permission success response with etag and modification timestamp" src="https://github.com/user-attachments/assets/1e648b06-5cb4-4352-a41a-f821935864ee" />

---

### Step 6: Verify Access Level After Change

Immediately following the remediation, re-query the container's public access property to confirm the change was applied. This verification step is mandatory for any security-sensitive operation.

```bash
az storage container show \
  --name $TARGET_CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --query "properties.publicAccess" \
  --output tsv
```

**Expected Output:**

```
(empty / null)
```

A `null` or empty response from this query is the authoritative confirmation that public access has been removed. The absence of a value means no access tier (`blob` or `container`) is assigned, and the container is fully private.

> **Why Null Means Private:** Azure's storage model treats `null` as the absence of any public access grant. Any value other than `null` (for example, `blob` or `container`) would indicate remaining public exposure.

**Screenshot: Post-change access level verification**

> Query returns empty (`null`), confirming public access has been fully revoked. The container now requires authenticated access for all operations.

<img width="1033" height="539" alt="az storage container show returning null public access after remediation confirming private state" src="https://github.com/user-attachments/assets/0874306a-f075-47e8-89a7-4f54a2d5567a" />

---

### Step 7: List All Containers for Final Validation

Perform a full inventory check across all containers in the storage account to confirm the broader access state. This final step validates that the target container is private and that no unintended side effects were introduced on adjacent containers.

```bash
az storage container list \
  --account-name $STORAGE_ACCOUNT \
  --query "[].{Name:name, PublicAccess:properties.publicAccess}" \
  --output table
```

**Validation Results:**

| Container Name | Public Access |
|---|---|
| `nautilus-container-11818` | `None` (Private) |
| `nautilus-priv-17871` | `None` (Private, unchanged) |

Both containers now report private access. The pre-existing private container `nautilus-priv-17871` was not affected by the operation.

> **Operational Note:** The `PublicAccess` column returning blank/null for both containers is the expected and desired state. This confirms the remediation scope was contained exactly as intended.

**Screenshot: Full container inventory validation**

> Container list confirms both containers are private. No unintended changes were introduced to adjacent resources.

<img width="1039" height="425" alt="az storage container list table output showing both containers with no public access configured" src="https://github.com/user-attachments/assets/7a391df4-3019-467c-953e-e311257ff107" />

---

## Final Outcome

| Objective | Status |
|---|---|
| Public access removed from target container | Confirmed |
| No container recreation | Confirmed |
| No data loss | Confirmed |
| Adjacent private container preserved | Confirmed |
| Post-change verification completed | Confirmed |

The storage account `nautilusst17006` now has all containers configured with private access. No unauthenticated access path remains.

---

## Errors and Resolutions

**Error: `--auth-mode login` not supported by `set-permission`**

- **Step:** Step 4
- **Command:** `az storage container set-permission ... --auth-mode login`
- **Error Message:** `az storage container set-permission: 'login' is not a valid value for '--auth-mode'. Allowed values: key.`
- **Root Cause:** The `set-permission` subcommand is a legacy storage data-plane operation that was never updated to support Azure AD token-based authentication. It exclusively supports account key authentication.
- **Resolution:** Re-ran the command without `--auth-mode`. The CLI automatically retrieved and used the storage account key, completing the operation successfully.
- **Lesson:** Not all `az storage` subcommands have parity with `--auth-mode login`. Always verify CLI documentation or test in a non-production environment when RBAC enforcement is a hard requirement.

---

## Key Decisions

**Why use storage account key instead of RBAC?**
The `az storage container set-permission` subcommand does not accept `--auth-mode login`, making key-based authentication the only available path for this specific operation. In environments where storage account key usage is restricted by policy (for example, via `AllowStorageAccountKeyAccess: false`), an alternative approach using the Azure Portal, ARM API, or Terraform would be required.

**Why use environment variables instead of inline values?**
Environment variables reduce the risk of typos in repeated commands, improve terminal session auditability, and make the runbook reusable across similar remediations with minimal modification.

**Why validate before and after the change?**
Security-sensitive operations require a verified baseline and a verified final state. A pre-change check confirms the problem exists. A post-change check confirms the remediation succeeded. Without both, the change cannot be closed with confidence.

---

## Best Practices and Operational Considerations

- **Always establish a pre-change baseline.** Query the current state before any modification. This provides evidence that the change was necessary and gives a reference point for rollback assessment if needed.
- **Verify the change immediately after applying it.** Do not assume success based on a zero exit code alone. Re-query the specific property that was modified.
- **Perform a broader inventory check as the final step.** A single-resource check confirms the target was changed. A multi-resource check confirms no blast radius on adjacent resources.
- **Prefer RBAC over account keys in production environments.** Storage account keys are highly privileged. Where possible, assign `Storage Blob Data Contributor` or scoped RBAC roles to service principals and use `--auth-mode login`. Reserve key-based access for operations where RBAC support is absent.
- **Disable public blob access at the storage account level.** In addition to per-container settings, Azure allows you to disable public access globally at the storage account level using `az storage account update --allow-blob-public-access false`. This prevents any container in the account from being set to public, regardless of individual container configuration.
- **Use environment variables for all parameterized runbooks.** Hardcoded resource names in CLI scripts are an operational anti-pattern. Variables improve maintainability, portability, and session auditability.
- **Tag and document storage remediation operations.** In production, changes of this nature should be tracked in a change management system with pre-change and post-change evidence attached.

---

## Key Learnings

- **Azure CLI automatically falls back to storage account key authentication** when no explicit credential is provided and `--auth-mode` is not specified. This behavior is documented but can mask underlying RBAC gaps if not understood.
- **Not all `az storage` subcommands support `--auth-mode login`.** Specifically, `set-permission` only accepts `key` as a valid auth mode. Engineers enforcing RBAC-only policies must account for this gap and use alternative remediation paths where key access is disabled.
- **A `null` public access value is the authoritative indicator of a private container.** Any non-null value (`blob` or `container`) indicates remaining public exposure.
- **Validation steps are not optional for security operations.** A change that is not verified is a change that cannot be trusted.
- **Misconfigured public storage is one of the highest-frequency cloud security findings.** Storage account public access should be reviewed as part of every cloud security baseline assessment.

---

## Why This Matters

Publicly accessible blob containers are consistently ranked among the top sources of cloud data exposure incidents. The consequences range from intellectual property theft to GDPR violations to full credential compromise when sensitive files such as configuration files, key backups, or service account credentials are stored in public containers.

This remediation demonstrates:

- **Practical cloud security governance:** Identifying a misconfiguration, remediating it with the minimum required access, and verifying the outcome
- **Defensive security thinking:** Attempting least-privilege authentication first before falling back to account key usage
- **Real-world CLI troubleshooting:** Recognizing and resolving a known CLI limitation without escalating or abandoning the task
- **Verification-driven execution:** Treating pre-change and post-change validation as non-negotiable steps rather than optional hygiene




























# Azure Blob Container Access Hardening (Public → Private)

## Project Overview

This project documents a **security hardening operation** performed on an Azure Blob Storage container.
A publicly accessible blob container was converted to **private access** to restrict data exposure and
align the storage account with internal-only access requirements.

The operation was executed using the **Azure CLI**, leveraging the **storage account key fallback**
authentication mechanism when explicit credentials were not provided.

---

## Architecture Context

**Azure Subscription** 
- `Azure Free Labs`


**Storage Layer**

- Storage Account: `nautilusst17006`
- Region: `eastus`


**Blob Containers**

- `nautilus-container-11818` (PUBLIC → PRIVATE)

- `nautilus-priv-17871` (PRIVATE, unchanged)


---

## Security Objective

- Remove public access from a blob container
- Prevent anonymous read access
- Preserve existing private container configuration
- Ensure no data loss or container recreation

---

## Tooling & Environment

- Azure CLI
- Azure Cloud Environment
- Service Principal Authentication
- Storage Account Key (auto-queried by CLI)

---

## Step 1: Validate Active Azure Account

```
az account show
```

#### Expected Outcome

- Subscription is Enabled

- Correct tenant and subscription selected

- Service principal authenticated

📸 Screenshot:
<img width="1030" height="580" alt="image" src="https://github.com/user-attachments/assets/867e0c69-c3cf-4264-8787-5e1ef4f96237" />

## Step 2: Define Environment Variables
```
export STORAGE_ACCOUNT="nautilusst17006"
export TARGET_CONTAINER="nautilus-container-11818"
export REGION="eastus"
```

#### Purpose

- Avoid hard-coding values

- Improve command readability

- Enable reusability

📸 Screenshot:
<img width="1031" height="751" alt="image" src="https://github.com/user-attachments/assets/1122a816-f0af-4940-a243-b86c5fbacdff" />

## Step 3: Check Current Container Access Level
```
az storage container show \
  --name $TARGET_CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --query "properties.publicAccess" \
  --output tsv
```

#### Observed Behavior

- CLI prompts for credentials

- Azure automatically queries storage account key

- Output returns container, confirming public access

📸 Screenshot:
<img width="1034" height="850" alt="image" src="https://github.com/user-attachments/assets/ca111580-4f2d-48db-abf4-b3081a07056c" />

## Step 4: Attempt RBAC Authentication (Observed Limitation)
```
az storage container set-permission \
  --name $TARGET_CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --public-access off \
  --auth-mode login
```

#### Result

- Command fails

- `--auth-mode` login not supported in this context

- Allowed value defaults to key-based authentication

📸 Screenshot:
<img width="1032" height="672" alt="image" src="https://github.com/user-attachments/assets/e18a314d-1593-4b43-beb6-f45da6924d16" />

## Step 5: Convert Public Container to Private (Successful)
```
az storage container set-permission \
  --name $TARGET_CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --public-access off
```

#### Outcome

- Azure CLI automatically retrieves storage account key

- Public access successfully disabled

- No impact on container contents

📸 Screenshot:
<img width="1032" height="510" alt="image" src="https://github.com/user-attachments/assets/1e648b06-5cb4-4352-a41a-f821935864ee" />

## Step 6: Verify Access Level After Change
```
az storage container show \
  --name $TARGET_CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --query "properties.publicAccess" \
  --output tsv
```
#### Expected Output

`null`

This confirms:

- No anonymous access

- Container is fully private

📸 Screenshot:
<img width="1033" height="539" alt="image" src="https://github.com/user-attachments/assets/0874306a-f075-47e8-89a7-4f54a2d5567a" />

## Step 7: List All Containers for Final Validation
```
az storage container list \
  --account-name $STORAGE_ACCOUNT \
  --query "[].{Name:name, PublicAccess:properties.publicAccess}" \
  --output table
```

#### Validation Results

- `nautilus-container-11818 → Private`

- `nautilus-priv-17871 → Private (unchanged)`

📸 Screenshot:
<img width="1039" height="425" alt="image" src="https://github.com/user-attachments/assets/7a391df4-3019-467c-953e-e311257ff107" />

## Final Outcome

- Public access successfully removed

- No container recreation

- No data loss

- Existing private container preserved

- Security posture improved

## Key Learnings

- Azure CLI will fallback to account keys if credentials are not explicitly supplied

- Not all storage operations support `--auth-mode login`

- Public blob access should be disabled unless explicitly required

- Validation steps are critical in security-sensitive operations


## Why This Matters

- Misconfigured public storage is one of the top cloud security risks.

This project demonstrates:

- Practical cloud governance

- Defensive security thinking

- Real-world CLI troubleshooting

- Verification-driven execution
