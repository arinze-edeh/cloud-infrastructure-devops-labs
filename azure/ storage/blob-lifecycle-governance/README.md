# Azure Blob Lifecycle Management Automation

> **Enterprise DevOps Solution** | Automating data retention cost optimization via Azure Blob Storage Lifecycle Policies using Azure CLI

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Solution Overview](#solution-overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Environment Reference](#environment-reference)
- [Implementation Guide](#implementation-guide)
  - [Phase 1: Authentication and Subscription Verification](#phase-1-authentication-and-subscription-verification)
  - [Phase 2: Storage Account Provisioning](#phase-2-storage-account-provisioning)
  - [Phase 3: Blob Container Creation](#phase-3-blob-container-creation)
  - [Phase 4: File Upload to Container](#phase-4-file-upload-to-container)
  - [Phase 5: Lifecycle Policy Configuration](#phase-5-lifecycle-policy-configuration)
  - [Phase 6: Validation](#phase-6-validation)
- [Challenges and Resolutions](#challenges-and-resolutions)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Cost Impact](#cost-impact)
- [Contributing](#contributing)

---

## Problem Statement

The Nautilus DevOps team faced escalating Azure Blob Storage costs driven by the indefinite retention of stale blobs. Without an automated deletion policy, old data continued to accumulate in storage containers, generating unnecessary charges. Manual cleanup was error-prone, inconsistent, and not scalable.

**Goal:** Implement an automated, policy-driven approach to delete blobs older than **7 days from last modification** within a designated container, reducing storage overhead without requiring manual intervention.

---

## Solution Overview

This implementation provisions an Azure Storage Account, creates a Blob Container, uploads a test artifact, and applies a named Lifecycle Management Policy using the Azure CLI. The solution is fully scripted, repeatable, and validated end-to-end.

| Component | Value |
|---|---|
| Storage Account | `nautilusstor28249` |
| Resource Group | `kml_rg_main-cacd4dc2b7844376` |
| Region | East US |
| Redundancy | Locally Redundant Storage (LRS) |
| Container | `nautilus-container28249` |
| Lifecycle Rule | `nautilus-del-rule` |
| Retention Threshold | 7 days after last modification |
| Test File | `tempfile.txt` |

---

## Architecture

```
Azure Subscription (Azure Free Labs)
 └── Resource Group: kml_rg_main-cacd4dc2b7844376 [eastus]
      └── Storage Account: nautilusstor28249 [Standard_LRS | StorageV2]
           └── Blob Container: nautilus-container28249
                ├── Blob: tempfile.txt
                └── Management Policy: nautilus-del-rule
                     └── Rule: Delete blockBlobs after 7 days of last modification
```

---

## Prerequisites

Before executing any commands, confirm the following are in place:

- Azure CLI version **2.67.0** or later installed on the client host
- Active Azure account with at minimum **Storage Account Contributor** and **Storage Blob Data Contributor** roles on the target resource group
- Network access to `management.azure.com` and `*.blob.core.windows.net`
- The file `tempfile.txt` present at `/root/tempfile.txt` on the client host
- Bash shell (Linux/macOS) or WSL on Windows

**Verify Azure CLI installation:**

```bash
az --version
```

**Expected output (abbreviated):**
```
azure-cli   2.67.0
core        2.67.0
```

> **Screenshot**

<img width="1033" height="494" alt="Image" src="https://github.com/user-attachments/assets/0b005131-51db-452a-943a-d8bc2345f146" />

> *Caption: Azure CLI 2.67.0 confirmed on the client host with Python 3.10 runtime.*

---

## Environment Reference

| Parameter | Value |
|---|---|
| Portal URL | https://portal.azure.com |
| Subscription Name | Azure Free Labs |
| Subscription ID | `f0c3bcdd-5ce2-4fa0-8cf3-41559747512b` |
| Tenant ID | `54c1a2d3-d100-453c-9636-3a109eb45552` |
| Resource Group | `kml_rg_main-cacd4dc2b7844376` |
| Region | `eastus` |

---

## Implementation Guide

### Phase 1: Authentication and Subscription Verification

#### Step 1.1 - List and verify available subscriptions

```bash
az account list --output table
```

**Expected output:**
```
Name             CloudName    SubscriptionId                        TenantId                              State    IsDefault
---------------  -----------  ------------------------------------  ------------------------------------  -------  -----------
Azure Free Labs  AzureCloud   f0c3bcdd-5ce2-4fa0-8cf3-41559747512b  54c1a2d3-d100-453c-9636-3a109eb45552  Enabled  True
```

#### Step 1.2 - Confirm the active account context

```bash
az account show --output table
```

**Expected output:**
```
EnvironmentName    HomeTenantId                          IsDefault    Name             State    TenantId
-----------------  ------------------------------------  -----------  ---------------  -------  ------------------------------------
AzureCloud         54c1a2d3-d100-453c-9636-3a109eb45552  True         Azure Free Labs  Enabled  54c1a2d3-d100-453c-9636-3a109eb45552
```

#### Step 1.3 - Explicitly set the target subscription

```bash
az account set --subscription "f0c3bcdd-5ce2-4fa0-8cf3-41559747512b"
```

> **Screenshot**

<img width="1088" height="328" alt="Image" src="https://github.com/user-attachments/assets/937ed68f-6bfd-4ec7-b87c-f97b9f2f920c" />

> *Caption: Subscription confirmed as "Azure Free Labs" and set as the active context.*

---

### Phase 2: Storage Account Provisioning

#### Step 2.1 - Discover existing resource groups

```bash
az group list --output table
```

**Expected output:**
```
Name                          Location    Status
----------------------------  ----------  ---------
kml_rg_main-cacd4dc2b7844376  eastus      Succeeded
```

> **IMPORTANT:** The lab identity does not have `Microsoft.Resources/subscriptions/resourcegroups/write` permission. **Do not attempt to create a new resource group.** Use the pre-existing group `kml_rg_main-cacd4dc2b7844376` for all resource provisioning. See [Challenges and Resolutions](#challenges-and-resolutions) for full details.

#### Step 2.2 - Create the Storage Account

```bash
az storage account create \
  --name "nautilusstor28249" \
  --resource-group "kml_rg_main-cacd4dc2b7844376" \
  --location "eastus" \
  --sku "Standard_LRS" \
  --kind "StorageV2"
```

**Parameter Reference:**

| Flag | Value | Reason |
|---|---|---|
| `--sku` | `Standard_LRS` | Locally Redundant Storage as specified |
| `--kind` | `StorageV2` | Required to support Lifecycle Management policies |
| `--location` | `eastus` | Matches resource group region |

#### Step 2.3 - Verify successful provisioning

```bash
az storage account show \
  --name "nautilusstor28249" \
  --resource-group "kml_rg_main-cacd4dc2b7844376" \
  --query "{Name:name, Location:location, SKU:sku.name, State:provisioningState}" \
  --output table
```

**Expected output:**
```
Name               Location    SKU           State
-----------------  ----------  ------------  ---------
nautilusstor28249  eastus      Standard_LRS  Succeeded
```

> **Screenshot Placeholder**
> ![Storage Account Created](./screenshots/03-storage-account-created.png)
> *Caption: Storage account `nautilusstor28249` provisioned in East US with Standard_LRS SKU.*

---

### Phase 3: Blob Container Creation

#### Step 3.1 - Retrieve and export the Storage Account key

```bash
STORAGE_KEY=$(az storage account keys list \
  --account-name "nautilusstor28249" \
  --resource-group "kml_rg_main-cacd4dc2b7844376" \
  --query "[0].value" \
  --output tsv)

echo $STORAGE_KEY
```

> **Security Note:** The key is exported to a shell variable for session use only. It is never written to disk or committed to version control. Rotate this key after the lab session ends.

#### Step 3.2 - Create the Blob Container

```bash
az storage container create \
  --name "nautilus-container28249" \
  --account-name "nautilusstor28249" \
  --account-key "$STORAGE_KEY" \
  --public-access "off"
```

**Expected output:**
```json
{
  "created": true
}
```

#### Step 3.3 - Verify container creation

```bash
az storage container show \
  --name "nautilus-container28249" \
  --account-name "nautilusstor28249" \
  --account-key "$STORAGE_KEY" \
  --query "{Container:name, LastModified:properties.lastModified, LeaseState:properties.leaseState}" \
  --output table
```

**Expected output:**
```
Container                LastModified
-----------------------  -------------------------
nautilus-container28249  2026-03-16T20:34:52+00:00
```

> **Screenshot Placeholder**
> ![Container Created](./screenshots/04-container-created.png)
> *Caption: Container `nautilus-container28249` created with public access disabled.*

---

### Phase 4: File Upload to Container

#### Step 4.1 - Confirm the source file exists on the client host

```bash
ls -la /root/tempfile.txt
```

**Expected output:**
```
-rw-r--r-- 1 root root 25 Mar 16 20:21 /root/tempfile.txt
```

#### Step 4.2 - Upload the file to the Blob Container

```bash
az storage blob upload \
  --container-name "nautilus-container28249" \
  --account-name "nautilusstor28249" \
  --account-key "$STORAGE_KEY" \
  --file "/root/tempfile.txt" \
  --name "tempfile.txt" \
  --overwrite
```

**Expected output (abbreviated):**
```
Finished[#############################################################]  100.0000%
{
  "lastModified": "2026-03-16T20:36:20+00:00",
  "request_server_encrypted": true
}
```

#### Step 4.3 - Verify the blob is listed in the container

```bash
az storage blob list \
  --container-name "nautilus-container28249" \
  --account-name "nautilusstor28249" \
  --account-key "$STORAGE_KEY" \
  --query "[].{Name:name, Size:properties.contentLength, LastModified:properties.lastModified}" \
  --output table
```

**Expected output:**
```
Name          Size    LastModified
------------  ------  -------------------------
tempfile.txt  25      2026-03-16T20:36:20+00:00
```

> **Screenshot Placeholder**
> ![File Uploaded](./screenshots/05-blob-upload-verified.png)
> *Caption: `tempfile.txt` (25 bytes) uploaded and visible in `nautilus-container28249`.*

---

### Phase 5: Lifecycle Policy Configuration

#### Step 5.1 - Create the Lifecycle Management policy JSON file

```bash
cat > /tmp/lifecycle-policy.json << 'EOF'
{
  "rules": [
    {
      "name": "nautilus-del-rule",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["nautilus-container28249/"]
        },
        "actions": {
          "baseBlob": {
            "delete": {
              "daysAfterModificationGreaterThan": 7
            }
          }
        }
      }
    }
  ]
}
EOF
```

**Policy Design Breakdown:**

| Field | Value | Purpose |
|---|---|---|
| `name` | `nautilus-del-rule` | Unique rule identifier |
| `enabled` | `true` | Rule is active immediately upon application |
| `type` | `Lifecycle` | Azure policy type for storage lifecycle management |
| `blobTypes` | `["blockBlob"]` | Applies to standard block blobs only |
| `prefixMatch` | `["nautilus-container28249/"]` | Scopes the rule to this specific container |
| `daysAfterModificationGreaterThan` | `7` | Triggers deletion 7 days after last modification |

#### Step 5.2 - Verify the JSON file is well-formed

```bash
cat /tmp/lifecycle-policy.json
```

> **Do not proceed to Step 5.3 unless the JSON output matches exactly.** Malformed JSON will cause the policy to be rejected with an `InvalidPolicyDocument` error.

#### Step 5.3 - Apply the Lifecycle Management policy to the Storage Account

```bash
az storage account management-policy create \
  --account-name "nautilusstor28249" \
  --resource-group "kml_rg_main-cacd4dc2b7844376" \
  --policy @/tmp/lifecycle-policy.json
```

**Expected output (abbreviated):**
```json
{
  "name": "DefaultManagementPolicy",
  "policy": {
    "rules": [
      {
        "enabled": true,
        "name": "nautilus-del-rule",
        "type": "Lifecycle",
        "definition": {
          "actions": {
            "baseBlob": {
              "delete": {
                "daysAfterModificationGreaterThan": 7.0
              }
            }
          },
          "filters": {
            "blobTypes": ["blockBlob"],
            "prefixMatch": ["nautilus-container28249/"]
          }
        }
      }
    ]
  }
}
```

> **Screenshots**

<img width="1094" height="659" alt="Image" src="https://github.com/user-attachments/assets/2b98438b-edeb-4c58-8060-f4f94f83c188" />

<img width="1092" height="866" alt="Image" src="https://github.com/user-attachments/assets/c223fb54-ea18-4ba4-bc45-49dbcfda8695" />

<img width="1088" height="843" alt="Image" src="https://github.com/user-attachments/assets/1e5f2ae9-e482-4ca2-9651-377173cd4933" />

<img width="1098" height="613" alt="Image" src="https://github.com/user-attachments/assets/cc198d6e-c8c0-4434-ab60-9fa4963db521" />

> *Caption: Management policy `nautilus-del-rule` successfully applied to storage account.*

---

### Phase 6: Validation

#### Step 6.1 - Retrieve and filter the applied policy by rule name

```bash
az storage account management-policy show \
  --account-name "nautilusstor28249" \
  --resource-group "kml_rg_main-cacd4dc2b7844376" \
  --query "policy.rules[?name=='nautilus-del-rule']" \
  --output json
```

**Full validated output:**
```json
[
  {
    "definition": {
      "actions": {
        "baseBlob": {
          "delete": {
            "daysAfterCreationGreaterThan": null,
            "daysAfterLastAccessTimeGreaterThan": null,
            "daysAfterLastTierChangeGreaterThan": null,
            "daysAfterModificationGreaterThan": 7.0
          }
        }
      },
      "filters": {
        "blobTypes": [
          "blockBlob"
        ],
        "prefixMatch": [
          "nautilus-container28249/"
        ]
      }
    },
    "enabled": true,
    "name": "nautilus-del-rule",
    "type": "Lifecycle"
  }
]
```

**Validation Checklist:**

| Criteria | Expected Value | Status |
|---|---|---|
| Rule name | `nautilus-del-rule` | Confirmed |
| Rule enabled | `true` | Confirmed |
| Blob type filter | `blockBlob` | Confirmed |
| Prefix scope | `nautilus-container28249/` | Confirmed |
| Deletion threshold | `7.0` days after modification | Confirmed |
| Policy type | `Lifecycle` | Confirmed |

> **Screenshot**

<img width="1091" height="830" alt="Image" src="https://github.com/user-attachments/assets/a38a7076-a8db-40df-a3e8-60e01f9d5731" />

> *Caption: Rule `nautilus-del-rule` confirmed with all required attributes via `management-policy show`.*

---

## Challenges and Resolutions

### Challenge 1: Insufficient Permissions to Create a New Resource Group

**Symptom:**

```
(AuthorizationFailed) The client '2a82af3b-34ae-46ec-88a1-d533e59e2c78' with object id
'f665ca0e-6a2a-47a2-b4e7-eade07835ed4' does not have authorization to perform action
'Microsoft.Resources/subscriptions/resourcegroups/write'
```

**Root Cause:**

The lab identity `kk_lab_user_main-cacd4dc2b7844376` was scoped with a role that excludes `Microsoft.Resources/subscriptions/resourcegroups/write`. This is a deliberate constraint in sandboxed lab environments to prevent resource sprawl.

**Resolution:**

Enumerated existing resource groups using `az group list --output table`, identified the pre-provisioned group `kml_rg_main-cacd4dc2b7844376`, and used it as the target resource group for all subsequent provisioning. No new group creation was attempted.

**Prevention:**

Always run `az group list` before attempting to create a resource group in restricted lab or production environments. Confirm RBAC scope by running:

```bash
az role assignment list --assignee <object-id> --output table
```

---

## Best Practices

### Identity and Access Management

* **Principle of Least Privilege:** Assign only the roles required. For this task, `Storage Account Contributor` and `Storage Blob Data Contributor` are sufficient. Do not use Owner or Contributor at the subscription level.
* **Never hardcode credentials** in scripts or commit them to version control. Use environment variables or Azure Key Vault references in production pipelines.
* **Rotate storage account keys** regularly. Prefer Azure AD-based authentication (`--auth-mode login`) over shared key access in production workloads.

### Storage Account Design

* **Always use `StorageV2`** as the `--kind` parameter. The older `BlobStorage` kind does not support all Lifecycle Management features including tiering actions.
* **Disable public blob access** (`--public-access off`) on all containers unless there is an explicit business need for public access.
* **Lock the Storage Account** with a resource lock in production to prevent accidental deletion:
  ```bash
  az lock create --name "storage-lock" \
    --resource-group "kml_rg_main-cacd4dc2b7844376" \
    --resource "nautilusstor28249" \
    --resource-type "Microsoft.Storage/storageAccounts" \
    --lock-type CanNotDelete
  ```

### Lifecycle Policy Design

* **Scope policies using `prefixMatch`** to target specific containers or blob path prefixes rather than applying broad account-level rules that could inadvertently delete important data.
* **Test with a staging container** before applying lifecycle policies to production containers.
* **Combine tiering and deletion rules** for cost optimization. Tier blobs to Cool after 30 days and Archive after 90 days before deleting at 180 days, rather than deleting immediately.
* **Enable access time tracking** if deletion should be based on last access rather than last modification:
  ```bash
  az storage account blob-service-properties update \
    --account-name "nautilusstor28249" \
    --resource-group "kml_rg_main-cacd4dc2b7844376" \
    --enable-last-access-tracking true
  ```

### Operational Hygiene

* **Always verify with `--output table`** after each provisioning command before proceeding.
* **Export shell variables** for reuse within the session (like `$STORAGE_KEY`) to avoid repeated API calls and reduce authentication errors.
* **Store lifecycle policy JSON in source control** (e.g., `infra/lifecycle-policies/nautilus-del-rule.json`) rather than generating it inline, enabling peer review and audit trails.
* **Tag all resources** for cost tracking and governance:
  ```bash
  az storage account update \
    --name "nautilusstor28249" \
    --resource-group "kml_rg_main-cacd4dc2b7844376" \
    --tags environment=lab team=nautilus-devops owner=nautilus
  ```

---

## Lessons Learned

### 1. Probe RBAC Permissions Before Designing the Runbook

This engagement revealed that the lab identity lacked resource group creation rights. In enterprise environments, RBAC scope varies significantly across teams. Always enumerate role assignments and test boundary operations (like resource group creation) during the discovery phase, not during execution.

### 2. `StorageV2` is Non-Negotiable for Lifecycle Management

Using `--kind BlobStorage` would have succeeded for basic blob storage but silently broken Lifecycle Management support. Specifying `StorageV2` explicitly is the only way to guarantee full feature availability including management policies, tiering, and versioning.

### 3. Shell Variables Do Not Persist Across Sessions

The `$STORAGE_KEY` variable is session-scoped. If the terminal is closed and reopened, the variable is lost and must be re-exported. In CI/CD pipelines, always retrieve the key inline or source it from a secrets manager rather than relying on shell state.

### 4. JSON Policy Files Must Be Verified Before Application

The `--policy @/tmp/lifecycle-policy.json` flag will fail or throw a cryptic error if the JSON is malformed. Always run `cat /tmp/lifecycle-policy.json` to visually inspect the file before applying the policy.

### 5. Lifecycle Policy Execution is Asynchronous

After applying the policy, Azure evaluates and executes it on a background schedule (typically once every 24 hours). The policy being applied and verified does not mean blobs are immediately deleted. Plan validation windows accordingly.

### 6. `prefixMatch` Uses the Container Name as a Prefix Root

The value `"nautilus-container28249/"` in `prefixMatch` tells Azure to match blobs whose full path begins with that string. The trailing `/` is required to scope the rule to the container. Omitting it could match unintended paths.

### 7. Use Query Filters During Validation

The `--query "policy.rules[?name=='nautilus-del-rule']"` pattern is far more reliable for validation than reviewing raw JSON output manually. It eliminates ambiguity and provides a clean, auditable confirmation output.

---

## Cost Impact

| Scenario | Monthly Cost Impact |
|---|---|
| Without lifecycle policy (indefinite retention) | Accumulates linearly with blob growth |
| With 7-day deletion policy | Capped to rolling 7-day window of storage consumption |
| Estimated reduction (typical log workload) | 60 to 90% reduction in blob storage spend |

> Actual savings vary based on blob ingestion rate, average blob size, and access patterns. Use the Azure Pricing Calculator with the `Standard_LRS` tier in `East US` for precise estimates.

---

## Contributing

1. Fork this repository
2. Create a feature branch: `git checkout -b feature/lifecycle-enhancement`
3. Commit your changes: `git commit -m "feat: add archive tier rule before deletion"`
4. Push to the branch: `git push origin feature/lifecycle-enhancement`
5. Open a Pull Request with a description of the change and its business justification

---

## License

This project is licensed under the MIT License. See `LICENSE` for details.

---

*Maintained by the Nautilus DevOps Team | Last updated: March 2026*


<img width="1094" height="358" alt="Image" src="https://github.com/user-attachments/assets/76ad36cf-75fe-41bb-80df-12c1a83130e6" />

<img width="1093" height="841" alt="Image" src="https://github.com/user-attachments/assets/a8938fa5-77ad-4112-a701-3a6164c8e2a1" />

<img width="1092" height="865" alt="Image" src="https://github.com/user-attachments/assets/58671e57-9eca-4d00-9c8c-4cc2a8434e06" />

<img width="1099" height="630" alt="Image" src="https://github.com/user-attachments/assets/d56c39d3-6ce7-4c59-a630-e6b3e88d494c" />

<img width="1090" height="614" alt="Image" src="https://github.com/user-attachments/assets/8f2cbb85-5269-4a1e-b559-9f070f243932" />

<img width="1095" height="750" alt="Image" src="https://github.com/user-attachments/assets/8c2e40c4-fa12-4a1c-bd31-32ce49f836db" />

<img width="1091" height="715" alt="Image" src="https://github.com/user-attachments/assets/89ff56e9-f413-42f6-809a-d26e2943f312" />

<img width="1091" height="715" alt="Image" src="https://github.com/user-attachments/assets/0c263d32-b3de-46d2-871c-4c9125534936" />




