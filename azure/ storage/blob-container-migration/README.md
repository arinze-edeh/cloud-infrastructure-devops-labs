# Azure Blob Container Migration Pipeline

> **Enterprise-grade blob data migration using Azure CLI with full integrity verification.**
> Storage Account: `nautilusst2986` | Source: `nautilus-source-4584` | Destination: `nautilus-dest-30683`

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Resolution Summary](#resolution-summary)
- [Prerequisites](#prerequisites)
- [Environment Setup](#environment-setup)
- [Phase 1 - Environment Verification](#phase-1---environment-verification)
- [Phase 2 - Source Container Validation](#phase-2---source-container-validation)
- [Phase 3 - Destination Container Provisioning](#phase-3---destination-container-provisioning)
- [Phase 4 - Data Migration](#phase-4---data-migration)
- [Phase 5 - Data Consistency Verification](#phase-5---data-consistency-verification)
- [Phase 6 - Final Audit and Cleanup](#phase-6---final-audit-and-cleanup)
- [Verification Results](#verification-results)
- [Known Issues and Resolutions](#known-issues-and-resolutions)
- [Security Considerations](#security-considerations)
- [Repository Structure](#repository-structure)

---

## Overview

This runbook documents the end-to-end process for migrating blob data between Azure Storage containers within the same storage account using the Azure CLI. It was executed as part of the **Nautilus DevOps Team** data migration project and covers container provisioning, blob copy, and multi-layered integrity verification.

Every step in this guide was validated in a live Azure Free Labs environment. No assumptions are made. All commands are production-ready and reproducible.

---

## Problem Statement

The team was required to:

1. Create a new **private** Azure Blob container named `nautilus-dest-30683` under the existing storage account `nautilusst2986`
2. Migrate the file `nautilus.txt` from the existing source container `nautilus-source-4584` to the newly created destination container
3. Confirm **data consistency** between both containers with no data loss or corruption
4. Perform all operations exclusively using the **Azure CLI**

---

## Resolution Summary

| Requirement | Status | Detail |
|---|---|---|
| New private container created | **PASSED** | `nautilus-dest-30683` provisioned with `public-access: off` |
| File migrated | **PASSED** | `nautilus.txt` copied via `az storage blob copy start` |
| Copy status confirmed | **PASSED** | `copy_status: success`, progress `33/33` bytes |
| MD5 checksum match | **PASSED** | `Lu7zilatbGguzSz2Ecn5IQ==` identical on both blobs |
| File size match | **PASSED** | `33` bytes in source and destination |
| Binary diff clean | **PASSED** | `diff` returned no output |
| Both containers audited | **PASSED** | `nautilus.txt` visible in both via `az storage blob list` |

---

## Prerequisites

Before executing this runbook, confirm the following are in place:

- Azure CLI version `2.x` or later installed
- An authenticated Azure session (service principal or interactive login)
- Access to the storage account `nautilusst2986` with key-level permissions
- A Unix-compatible shell (bash, zsh) for environment variable support
- `diff` utility available on the local machine (standard on all Unix/Linux systems)

---

## Environment Setup

### Critical: Use Shell Variables for Account Key

**Never paste the storage account key directly into commands.** Azure storage keys contain `+` and `=` characters that the shell will misinterpret as argument separators if not properly handled. Injecting them via copy-paste caused authentication failures during initial execution (see [Known Issues and Resolutions](#known-issues-and-resolutions)).

The safe approach is to load the key directly into a shell variable at the start of your session and reference it as `"$ACCOUNT_KEY"` throughout:

```bash
ACCOUNT_KEY=$(az storage account keys list \
  --account-name nautilusst2986 \
  --query "[0].value" \
  --output tsv)
```

Verify it was captured correctly:

```bash
echo $ACCOUNT_KEY
```

> This variable persists only for the duration of your terminal session. If you open a new terminal or session, you must re-run the export command above before proceeding.

---

## Phase 1 - Environment Verification

### Step 1 - Verify Azure CLI Installation

```bash
az --version
```

**Expected output:** Azure CLI version `2.x` or higher with core components listed.

***Screenshot Placeholder: Step 1 - az --version output showing CLI version 2.67.0***

---

### Step 2 - Confirm Active Authentication

```bash
az account show
```

**Expected output:** A JSON object confirming the active subscription, tenant, and authenticated user or service principal.

***Screenshot Placeholder: Step 2 - az account show confirming Azure Free Labs subscription as active and enabled***

> If `"state"` is not `"Enabled"` or `"isDefault"` is `false`, run `az account set --subscription "<your-subscription-id>"` before continuing.

---

### Step 3 - Set Target Subscription (Conditional)

Only required if the account has multiple subscriptions. Skip if `"isDefault": true` was returned in Step 2.

```bash
az account set --subscription "f0c3bcdd-5ce2-4fa0-8cf3-41559747512b"
```

---

## Phase 2 - Source Container Validation

### Step 4 - Verify Storage Account Exists

```bash
az storage account show \
  --name nautilusst2986 \
  --query "{name:name, resourceGroup:resourceGroup, location:location}" \
  --output table
```

**Expected output:**

```
Name            ResourceGroup                 Location
--------------  ----------------------------  ----------
nautilusst2986  kml_rg_main-153b09c2ec2c4df8  eastus
```

***Screenshot Placeholder: Step 4 - Storage account show confirming nautilusst2986 in resource group kml_rg_main-153b09c2ec2c4df8 located in eastus***

---

### Step 5 - Load Storage Account Key into Shell Variable

```bash
ACCOUNT_KEY=$(az storage account keys list \
  --account-name nautilusst2986 \
  --query "[0].value" \
  --output tsv)
```

Verify:

```bash
echo $ACCOUNT_KEY
```

***Screenshot Placeholder: Step 5 - echo $ACCOUNT_KEY confirming full base64 key was captured without truncation***

> Do not log, commit, or share the value of `$ACCOUNT_KEY`. Treat it as a secret credential.

---

### Step 6 - Confirm Source Container Exists

```bash
az storage container show \
  --name nautilus-source-4584 \
  --account-name nautilusst2986 \
  --account-key "$ACCOUNT_KEY"
```

**Expected output:** A JSON object with `"name": "nautilus-source-4584"` and `"publicAccess": null`.

***Screenshot Placeholder: Step 6 - az storage container show returning JSON confirming nautilus-source-4584 exists with lease state available and status unlocked***

---

### Step 7 - Confirm Source Blob Exists

```bash
az storage blob show \
  --container-name nautilus-source-4584 \
  --name nautilus.txt \
  --account-name nautilusst2986 \
  --account-key "$ACCOUNT_KEY" \
  --query "{name:name, size:properties.contentLength, lastModified:properties.lastModified}" \
  --output table
```

**Expected output:**

```
Name          Size    LastModified
------------  ------  -------------------------
nautilus.txt  33      2026-03-12T00:09:26+00:00
```

***Screenshot Placeholder: Step 7 - blob show table confirming nautilus.txt exists in source container with size 33 bytes***

> Record the `Size` value. You will validate this same number appears in the destination after migration.

---

## Phase 3 - Destination Container Provisioning

### Step 8 - Create the Destination Container

```bash
az storage container create \
  --name nautilus-dest-30683 \
  --account-name nautilusst2986 \
  --account-key "$ACCOUNT_KEY" \
  --public-access off
```

**Expected output:**

```json
{
  "created": true
}
```

***Screenshot Placeholder: Step 8 - az storage container create returning created: true confirming successful provisioning of nautilus-dest-30683***

> Any output other than `"created": true` indicates a failure. Do not proceed until this is confirmed.

---

### Step 9 - Verify Container Privacy Setting

```bash
az storage container show \
  --name nautilus-dest-30683 \
  --account-name nautilusst2986 \
  --account-key "$ACCOUNT_KEY" \
  --query "{name:name, publicAccess:properties.publicAccess}" \
  --output table
```

**Expected output:** Only the `Name` column is populated. The absence of a `publicAccess` value confirms the container has no public access configured.

***Screenshot Placeholder: Step 9 - container show table displaying only Name column for nautilus-dest-30683 confirming private access***

---

## Phase 4 - Data Migration

### Step 10 - Copy Blob from Source to Destination

```bash
az storage blob copy start \
  --source-account-name nautilusst2986 \
  --source-account-key "$ACCOUNT_KEY" \
  --source-container nautilus-source-4584 \
  --source-blob nautilus.txt \
  --destination-container nautilus-dest-30683 \
  --destination-blob nautilus.txt \
  --account-name nautilusst2986 \
  --account-key "$ACCOUNT_KEY"
```

**Expected output:**

```json
{
  "copy_id": "162f6127-1b2b-4dc3-bb7e-5311994b562a",
  "copy_status": "success",
  "date": "2026-03-12T00:24:40+00:00",
  "last_modified": "2026-03-12T00:24:41+00:00"
}
```

***Screenshot Placeholder: Step 10 - blob copy start returning JSON with copy_status success and copy_id 162f6127-1b2b-4dc3-bb7e-5311994b562a***

> For larger files, `copy_status` may initially return `"pending"`. In that case, proceed to Step 11 and poll until `"success"` is confirmed before continuing.

---

### Step 11 - Poll Copy Status on Destination Blob

```bash
az storage blob show \
  --container-name nautilus-dest-30683 \
  --name nautilus.txt \
  --account-name nautilusst2986 \
  --account-key "$ACCOUNT_KEY" \
  --query "{copyStatus:properties.copy.status, copyProgress:properties.copy.progress}" \
  --output table
```

**Expected output:**

```
CopyStatus    CopyProgress
------------  --------------
success       33/33
```

***Screenshot Placeholder: Step 11 - blob show table confirming CopyStatus success and CopyProgress 33/33 on destination blob***

> Do not proceed to Phase 5 until `CopyStatus` is `success`. Re-run this command if still showing `pending`.

---

## Phase 5 - Data Consistency Verification

### Step 12 - Retrieve MD5 Checksum from Source

```bash
az storage blob show \
  --container-name nautilus-source-4584 \
  --name nautilus.txt \
  --account-name nautilusst2986 \
  --account-key "$ACCOUNT_KEY" \
  --query "properties.contentSettings.contentMd5" \
  --output tsv
```

**Result:** `Lu7zilatbGguzSz2Ecn5IQ==`

***Screenshot Placeholder: Step 12 - tsv output of source blob MD5 checksum Lu7zilatbGguzSz2Ecn5IQ==***

---

### Step 13 - Retrieve MD5 Checksum from Destination

```bash
az storage blob show \
  --container-name nautilus-dest-30683 \
  --name nautilus.txt \
  --account-name nautilusst2986 \
  --account-key "$ACCOUNT_KEY" \
  --query "properties.contentSettings.contentMd5" \
  --output tsv
```

**Result:** `Lu7zilatbGguzSz2Ecn5IQ==`

***Screenshot Placeholder: Step 13 - tsv output of destination blob MD5 checksum matching source at Lu7zilatbGguzSz2Ecn5IQ==***

> **Both values must be identical.** A mismatch indicates data corruption during transfer. If checksums differ, abort and re-execute Phase 4.

---

### Step 14 - Compare File Sizes

```bash
# Source
az storage blob show \
  --container-name nautilus-source-4584 \
  --name nautilus.txt \
  --account-name nautilusst2986 \
  --account-key "$ACCOUNT_KEY" \
  --query "properties.contentLength" \
  --output tsv

# Destination
az storage blob show \
  --container-name nautilus-dest-30683 \
  --name nautilus.txt \
  --account-name nautilusst2986 \
  --account-key "$ACCOUNT_KEY" \
  --query "properties.contentLength" \
  --output tsv
```

**Expected output from both commands:** `33`

***Screenshot Placeholder: Step 14 - both contentLength queries returning 33 confirming identical file sizes in source and destination***

---

### Step 15 - Download and Binary Diff Both Files

```bash
# Download from source
az storage blob download \
  --container-name nautilus-source-4584 \
  --name nautilus.txt \
  --file ./nautilus_source.txt \
  --account-name nautilusst2986 \
  --account-key "$ACCOUNT_KEY"

# Download from destination
az storage blob download \
  --container-name nautilus-dest-30683 \
  --name nautilus.txt \
  --file ./nautilus_dest.txt \
  --account-name nautilusst2986 \
  --account-key "$ACCOUNT_KEY"

# Compare
diff nautilus_source.txt nautilus_dest.txt
```

**Expected output from `diff`:** No output. Silence confirms the files are byte-for-byte identical.

***Screenshot Placeholder: Step 15 - diff command returning empty output confirming nautilus_source.txt and nautilus_dest.txt are byte-for-byte identical***

---

## Phase 6 - Final Audit and Cleanup

### Step 16 - Audit Source Container Blob Listing

```bash
az storage blob list \
  --container-name nautilus-source-4584 \
  --account-name nautilusst2986 \
  --account-key "$ACCOUNT_KEY" \
  --output table
```

**Expected output:**

```
Name          Blob Type    Blob Tier    Length    Content Type    Last Modified
------------  -----------  -----------  --------  --------------  -------------------------
nautilus.txt  BlockBlob    Hot          33        text/plain      2026-03-12T00:09:26+00:00
```

***Screenshot Placeholder: Step 16 - blob list table for nautilus-source-4584 showing nautilus.txt as BlockBlob with length 33***

---

### Step 17 - Audit Destination Container Blob Listing

```bash
az storage blob list \
  --container-name nautilus-dest-30683 \
  --account-name nautilusst2986 \
  --account-key "$ACCOUNT_KEY" \
  --output table
```

**Expected output:**

```
Name          Blob Type    Blob Tier    Length    Content Type    Last Modified
------------  -----------  -----------  --------  --------------  -------------------------
nautilus.txt  BlockBlob    Hot          33        text/plain      2026-03-12T00:24:41+00:00
```

***Screenshot Placeholder: Step 17 - blob list table for nautilus-dest-30683 confirming nautilus.txt present with length 33 and copy timestamp***

---

### Step 18 - Remove Local Verification Files

```bash
rm nautilus_source.txt nautilus_dest.txt
```

> These files were downloaded solely for the purpose of the `diff` check. They must be removed after verification to avoid leaving sensitive blob content on the local filesystem.

***Screenshot: rm command executing cleanly with no output and clean shell prompt returned***
<img width="1039" height="467" alt="image" src="https://github.com/user-attachments/assets/7d7bc4f4-2f2b-49b5-91df-5fde76e98832" />

---

## Verification Results

The following table captures the complete verification chain executed in this runbook:

| Verification Gate | Method | Expected | Actual | Result |
|---|---|---|---|---|
| Source container exists | `az storage container show` | JSON with container name | `nautilus-source-4584` confirmed | **PASS** |
| Source blob exists | `az storage blob show` | `nautilus.txt`, 33 bytes | `nautilus.txt`, 33 bytes | **PASS** |
| Destination container created | `az storage container create` | `"created": true` | `"created": true` | **PASS** |
| Destination container is private | `az storage container show` | `publicAccess: null` | `publicAccess: null` | **PASS** |
| Blob copy completed | `az storage blob copy start` | `copy_status: success` | `copy_status: success` | **PASS** |
| Copy progress confirmed | `az storage blob show` | `33/33` | `33/33` | **PASS** |
| MD5 source | `contentSettings.contentMd5` | Any valid hash | `Lu7zilatbGguzSz2Ecn5IQ==` | **PASS** |
| MD5 destination | `contentSettings.contentMd5` | Must match source | `Lu7zilatbGguzSz2Ecn5IQ==` | **PASS** |
| File size source | `contentLength` | `33` | `33` | **PASS** |
| File size destination | `contentLength` | `33` | `33` | **PASS** |
| Binary diff | `diff` | No output | No output | **PASS** |
| Source audit | `az storage blob list` | `nautilus.txt` present | `nautilus.txt` present | **PASS** |
| Destination audit | `az storage blob list` | `nautilus.txt` present | `nautilus.txt` present | **PASS** |

**All 13 verification gates passed. Migration is complete and verified.**

---

## Known Issues and Resolutions

### Issue 1 - Authentication Failure When Pasting Account Key Directly

**Symptom:**

```
Authentication failure. This may be caused by either invalid account key,
connection string or sas token value provided for your storage account.
```

**Root Cause:**

The Azure storage account key contains `+` characters. When pasted directly into the shell as a raw argument (with or without quotes in certain shell configurations), the `+` is misinterpreted as whitespace, silently corrupting the key before it reaches the Azure CLI.

**Resolution:**

Load the key into a shell variable using command substitution. This bypasses all manual copy-paste handling entirely:

```bash
ACCOUNT_KEY=$(az storage account keys list \
  --account-name nautilusst2986 \
  --query "[0].value" \
  --output tsv)
```

Then reference it in all subsequent commands as `"$ACCOUNT_KEY"` with double quotes.

***Screenshot Placeholder: Known Issue 1 - terminal showing authentication failure on direct key paste followed by successful authentication using $ACCOUNT_KEY variable***

---

### Issue 2 - CLI Reports 2 Available Updates

**Symptom:**

```
You have 2 update(s) available. Consider updating your CLI installation with 'az upgrade'
```

**Impact:** None. This is informational only. The installed version `2.67.0` supports all commands used in this runbook.

**Resolution:** No action required for this migration. To suppress in future sessions, run `az upgrade` after the migration is complete in a non-production window.

---

## Security Considerations

* **Never hardcode the storage account key** in scripts, commits, or documentation. Always use shell variable injection or a secrets manager such as Azure Key Vault.
* **The `$ACCOUNT_KEY` variable is session-scoped.** It is not persisted to disk and will not survive a terminal restart.
* **Local downloads from Step 15** contain live blob content and must be deleted immediately after the diff check. Step 18 handles this.
* **Container access is private by default.** The `--public-access off` flag was explicitly set during provisioning and verified via `az storage container show` before migration began.
* **Rotate the storage account key** after this runbook if it was exposed in terminal history or logs.

---

## Repository Structure

```
azure/
└── storage/
    └── blob-migration-pipelines/
        └── README.md          # This document
```

---

## Authors

**Nautilus DevOps Team**
Azure Infrastructure Engineering
`nautilusst2986` | East US | Azure Free Labs

---

*Executed: 2026-03-12 | Azure CLI 2.67.0 | Storage API Version 2022-11-02*


<img width="1031" height="750" alt="image" src="https://github.com/user-attachments/assets/f7836c1c-3af5-4d86-ae87-03fd3ffb1707" />
<img width="1037" height="504" alt="image" src="https://github.com/user-attachments/assets/356f979b-fcc1-4143-9f69-01d3542d50d0" />
<img width="1030" height="612" alt="image" src="https://github.com/user-attachments/assets/ef80a67f-ab2a-4a05-8846-13f504863309" />
<img width="1031" height="331" alt="image" src="https://github.com/user-attachments/assets/0f58347c-eff1-4ea6-a1ad-a389edea491e" />
<img width="1033" height="386" alt="image" src="https://github.com/user-attachments/assets/ad1e52c0-ea15-439a-a555-1fe8dd5fbac0" />
<img width="1035" height="736" alt="image" src="https://github.com/user-attachments/assets/661ecc82-9df3-44b9-a8f2-5d7e061d9de9" />
<img width="1035" height="736" alt="image" src="https://github.com/user-attachments/assets/6dbd34a9-03a3-4ecb-88e1-2f4d24754779" />
<img width="1033" height="847" alt="image" src="https://github.com/user-attachments/assets/d10c3027-850d-41bb-905a-be1c067c597f" />
<img width="1035" height="419" alt="image" src="https://github.com/user-attachments/assets/3d7a96c8-bb37-4e6c-8a4d-fed96519a84d" />
<img width="1029" height="615" alt="image" src="https://github.com/user-attachments/assets/7be535e3-923f-4485-99f6-1690b2191444" />
<img width="1029" height="802" alt="image" src="https://github.com/user-attachments/assets/8fa42ab8-8465-472b-947c-5908734141dc" />
<img width="1030" height="849" alt="image" src="https://github.com/user-attachments/assets/4f452bb9-28ad-4de8-9598-a3215308a719" />
<img width="1033" height="827" alt="image" src="https://github.com/user-attachments/assets/7ff48c94-23b0-4923-aa52-1f90da13a29f" />
<img width="1028" height="598" alt="image" src="https://github.com/user-attachments/assets/fa46a938-b276-4e49-a96b-e022034ba730" />
<img width="1026" height="733" alt="image" src="https://github.com/user-attachments/assets/bc8374aa-b87b-4570-9acd-a6c51e71e996" />
<img width="1028" height="856" alt="image" src="https://github.com/user-attachments/assets/5a56e483-1105-487e-99f3-aacbf0d50e1d" />
<img width="1028" height="859" alt="image" src="https://github.com/user-attachments/assets/880094cc-9f97-42b5-ad3f-d7556125dab3" />
<img width="1031" height="867" alt="image" src="https://github.com/user-attachments/assets/8c407f2b-7355-465b-91b5-eb3d9b52aef2" />
<img width="1035" height="866" alt="image" src="https://github.com/user-attachments/assets/ecf4d739-1464-4d8a-ae24-fa7320006906" />
<img width="1030" height="533" alt="image" src="https://github.com/user-attachments/assets/81afda70-4bc2-4b41-a38d-19d991a12ff0" />
<img width="1035" height="427" alt="image" src="https://github.com/user-attachments/assets/afa0b908-6073-4aad-a84b-21c917fa680b" />

